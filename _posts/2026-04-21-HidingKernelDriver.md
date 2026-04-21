---
title: Hiding Kernel Driver (KDMapper)
date: 2026-04-21 12:26:00 +0900
categories: [Windows Internals]
tags: [Windows Internals]
---

## PsLoadedModuleList
일반적으로 윈도우에서 드라이버가 로드되면 PsLoadedModuleList에 기록된다.<br>
이는 현재 로드된 드라이버 목록이다. windbg에서 lmk 명령어를 사용해 확인한 리스트와 동일하다.

대부분 해킹툴들은 **iqvw64e.sys**와 같은 취약 드라이버를 로드하고, 사용이 완료되면 언로드하기 때문에 PsLoadedModuleList에는 흔적이 남지 않는다.
```
2: kd> lmk
start             end                 module name
ffffcc98`f1200000 ffffcc98`f14d6000   win32kbase   (deferred)             
ffffcc98`f14e0000 ffffcc98`f1894000   win32kfull   (deferred)   
...
```

## MmUnloadedDrivers
언로드 된 드라이버 목록이다. 마찬가지로 windbg에서 lmk 명령어를 통해 확인할 수 있다.
```
Unloaded modules:
fffff801`aa390000 fffff801`aa397000   RTCore64.sys
fffff801`aa580000 fffff801`aa58d000   PROCEXP152.sys
...
```

다음과 같은 스크립트[MmUnloadedDrivers](https://github.com/wlsgjd/Windbg-Scripts/blob/main/MmUnloadedDrivers)를 통해서도 확인할 수 있다.
```
r $t0 = poi(nt!MmUnloadedDrivers)
r $t1 = poi(nt!MmUnloadedDrivers + 0x8)

.printf "MmUnloadedDrivers: %p\n", @$t0
.printf "Count: %d\n", @$t1

.for (r $t2 = 0; @$t2 < @$t1; r $t2 = @$t2 + 1)
{
    .printf "----------------------------------------\n"
    .printf "Entry %d\n", @$t2

    r $t3 = @$t0 + (@$t2 * 0x28)

    .printf "Struct @ %p\n", @$t3

    .printf "Name: "
    du poi(@$t3 + 0x8)

    .printf "Start: %p\n", poi(@$t3 + 0x10)
    .printf "End  : %p\n", poi(@$t3 + 0x18)

    .printf "UnloadTime: %I64u\n", poi(@$t3 + 0x20)
}
```

아래는 출력 예시이다.
```
2: kd> $$>< C:\scripts\windbg\MmUnloadedDrivers
MmUnloadedDrivers: ffffdf8b25d11340
Count: 34
----------------------------------------
Entry 0
Struct @ ffffdf8b25d11340
Name: ffffdf8b`286a23f0  "hwpolicy.sys"
Start: fffff80155170000
End  : fffff80155181000
UnloadTime: 134208079057490642
----------------------------------------
Entry 1
Struct @ ffffdf8b25d11368
Name: ffffdf8b`286a3080  "WdBoot.sys"
Start: fffff80154000000
End  : fffff8015400a000
UnloadTime: 134208079059677628
----------------------------------------
```

### _MM_UNLOADED_DRIVER
MmUnloadedDrivers은 **_MM_UNLOADED_DRIVER**에 대한 배열 메모리이다.
```cpp
typedef struct _MM_UNLOADED_DRIVER
{
    UNICODE_STRING 	Name;
    PVOID 			ModuleStart;
    PVOID 			ModuleEnd;
    ULONG64 		UnloadTime;
} MM_UNLOADED_DRIVER, * PMM_UNLOADED_DRIVER;
```

windbg를 통해 확인해보면 다음과 같이 구성되어 있다.
```
2: kd> dqs MmUnloadedDrivers
fffff801`5122a440  ffffdf8b`25d11340 // MmUnloadedDrivers
fffff801`5122a448  00000000`00000022 // MmUnloadedDriversCount
```

배열 메모리이기 때문에 내용을 쉽게 확인할 수 있다.
```
2: kd> dqs ffffdf8b`25d11340
ffffdf8b`25d11340  00000000`00180018
ffffdf8b`25d11348  ffffdf8b`286a23f0
ffffdf8b`25d11350  fffff801`55170000 <Unloaded_hwpolicy.sys>
ffffdf8b`25d11358  fffff801`55181000
ffffdf8b`25d11360  01dccd89`6e3592d2
ffffdf8b`25d11368  00000000`00140014
ffffdf8b`25d11370  ffffdf8b`286a3080
ffffdf8b`25d11378  fffff801`54000000 <Unloaded_WdBoot.sys>
ffffdf8b`25d11380  fffff801`5400a000
ffffdf8b`25d11388  01dccd89`6e56f1bc
ffffdf8b`25d11390  00000000`000e000e
ffffdf8b`25d11398  ffffdf8b`282ff970
ffffdf8b`25d113a0  fffff801`68ff0000 dump_dumpfve!FveLibInitEx <PERF> (dump_dumpfve+0x0)
ffffdf8b`25d113a8  fffff801`6900d000
ffffdf8b`25d113b0  01dccd89`6e8dc4a4
ffffdf8b`25d113b8  00000000`00200020
```

### MiRememberUnloadedDriver
드라이버를 언로드하게 되면 OS 내부에서 MiRememberUnloadedDriver이 호출된다.<br>
해당 함수는 호출 시 <code>LDR_DATA_TABLE_ENTRY</code>를 전달 받으며 MmUnloadedDrivers에 내용을 기록한다.<br>
```
2: kd> dt_LDR_DATA_TABLE_ENTRY
nt!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY
   +0x010 InMemoryOrderLinks : _LIST_ENTRY
   +0x020 InInitializationOrderLinks : _LIST_ENTRY
   +0x030 DllBase          : Ptr64 Void
   +0x038 EntryPoint       : Ptr64 Void
   +0x040 SizeOfImage      : Uint4B
   +0x048 FullDllName      : _UNICODE_STRING
   +0x058 BaseDllName      : _UNICODE_STRING
```

함수 내부를 자세히 보면 전달 받은 **BaseDllName**의 길이를 검사한다. 길이가 0인 경우 기록하지 않는다.
```
2: kd> uf MiRememberUnloadedDriver
nt!MiRememberUnloadedDriver:
fffff801`50d5f204 48895c2408      mov     qword ptr [rsp+8],rbx
fffff801`50d5f209 48896c2410      mov     qword ptr [rsp+10h],rbp
fffff801`50d5f20e 4889742418      mov     qword ptr [rsp+18h],rsi
fffff801`50d5f213 57              push    rdi
fffff801`50d5f214 4156            push    r14
fffff801`50d5f216 4157            push    r15
fffff801`50d5f218 4883ec20        sub     rsp,20h
fffff801`50d5f21c 4533ff          xor     r15d,r15d
fffff801`50d5f21f 458bf0          mov     r14d,r8d
fffff801`50d5f222 488bea          mov     rbp,rdx
fffff801`50d5f225 488bf1          mov     rsi,rcx
fffff801`50d5f228 66443939        cmp     word ptr [rcx],r15w
fffff801`50d5f22c 0f84b3000000    je      nt!MiRememberUnloadedDriver+0xe1 (fffff801`50d5f2e5)  Branch

nt!MiRememberUnloadedDriver+0xe1:
fffff801`50d5f2e5 488b5c2440      mov     rbx,qword ptr [rsp+40h]
fffff801`50d5f2ea 488b6c2448      mov     rbp,qword ptr [rsp+48h]
fffff801`50d5f2ef 488b742450      mov     rsi,qword ptr [rsp+50h]
fffff801`50d5f2f4 4883c420        add     rsp,20h
fffff801`50d5f2f8 415f            pop     r15
fffff801`50d5f2fa 415e            pop     r14
fffff801`50d5f2fc 5f              pop     rdi
fffff801`50d5f2fd c3              ret
```

[kdmapper](https://github.com/TheCruZ/kdmapper)는 이를 이용하여 MmUnloadedDrivers에 기록되지 않도록 길이와 버퍼를 제거한다.
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L834-L855
	UNICODE_STRING us_driver_base_dll_name = { 0 };

	if (!ReadMemory(driver_section + 0x58, &us_driver_base_dll_name, sizeof(us_driver_base_dll_name)) || us_driver_base_dll_name.Length == 0) {
		kdmLog(L"[!] Failed to find driver name" << std::endl);
		return false;
	}

	auto unloadedName = std::make_unique<wchar_t[]>((ULONG64)us_driver_base_dll_name.Length / 2ULL + 1ULL);
	if (!ReadMemory((uintptr_t)us_driver_base_dll_name.Buffer, unloadedName.get(), us_driver_base_dll_name.Length)) {
		kdmLog(L"[!] Failed to read driver name" << std::endl);
		return false;
	}

	us_driver_base_dll_name.Length = 0; //MiRememberUnloadedDriver will check if the length > 0 to save the unloaded driver

	if (!WriteMemory(driver_section + 0x58, &us_driver_base_dll_name, sizeof(us_driver_base_dll_name))) {
		kdmLog(L"[!] Failed to write driver name length" << std::endl);
		return false;
	}

	kdmLog(L"[+] MmUnloadedDrivers Cleaned: " << unloadedName << std::endl);
	return true;
```

## PiDDBCache
PiDDBCache은 드라이버가 로드되면서 남는 캐시 데이터이다.<br>다음과 같은 스크립트 [piddbcachelist](https://github.com/wlsgjd/Windbg-Scripts/blob/main/piddbcachelist)를 통해서도 간단하게 확인이 가능하다.
```
r $t0 = poi(nt!PiDDBCacheList)

.for ( ; @$t0 != nt!PiDDBCacheList; r $t0 = poi(@$t0) ) {

    .printf "====================\n"
    .printf "Entry: %p\n", @$t0

    .printf "Driver: "
    du poi(@$t0 + 0x18)

    .printf "Timestamp: "
    dd (@$t0 + 0x20) L1

    .printf "\n"
}
```
아래는 출력 예시이다.
```
2: kd> $$>< C:\scripts\windbg\piddbcachelist
====================
Entry: ffff830de6eacda0
Driver: ffff830d`e6d5b940  "Wdf01000.sys"
Timestamp: ffff830d`e6eacdc0  488b79bb

====================
Entry: ffff830de6eac160
Driver: ffff830d`e6d5b7c0  "acpiex.sys"
Timestamp: ffff830d`e6eac180  c8d60b44
```


PiDDBCache은 <code>_PI_DDB_CACHE_ENTRY</code>를 기록하며 구조는 다음과 같다.
```cpp
typedef struct _PI_DDB_CACHE_ENTRY
{
    LIST_ENTRY      List;          // 연결 리스트
    UNICODE_STRING  DriverName;    // 드라이버 이름
    ULONG           TimeDateStamp; // PE 헤더 timestamp
    NTSTATUS        LoadStatus;    // 로드 결과
    char            _padding[16];  // (버전에 따라 다름)
} PI_DDB_CACHE_ENTRY, *PPI_DDB_CACHE_ENTRY;
```

PiDDBCacheTable, PiDDBCacheList에 기록되며 각각 AVL 트리, 리스트 구조를 가진다.<br>아래는 windbg에서 확인한 리스트 내용 중 일부이다.
```
2: kd> dqs nt!PiDDBCacheList
fffff801`5132ebe0  ffff830d`e6eacda0
fffff801`5132ebe8  ffff830d`eee0c880

2: kd> dqs ffff830d`e6eacda0
ffff830d`e6eacda0  ffff830d`e6eac160
ffff830d`e6eacda8  fffff801`5132ebe0 nt!PiDDBCacheList
ffff830d`e6eacdb0  00000000`00180018
ffff830d`e6eacdb8  ffff830d`e6d5b940

2: kd> du ffff830d`e6d5b940
ffff830d`e6d5b940  "Wdf01000.sys"
```

kdmapper는 각 요소에 대한 데이터를 모두 제거한다.
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L1027
	// first, unlink from the list
	PLIST_ENTRY prev;
	if (!ReadMemory((uintptr_t)pFoundEntry + (offsetof(struct nt::_PiDDBCacheEntry, List.Blink)), &prev, sizeof(_LIST_ENTRY*))) {
		kdmLog(L"[-] Can't get prev entry" << std::endl);
		ExReleaseResourceLite(PiDDBLock);
		return false;
	}
	PLIST_ENTRY next;
	if (!ReadMemory((uintptr_t)pFoundEntry + (offsetof(struct nt::_PiDDBCacheEntry, List.Flink)), &next, sizeof(_LIST_ENTRY*))) {
		kdmLog(L"[-] Can't get next entry" << std::endl);
		ExReleaseResourceLite(PiDDBLock);
		return false;
	}

	kdmLog("[+] Found Table Entry = 0x" << std::hex << pFoundEntry << std::endl);

	if (!WriteMemory((uintptr_t)prev + (offsetof(struct _LIST_ENTRY, Flink)), &next, sizeof(_LIST_ENTRY*))) {
		kdmLog(L"[-] Can't set next entry" << std::endl);
		ExReleaseResourceLite(PiDDBLock);
		return false;
	}
	if (!WriteMemory((uintptr_t)next + (offsetof(struct _LIST_ENTRY, Blink)), &prev, sizeof(_LIST_ENTRY*))) {
		kdmLog(L"[-] Can't set prev entry" << std::endl);
		ExReleaseResourceLite(PiDDBLock);
		return false;
	}
```
```cpp
	// then delete the element from the avl table
	if (!RtlDeleteElementGenericTableAvl(PiDDBCacheTable, pFoundEntry)) {
		kdmLog(L"[-] Can't delete from PiDDBCacheTable" << std::endl);
		ExReleaseResourceLite(PiDDBLock);
		return false;
	}
```

## KernelHashBucketList
KernelHashBucketList은 드라이버가 로드될 때 기록되는 해시 리스트이다.<br>
ci.dll에 존재하는 전역 변수이며 일반적인 방식으로는 가져오는게 불가능하다.

KDMapper에서도 다음과 같이 시그니처를 통해 찾는 것으로 보인다.
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L1151
	PVOID g_KernelHashBucketList = (PVOID)(ci + g_KernelHashBucketListOffset);
	PVOID g_HashCacheLock = (PVOID)(ci + g_HashCacheLockOffset);
#else
	auto sig = FindPatternInSectionAtKernel("PAGE", ci, PUCHAR("\x48\x8B\x1D\x00\x00\x00\x00\xEB\x00\xF7\x43\x40\x00\x20\x00\x00"), "xxx????x?xxxxxxx");
	if (!sig) {
		kdmLog(L"[-] Can't Find g_KernelHashBucketList" << std::endl);
		return false;
	}
	auto sig2 = FindPatternAtKernel((uintptr_t)sig - 50, 50, PUCHAR("\x48\x8D\x0D"), "xxx");
	if (!sig2) {
		kdmLog(L"[-] Can't Find g_HashCacheLock" << std::endl);
		return false;
	}
	const auto g_KernelHashBucketList = ResolveRelativeAddress((PVOID)sig, 3, 7);
	const auto g_HashCacheLock = ResolveRelativeAddress((PVOID)sig2, 3, 7);
	if (!g_KernelHashBucketList || !g_HashCacheLock)
	{
		kdmLog(L"[-] Can't Find g_HashCache relative address" << std::endl);
		return false;
	}
#endif
```

해당 시그니처를 검색하면 다음과 같다. <code>CI!g_KernelHashBucketList</code>를 찾는 것으로 보인다.
```
2: kd> u fffff801`53c7e8ff
CI!I_SetSecurityState+0x57:
fffff801`53c7e8ff 488b1d8a370400  mov     rbx,qword ptr [CI!g_KernelHashBucketList (fffff801`53cc2090)]
fffff801`53c7e906 eb4c            jmp     CI!I_SetSecurityState+0xac (fffff801`53c7e954)
```

g_KernelHashBucketList에는 <code>_HashBucketEntry</code>이 리스트 구조로 기록된다.
```cpp
	typedef struct _HashBucketEntry
	{
		struct _HashBucketEntry* Next;
		UNICODE_STRING DriverName;
		ULONG CertHash[5];
	} HashBucketEntry, * PHashBucketEntry;
```

windbg를 통해 확인해보면 다음과 같다. 드라이버 경로와 인증서 관련 해시가 기록되는 것으로 보인다.
```
2: kd> dqs CI!g_KernelHashBucketList
fffff801`53cc2090  ffff830d`f519ed70
fffff801`53cc2098  00000000`00000000

2: kd> dqs ffff830d`f519ed70
ffff830d`f519ed70  ffff830d`f5205ad0
ffff830d`f519ed78  ffff830d`004c004a
ffff830d`f519ed80  ffff830d`f519edb8

2: kd> du ffff830d`f519edb8
ffff830d`f519edb8  "\Windows\System32\drivers\myfault.sys"
```

kdmapper에서는 리스트 내에서 대상과 일치하는 드라이버를 찾고 링크를 끊어버린다.
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L1232-L1250

			size_t find_result = std::wstring(wsName.get()).find(wdname);
			if (find_result != std::wstring::npos) {
				kdmLog(L"[+] Found In g_KernelHashBucketList: " << std::wstring(&wsName[find_result]) << std::endl);
				nt::HashBucketEntry* Next = 0;
				if (!ReadMemory((uintptr_t)entry, &Next, sizeof(Next))) {
					kdmLog(L"[-] Failed to read g_KernelHashBucketList next entry ptr!" << std::endl);
					if (!ExReleaseResourceLite(g_HashCacheLock)) {
						kdmLog(L"[-] Failed to release g_KernelHashBucketList lock!" << std::endl);
					}
					return false;
				}

				if (!WriteMemory((uintptr_t)prev, &Next, sizeof(Next))) {
					kdmLog(L"[-] Failed to write g_KernelHashBucketList prev entry ptr!" << std::endl);
					if (!ExReleaseResourceLite(g_HashCacheLock)) {
						kdmLog(L"[-] Failed to release g_KernelHashBucketList lock!" << std::endl);
					}
					return false;
				}
```
## WdFilterDriverList
WdFilterDriverList은 Windows Defender에서 사용하는 드라이버 목록입니다. <br>일부 안티치트에서 해당 리스트를 통해 탐지하고 있는 것으로 알려져 있습니다.
```
// https://www.unknowncheats.me/forum/anti-cheat-bypass/509917-eac-taking-advantage-wdfilter-driver.html
EAC taking advantage of WdFilter driver
It came to my attention while running the EAC driver in my kernel emulator in kernel they do a pattern scan for "48 8B 0D ? ? ? ? FF 05" in WdFilter.sys(windows defender's file system filter driver) getting a qword in data section, then they subtract 0x8 from that to get a pointer to struct below.

From here they parse DriverInfoList to find all vulnerable drivers loaded / unloaded.
```

당연하게도 공식적으로 지원하지 않는 방식이라 시그니처를 통해 주소를 스캔하는 것으로 보입니다.
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L258-L286

	// MpCleanupDriverInfo->MpFreeDriverInfoEx
	// The pattern only focus in the 0x8 offset and the possibility of the different order for the instructions
	/*
		49 8B C9                      mov     rcx, r9         ; P
		49 89 50 08                   mov     [r8+8], rdx
		E8 FB F0 FD FF                call    MpFreeDriverInfoEx
		48 8B 0D FC AA FA FF          mov     rcx, cs:qword_1C0021BF0
		E9 21 FF FF FF                jmp     loc_1C007701A
	*/
	auto MpFreeDriverInfoExRef = FindPatternInSectionAtKernel("PAGE", WdFilter, (PUCHAR)"\x89\x00\x08\xE8\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xE9", "x?xx???????????x");
	if (!MpFreeDriverInfoExRef) {
		/*
			48 89 4A 08                   mov     [rdx+8], rcx
			49 8B C8                      mov     rcx, r8         ; P
			E8 C3 58 FE FF                call    sub_1C0065308
			48 8B 0D 44 41 FA FF          mov     rcx, cs:qword_1C0023B90
			E9 39 FF FF FF                jmp     loc_1C007F98A
		*/
		MpFreeDriverInfoExRef = FindPatternInSectionAtKernel("PAGE", WdFilter, (PUCHAR)"\x89\x00\x08\x00\x00\x00\xE8\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xE9", "x?x???x???????????x");
		if (!MpFreeDriverInfoExRef) {
			kdmLog("[!] Failed to find WdFilter MpFreeDriverInfoEx" << std::endl);
			return false;
		}
		else {
			kdmLog("[+] Found WdFilter MpFreeDriverInfoEx with second pattern" << std::endl);
		}
		MpFreeDriverInfoExRef += 0x3; // adjust for next sum offset
	}
```

kdmapper 코드를 살펴보면 리스트 내 연결을 끊고, RuntimeDriversCount를 조작하고, 배열 메모리를 조작하고, 관련 메모리도 해제하고... 다양하게 조지고 있습니다. ㅎ호
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L327-L336
					if (!removedRuntimeDriversArray) {
						kdmLog("[!] Failed to remove from RuntimeDriversArray" << std::endl);
						return false;
					}

					auto NextEntry = ReadListEntry(uintptr_t(Entry) + (offsetof(struct _LIST_ENTRY, Flink)));
					auto PrevEntry = ReadListEntry(uintptr_t(Entry) + (offsetof(struct _LIST_ENTRY, Blink)));

					WriteMemory(uintptr_t(NextEntry) + (offsetof(struct _LIST_ENTRY, Blink)), &PrevEntry, sizeof(LIST_ENTRY::Blink));
					WriteMemory(uintptr_t(PrevEntry) + (offsetof(struct _LIST_ENTRY, Flink)), &NextEntry, sizeof(LIST_ENTRY::Flink));
```
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L313-L325
					//remove from RuntimeDriversArray
					bool removedRuntimeDriversArray = false;
					PVOID SameIndexList = (PVOID)((uintptr_t)Entry - 0x10);
					for (int k = 0; k < 256; k++) { // max RuntimeDriversArray elements
						PVOID value = 0;
						ReadMemory(RuntimeDriversArray + (k * 8), &value, sizeof(PVOID));
						if (value == SameIndexList) {
							PVOID emptyval = (PVOID)(RuntimeDriversCount + 1); // this is not count+1 is position of cout addr+1
							WriteMemory(RuntimeDriversArray + (k * 8), &emptyval, sizeof(PVOID));
							removedRuntimeDriversArray = true;
							break;
						}
					}
```
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L338-L342
					// decrement RuntimeDriversCount
					ULONG current = 0;
					ReadMemory(RuntimeDriversCount, &current, sizeof(ULONG));
					current--;
					WriteMemory(RuntimeDriversCount, &current, sizeof(ULONG));
```
```cpp
// https://github.com/TheCruZ/kdmapper/blob/280f65733d6469cfdae9dff76f24793d90f1f672/kdmapper/intel_driver.cpp#L344-L355
					// call MpFreeDriverInfoEx
					uintptr_t DriverInfo = (uintptr_t)Entry - 0x20;

					//verify DriverInfo Magic
					USHORT Magic = 0;
					ReadMemory(DriverInfo, &Magic, sizeof(USHORT));
					if (Magic != 0xDA18) {
						kdmLog("[!] DriverInfo Magic is invalid, new wdfilter version?, driver info will not be released to prevent bsod" << std::endl);
					}
					else {
						CallKernelFunction<void>(nullptr, MpFreeDriverInfoEx, DriverInfo);
					}

					kdmLog("[+] WdFilterDriverList Cleaned: " << ImageName << std::endl);
```



## References
- https://github.com/TheCruZ/kdmapper
- https://www.unknowncheats.me/forum/anti-cheat-bypass/509917-eac-taking-advantage-wdfilter-driver.html
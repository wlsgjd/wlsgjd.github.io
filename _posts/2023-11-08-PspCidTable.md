---
title: 오브젝트 테이블 변조 (PspCidTable)
categories: [Analysis Report]
tags: [Windows Kernel]
---

## STATUS_INVALID_CID (0xC000000B)
OpenProcess를 호출하면 0xC000000B를 반환하며 실패하는 해킹툴이 발견되었습니다.
![](/assets/posts/2023-11-08-PspCidTable/1.png)

다음과 같이 NtOpenProcess 시스콜 이후에 실패하고 있으며, 이를 봤을 때 커널 레벨에서 조작된 것을 추측할 수 있습니다.
![](/assets/posts/2023-11-08-PspCidTable/2.png)

해당 값은 [MS-ERREF](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55) 문서에 나와있는데 클라이언트 ID가 잘못되었음을 나타냅니다. 해당 내용만 봐서는 딱히 짐작이 되질 않습니다.

| Return value/code   |	Description |
|---------------------|----------------|
| 0xC000000B<br><br>STATUS_INVALID_CID |  An invalid client ID was specified. |

## PsLookupProcessByProcessId
유저 레벨에서 OpenProcess를 호출할 경우, 커널 내부에서 PsLookupProcessByProcessId를 호출하게 됩니다. 해당 함수에서 실패 시 STATUS_INVALID_CID를 리턴하고 있습니다.
![](/assets/posts/2023-11-08-PspCidTable/3.png)

호출 스택은 다음과 같이 정리할 수 있습니다.

| Ring | Call Stacks                    |
|:-:|-----------------------------------|
| 3 | OpenProcess                       |
| 3 | NtOpenProcess                     |
| 0 | PsOpenProcess                     |
| 0 | *PsLookupProcessByProcessId       |
| 0 | PspReferenceCidTableEntry         |
| 0 | ExpLookupHandleTableEntry         |

내부에서 사용되는 함수들은 다음과 같은 호출 구조를 가진 것으로 추측합니다. 문서화되지 않은 부분이라 정확하진 않습니다.
```cpp
BOOL PspReferenceCidTableEntry(ULONG pid, PEPROCESS* process);
void* ExpLookupHandleTableEntry(void* PspCidTable, ULONG pid);
```

## ExpLookupHandleTableEntry
PspCidTable이라는 프로세스 오브젝트 테이블을 전달 받으며, 특정한 연산을 통해 PID를 기반으로 테이블 인덱스를 생성하고 데이터를 반환합니다. 
![](/assets/posts/2023-11-08-PspCidTable/4.png)

일반적인 경우에 내부 연산은 다음과 같습니다.
```cpp
__int64 __fastcall ExpLookupHandleTableEntry(unsigned int *a1, __int64 a2)
{
    unsigned __int64 v2;    // rdx
    __int64 v3;             // r8

    v2 = a2 & 0xFFFFFFFFFFFFFFFCui64;
    if ( v2 >= *a1 )
        return 0i64;

    v3 = *((_QWORD *)a1 + 1);
    if ( (v3 & 3) == 1 )
        return *(_QWORD *)(v3 + 8 * (v2 >> 10) - 1) + 4 * (v2 & 0x3FF);
}
```

## PspReferenceCidTableEntry
ExpLookupHandleTableEntry를 통해 얻은 값을 쉬프트 연산자(SHR, 0x10)를 통해 EPROCESS 주소로 변환합니다.
![](/assets/posts/2023-11-08-PspCidTable/5.png)

## Kernel Memory Dump
위에 확인된 내용들을 바탕으로, 커널 메모리 덤프를 수집 후 해킹툴 분석을 추가로 진행하였습니다.

먼저, 덤프에서 조작된 프로세스의 eprocess 주소와 pid를 구합니다.
```
0: kd> !process 0 0 dwm.exe
PROCESS ffff9a8164694300
    SessionId: 1  Cid: 0450    Peb: 6232d5f000  ParentCid: 0374
    DirBase: 114606002  ObjectTable: ffff8a0997ac2b40  HandleCount: 1260.
    Image: dwm.exe

0: kd> dt_eprocess ffff9a8164694300
ntdll!_EPROCESS
   +0x440 UniqueProcessId  : 0x00000000`00000450 Void
```

그리고 ExpLookupHandleTableEntry 함수와 동일하게 연산하여 PspCidTable 내에 프로세스 오브젝트 주소를 구합니다.
```
0: kd> uf nt!ExpLookupHandleTableEntry
nt!ExpLookupHandleTableEntry:
fffff805`4c3f62c0 8b01            mov     eax,dword ptr [rcx]
fffff805`4c3f62c2 4883e2fc        and     rdx,0FFFFFFFFFFFFFFFCh
fffff805`4c3f62c6 483bd0          cmp     rdx,rax
fffff805`4c3f62c9 735a            jae     nt!ExpLookupHandleTableEntry+0x65 (fffff805`4c3f6325)  Branch

nt!ExpLookupHandleTableEntry+0xb:
fffff805`4c3f62cb 4c8b4108        mov     r8,qword ptr [rcx+8]
fffff805`4c3f62cf 418bc0          mov     eax,r8d
fffff805`4c3f62d2 83e003          and     eax,3
fffff805`4c3f62d5 83f801          cmp     eax,1
fffff805`4c3f62d8 7518            jne     nt!ExpLookupHandleTableEntry+0x32 (fffff805`4c3f62f2)  Branch

nt!ExpLookupHandleTableEntry+0x1a:
fffff805`4c3f62da 488bc2          mov     rax,rdx
fffff805`4c3f62dd 48c1e80a        shr     rax,0Ah
fffff805`4c3f62e1 81e2ff030000    and     edx,3FFh
fffff805`4c3f62e7 498b44c0ff      mov     rax,qword ptr [r8+rax*8-1]
fffff805`4c3f62ec 488d0490        lea     rax,[rax+rdx*4]
fffff805`4c3f62f0 c3              ret

nt!ExpLookupHandleTableEntry+0x32:
fffff805`4c3f62f2 85c0            test    eax,eax
fffff805`4c3f62f4 7506            jne     nt!ExpLookupHandleTableEntry+0x3c (fffff805`4c3f62fc)  Branch

nt!ExpLookupHandleTableEntry+0x36:
fffff805`4c3f62f6 498d0490        lea     rax,[r8+rdx*4]
fffff805`4c3f62fa c3              ret

nt!ExpLookupHandleTableEntry+0x3c:
fffff805`4c3f62fc 488bca          mov     rcx,rdx
fffff805`4c3f62ff 48c1e90a        shr     rcx,0Ah
fffff805`4c3f6303 488bc1          mov     rax,rcx
fffff805`4c3f6306 81e1ff010000    and     ecx,1FFh
fffff805`4c3f630c 48c1e809        shr     rax,9
fffff805`4c3f6310 81e2ff030000    and     edx,3FFh
fffff805`4c3f6316 498b44c0fe      mov     rax,qword ptr [r8+rax*8-2]
fffff805`4c3f631b 488b04c8        mov     rax,qword ptr [rax+rcx*8]
fffff805`4c3f631f 488d0490        lea     rax,[rax+rdx*4]
fffff805`4c3f6323 c3              ret

nt!ExpLookupHandleTableEntry+0x65:
fffff805`4c3f6325 33c0            xor     eax,eax
fffff805`4c3f6327 c3              ret
```

해당 코드를 순차적으로 계산하면 다음과 같습니다.
```
0: kd> dqs nt!PspCidTable
fffff805`4cafc5d0  ffff8a09`93444c00

rcx : ffff8a09`93444c00 // nt!PspCidTable
rdx : 0x450 // Process Id

0: kd> dqs ffff8a09`93444c00
ffff8a09`93444c00  00000000`00003000

eax : 0x00003000
rdx : 0x450 (0x450 & 0xFFFFFFFFFFFFFFFC)

0: kd> dqs ffff8a09`93444c00+8
ffff8a09`93444c08  ffff8a09`97a56001

r8 : ffff8a09`97a56001
eax : 97a56001 (r8d)

eax : 1 (eax & 3)

rax : 0x450 (rdx)
rax : 0x01 (rax >> 0x0A)
edx : 0x50 (edx & 0x3FF)

r8+rax*8-1 = ffff8a09`97a56008

0: kd> dqs ffff8a09`97a56008
ffff8a09`97a56008  ffff8a09`97a57000

rax : ffff8a09`97a57000

rax+rdx*4 = ffff8a09`97a57140
```
연산을 통해 획득한 PspCidTable 테이블 내 데이터를 확인하면 eprocess가 존재해야하는데 0으로 초기화되어 있습니다. 해킹툴에서 안티치트를 우회하기 위해 조작한 것으로 보입니다.
```
0: kd> dqs ffff8a09`97a57140
ffff8a09`97a57140  00000000`00000000 // dwm.exe => NULL
ffff8a09`97a57148  00000000`00000000
ffff8a09`97a57150  9a815e11`8080b569
ffff8a09`97a57158  00000000`00000000
```
다음과 같이 SHR(0x10)과 반대로 eprocess 주소를 SHL(0x10)한 뒤에 0x01을 더하면 복원이 가능합니다. 복원 시 PsLookupProcessByProcessId 및 OpenProcess 또한 정상적으로 동작합니다.
```
0: kd> eq ffff8a09`97a57140 (ffff9a8164694300 << 0x10) + 0x01

0: kd> dqs ffff8a09`97a57140
ffff8a09`97a57140  9a816469`43000001
ffff8a09`97a57148  00000000`00000000
ffff8a09`97a57150  9a815e11`8080b569
ffff8a09`97a57158  00000000`00000000
```

## Windbg Script
덤프 분석 시 활용할 수 있는 스크립트를 개발하였습니다. PID를 전달하면 PspCidTable을 참조하여 프로세스 오브젝트를 반환합니다. 해당 코드는 [GitHub](https://github.com/cshelldll/Windbg-Scripts/blob/main/LookupPspCidTable)에도 업로드되어 있습니다.
```
$$ nt!ExpLookupHandleTableEntry - Windbg 스크립트 내부 구현
$$ 작성 버전 : Windows 10 Kernel Version 19041 MP (8 procs) Free x64 
$$ 사용 방법 : CheckPspCidTable [pid]
$$ 작성일 : 2022-02-24
$$ 작성자 : cshelldll

r $t0 = poi(nt!PspCidTable)
r $t1 = ${$arg1} & 0xFFFFFFFFFFFFFFFC

.if (@$t1 < poi(@$t0))
{
	r $t2 = poi(@$t0 + 0x08)
	r $t3 = @$t2 & 0x0000FFFF & 3
	
	.if (@$t3 == 1)
	{
		$$ 대부분의 경우 해당 케이스로 빠짐.
		
		r $t4 = @$t1 >> 0x0A
		r $t1 = @$t1 & 0x03FF
		r $t5 = poi(@$t2 + (@$t4 * 8) - 0x01)
		
		.printf "Case 1.\n"
		dqs (@$t5 + @$t1 * 4) L? 2
	}
	.elsif (@$t3 == 0)
	{
		$$ 테스트 X 구현만 미리함.
		
		.printf "Case 2.\n"
		dqs (@$t2 + @$t1 * 4) L? 2
	}
	.else
	{
		$$ 테스트 X 구현만 미리함.
		
		r $t4 = @$t1 >> 0x0A
		r $t5 = @$t4
		r $t4 = @$t4 &0x01FF
		r $t5 = @$t5 >> 0x09
		r $t1 = @$t1 & 0x03FF
		
		r $t5 = poi(@$t2 + @$t5 * 8 - 2)
		r $t5 = poi(@$t5 + @$t4 * 8)
		
		.printf "Case 3.\n"
		dqs (@$t5 + @$t1 * 4) L? 2
	}
}
```
다음과 같이 사용이 가능하며, EPROCESS 주소로 변환하려는 경우 SHR(0x10)을 추가로 연산하면 됩니다.
```
0: kd> $$>a< C:\scripts\CheckPspCidTable 0x328
Case 1.
ffff8a09`934b4ca0  9a8163e1`4140acc1 // SHR(0x10) => ffff9a8163e14140
ffff8a09`934b4ca8  00000000`00000000
```
아래는 ActiveProcessLinks를 통해 획득한 EPROCESS 주소입니다. PspCidTable을 통해 획득한 주소와 비교 시 동일하며, 스크립트가 정상 동작하는 것을 확인할 수 있습니다.
```
0: kd> !process 0 0 csrss.exe
PROCESS ffff9a8163e14140
    SessionId: 1  Cid: 0328    Peb: 1ef594b000  ParentCid: 0320
    DirBase: 10d61f002  ObjectTable: ffff8a0996fb8b80  HandleCount: 670.
    Image: csrss.exe

0: kd> dt_eprocess ffff9a8163e14140
ntdll!_EPROCESS
   +0x440 UniqueProcessId  : 0x00000000`00000328 Void

0: kd> $$>a< C:\scripts\CheckPspCidTable 0x328
Case 1.
ffff8a09`934b4ca0  9a8163e1`4140acc1
ffff8a09`934b4ca8  00000000`00000000
```

## Null Object
해킹툴이 적용된 프로세스의 경우, 다음과 같이 테이블이 조작되어 null 또는 다른 프로세스 오브젝트를 반환하는 경우가 존재합니다. NULL 값으로 조작된 경우에는 비교적 빠르게 확인이 가능합니다.
```
// Null Object
0: kd> $$>a< C:\scripts\CheckPspCidTable 0x450
Case 1.
ffff8a09`97a57140  00000000`00000000 // 9a816469`43000001 => NULL
ffff8a09`97a57148  00000000`00000000
```

## Dummy Process
다음과 같이 더미 프로세스를 반환하도록 조작되면 확인이 쉽지 않습니다. ActiveProcessLinks 등을 통해 획득한 원본 eprocess 주소와 pid를 비교해 확인이 가능합니다.
```
0: kd> !process 0 0 dwm.exe
PROCESS ffff9a8164694300
    SessionId: 1  Cid: 0450    Peb: 6232d5f000  ParentCid: 0374
    DirBase: 114606002  ObjectTable: ffff8a0997ac2b40  HandleCount: 1260.
    Image: dwm.exe

0: kd> dt_eprocess ffff9a8164694300
ntdll!_EPROCESS
   +0x440 UniqueProcessId  : 0x00000000`00000450 Void

// Dummy Object
0: kd> $$>a< C:\scripts\CheckPspCidTable 0x450
Case 1.
ffff8a09`97a57140  9a8163e1`41400001 // dwm.exe => dummy.exe
ffff8a09`97a57148  00000000`00000000

0: kd> dt_eprocess ffff9a8163e1`4140
ntdll!_EPROCESS
   +0x440 UniqueProcessId  : 0x00000000`00000328 Void
      +0x5a8 ImageFileName    : [15]  "dummy.exe"
```

## POC Code
분석된 내용을 바탕으로 PspCidTable를 조작하는 도구를 개발하였습니다. 전체 소스코드는 [GitHub](https://github.com/cshelldll/MyPOC/tree/main/PCTSample)에 업로드 하였습니다.

아래는 ntkrnlmp 내에서 PspCidTable을 찾는 코드입니다. Export 된 변수가 아니기 때문에 시그니처를 통해 스캔합니다.
```cpp
ULONG64 SearchPspCidTable(ULONG64 pagestart, ULONG64 pageend) {
	BYTE PspCidTableCode[] = { 0x66,0xFF,0x89,0xE6,0x01,0x00,0x00,0x49,0x8B,0xF0,0x48,0x8B,0xF9 };

	ULONG64 Address = (ULONG64)HideMmSearchMemory((PVOID)pagestart, pageend - pagestart, PspCidTableCode, sizeof(PspCidTableCode));
	if (Address != 0) {
		Address = Address + 13;
		ULONG OffSet = *(PULONG)(Address + 3);
		OffSet = (ULONG)Address + OffSet + 7;

		Address = (Address & 0xFFFFFFFF00000000) + OffSet;
		Address = *(PULONG64)Address;
	}
	return Address;
}
```

검색된 PspCidTable 내에서 PID가 참조하고 있는 위치를 0으로 초기화합니다. 이렇게 되면 EPROCESS 대신 NULL 값을 반환하여 프로세스 관련된 함수들이 정상적으로 동작하지 않습니다.
```cpp
unsigned __int64 ExpLookupHandleTableEntry(unsigned int* a1, __int64 a2)
{
	unsigned __int64 v2; // rdx
	__int64 v3; // r8
	__int64 v4; // rax
	__int64 v5; // rax

	v2 = a2 & 0xFFFFFFFFFFFFFFFCui64;
	if (v2 >= *a1)
		return 0i64;
	v3 = *((ULONG64*)a1 + 1);
	v4 = *((ULONG64*)a1 + 1) & 3i64;
	if ((ULONG)v4 == 1)
	{
		v5 = *(ULONG64*)(v3 + 8 * (v2 >> 10) - 1);
		return v5 + 4 * (v2 & 0x3FF);
	}
	if ((ULONG)v4)
	{
		v5 = *(ULONG64*)(*(ULONG64
			*)(v3 + 8 * (v2 >> 19) - 2) + 8 * ((v2 >> 10) & 0x1FF));
		return v5 + 4 * (v2 & 0x3FF);
	}
	return v3 + 4 * v2;
}

VOID SetPspCidTable(ULONG64 Pid) 
{
	ULONG64 PspCidTable = 0;

	HideSearch(&PspCidTable);

	if (PspCidTable == 0) {
		KeBugCheckEx(0, 0, 0, 0, 0);
	}

	PULONG64 handle_table_entry = (PULONG64)ExpLookupHandleTableEntry((PVOID)PspCidTable, Pid);

	*handle_table_entry = 0;
}
```

또는 다음과 같이 프로세스 오브젝트를 넣어주면 다른 프로세스로 위장하거나, 원본 프로세스가 반환되도록 복원이 가능합니다.
```cpp
VOID SetPspCidTable(ULONG64 Pid, ULONG64 eprocess)
{

	ULONG64 PspCidTable = 0;

	HideSearch(&PspCidTable);

	if (PspCidTable == 0) {
		KeBugCheckEx(0, 0, 0, 0, 0);
	}

	PULONG64 handle_table_entry = (PULONG64)ExpLookupHandleTableEntry((PVOID)PspCidTable, Pid);

	/* PspReferenceCidTableEntry
	if ( !(unsigned __int8)ExLockHandleTableEntry(PspCidTable, v4) )
	  return 0i64;

	v11 = (struct _DMA_ADAPTER *)((*(__int64 *)v4 >> 0x10) & 0xFFFFFFFFFFFFFFF0ui64);
	*/

	/* windbg dump
	2: kd> $$>a< C:\scripts\CheckPspCidTable 100
	Case 1.
	ffffe30d`a8ab0060  a48a7bb3`90400001

	a48a7bb3`90400001 >> 0x10 = EPROCESS
	EPROCESS << 0x10 | 0x01 = a48a7bb3`90400001
	*/

	if (eprocess)
		*handle_table_entry = eprocess << 0x10 | 0x01;
	else
		*handle_table_entry = 0;
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPat)
{
	UNREFERENCED_PARAMETER(DriverObject);
	UNREFERENCED_PARAMETER(RegistryPat);

	PEPROCESS eprocess;
	HANDLE pid;
	if (FindProcessByName("dwm.exe", &eprocess))
	{
		pid = GetProcessId(eprocess);

		DbgPrint("PID : %d\n", pid);
		DbgPrint("EPROCESS : 0x%p\n", eprocess);

		// Dummy Process
		PEPROCESS dummy_process;
		SetPspCidTable((ULONG64)pid, (ULONG64)dummy_process);

		// Original Process
		PEPROCESS original_process;
		SetPspCidTable((ULONG64)pid, (ULONG64)original_process);
	}
	return STATUS_UNSUCCESSFUL;
}
```
작업관리자를 통해 종료를 시도 시, 보호가 적용된 것을 확인할 수 있습니다. 
![](/assets/posts/2023-11-08-PspCidTable/6.png)
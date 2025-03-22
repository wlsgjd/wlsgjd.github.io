---
title: Find Hidden Process with windbg
date: 2025-03-21 00:00:00 +/-TTTT
categories: [Analysis Report]
tags: [Windows Kernel]
---

## Rootkit
최근 발견되는 해킹툴 대부분은 루트킷 기능을 포함하고 있습니다.

루트킷은 ActiveProcessLinks와 같은 커널 오브젝트를 수정하여 사용자로부터 프로세스를 숨기는 행위입니다.
Windows 10 버전에서는 서명되지 않은 드라이버는 허용하지 않습니다. 

그렇기에, 해킹툴 개발자들은 [kdmapper](https://github.com/TheCruZ/kdmapper)와 같은 잘 알려진 드라이버 취약점을 이용하거나, 탈취 된 인증서 등을 이용하여 루트킷 기능을 포함한 해킹툴 드라이버를 로드합니다.

이러한 루트킷 프로세스들은 작업관리자와 같은 Process Viewer에서는 확인이 어렵고, 일반적인 방식으로는 덤프 파일에서도 확인이 되지 않습니다.
![](/assets/posts/2025-03-21-FindHiddenProcess/1.png)

해당 페이지에서는 이러한 숨은 프로세스들을 찾는 방법들에 대하여 리뷰합니다.

## ActiveProcessLinks
예전부터 가장 많이 사용되고 있는 루트킷 방식입니다. 

윈도우에서 프로세스를 실행하게 되면 eprocess라는 커널 오브젝트가 생성됩니다. 
해당 구조체에는 프로세스에 대한 다양한 정보들이 포함되어 있으며, 이 중에서 ActiveProcessLinks라는 멤버가 루트킷에 활용됩니다.
```
12: kd> dt_eprocess
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : Ptr64 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY
```

ActiveProcessLinks는 리스트 형태로 구성된 프로세스 목록입니다. User Level에서 CreateToolhelp32Snapshot, EnumProcess 등을 통해 프로세스 목록을 가져올 때 참조됩니다.

Flink(Forward Link)는 다음 노드, Blink(Backward Link)는 이전 노드를 의미합니다. 해당 노드를 조작하면 작업관리자와 같은 프로세스 목록에서 사라지게 할 수 있습니다. 오래된 기법이지만 현재까지도 많은 해킹툴에서 사용되고 있습니다.
```
12: kd> dx -id 0,0,ffff8287ad34f040 -r1 (*((ntkrnlmp!_LIST_ENTRY *)0xffff8287be79c548))
(*((ntkrnlmp!_LIST_ENTRY *)0xffff8287be79c548))                 [Type: _LIST_ENTRY]
    [+0x000] Flink            : 0xffff8287be7784c8 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0xffff8287be72e5c8 [Type: _LIST_ENTRY *]
```

아래는 ActiveProcessLinks를 통해 실행 중인 프로세스를 열거하는 windbg 스크립트 입니다. [!process](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/debuggercmds/-process)와 동일한 기능을 수행하며, 스크립트 파일은 [GitHub](https://github.com/wlsgjd/Windbg-Scripts/blob/main/CheckActiveProcessLinks)에 업로드 되어있습니다. 
```
$$ ActiveProcessLinks 프로세스 리스트
$$ 작성 버전 : Windows 10 Kernel Version 19041 MP (4 procs) Free x64 
$$ 사용 방법 : CheckActiveProcessLinks
$$ 작성일 : 2022-03-16
$$ 작성자 : wlsgjd

$$ ProcessEntry
r $t0 = poi(nt!PsActiveProcessHead)-448 
r $t1 = @$t0
r $t2 = 0

.while(@$t0)
{
    .printf "%d %d %p %ma\n", @$t2, poi(@$t0+440),  @$t0, @$t0 + 5a8
    r $t0 = poi(@$t0+448+8)-448
    r $t2 = @$t2+1

    .if (@$t0 == @$t1)
    {
        .break
    }
}
```

스크립트를 실행하면 PsActiveProcessHead 참조하여 프로세스 목록을 열거합니다. ActiveProcessLinks를 변조하는 루트킷은 검출하기 어렵지만, 간혹 FileName이나 PID를 변조하는 루트킷도 존재하기에 이러한 프로세스는 확인할 수 있습니다.
```
12: kd> $$>< C:\git\Windbg-Scripts\CheckActiveProcessLinks
0 4 ffff8287ad34f040 System
1 -1389366624 fffff80055c37a38 
2 10644 ffff8287c7427080 svchost.exe
3 14296 ffff8287c74240c0 conhost.exe
4 14280 ffff8287c73a90c0 VSHiveStub.exe
5 14240 ffff8287c72bf0c0 sppsvc.exe
6 14196 ffff8287c72cc080 mscorsvw.exe
```

## HandleTable
핸들 테이블은 ActiveProcessLinks와 함께 오랜 기간동안 사용된 루트킷 탐지 기법 중 하나입니다.

프로세스 핸들에 대한 테이블로 보이며, UniqueProcessId에는 프로세스의 PID, QuotaProcess에는 EPROCESS 주소가 포함되어 있습니다.
```
12: kd> dt_handle_table
nt!_HANDLE_TABLE
   +0x000 NextHandleNeedingPool : Uint4B
   +0x004 ExtraInfoPages   : Int4B
   +0x008 TableCode        : Uint8B
   +0x010 QuotaProcess     : Ptr64 _EPROCESS
   +0x018 HandleTableList  : _LIST_ENTRY
   +0x028 UniqueProcessId  : Uint4B
   +0x02c Flags            : Uint4B
   +0x02c StrictFIFO       : Pos 0, 1 Bit
   +0x02c EnableHandleExceptions : Pos 1, 1 Bit
   +0x02c Rundown          : Pos 2, 1 Bit
   +0x02c Duplicated       : Pos 3, 1 Bit
   +0x02c RaiseUMExceptionOnInvalidHandleClose : Pos 4, 1 Bit
   +0x030 HandleContentionEvent : _EX_PUSH_LOCK
   +0x038 HandleTableLock  : _EX_PUSH_LOCK
   +0x040 FreeLists        : [1] _HANDLE_TABLE_FREE_LIST
   +0x040 ActualEntry      : [32] UChar
   +0x060 DebugInfo        : Ptr64 _HANDLE_TRACE_DEBUG_INFO
```

ActiveProcessLinks와 마찬가지로 리스트 형태로 구성되어 있으며, HandleTableList를 통해 프로세스를 열거할 수 있습니다. ActiveProcessLinks만 조작된 경우에는 핸들 테이블 열거를 통해 루트킷 검출이 가능합니다. 하지만 대부분 루트킷 개발자들도 이정도는 이미 알고 있기에 함께 패치하는 경우가 대부분입니다.

아래는 windbg에서 핸들 테이블을 통해 프로세스를 열거하는 스크립트입니다. QuotaProcess를 참조하는게 더 정석으로 느껴지지만, 이미 몇년 전에 제작된 스크립트기에... 수정은 따로 하지 않겠습니다. 해당 스크립트는 [GitHub](https://github.com/wlsgjd/Windbg-Scripts/blob/main/CheckHandleTableList)에 업로드 되어있습니다.
```
$$ HandleTableList 프로세스 리스트
$$ 작성 버전 : Windows 10 Kernel Version 19041 MP (4 procs) Free x64 
$$ 사용 방법 : CheckHandleTableList
$$ 작성일 : 2022-03-16
$$ 작성자 : wlsgjd

$$ ProcessEntry
r $t0 = poi(nt!HandleTableListHead + 0x08)
r $t1 = nt!HandleTableListHead
r $t2 = 0

.while(@$t0)
{
    r $t3 = poi(@$t0 - 0x08)
    .if (@$t3 == 0) { .break }

    .printf "%d %d %p %ma\n", @$t2, poi(@$t3+440),  @$t3, @$t3 + 5a8
    r $t2 = @$t2+1
    r $t0 = poi(@$t0+0x08)
    .if (@$t0 == @$t1) { .break }
}
```

사용하면 다음과 같이 실행중인 프로세스 목록을 보여줍니다.
```
12: kd> $$>< C:\git\Windbg-Scripts\CheckHandleTableList
0 10644 ffff8287c7427080 svchost.exe
1 14296 ffff8287c74240c0 conhost.exe
2 14280 ffff8287c73a90c0 VSHiveStub.exe
3 14240 ffff8287c72bf0c0 sppsvc.exe
4 14196 ffff8287c72cc080 mscorsvw.exe
5 13504 ffff8287c6ff1080 SecurityHealth
6 10148 ffff8287c6ec20c0 vmmemCmSnapsho
7 11624 ffff8287c6e05080 vmwp.exe
8 12312 ffff8287c6e7a080 svchost.exe
9 13268 ffff8287c6e070c0 MicrosoftEdgeU
10 13092 ffff8287c6d11080 svchost.exe
```

## MmProcessLinks
몇년 전, 해킹툴을 분석하던 중에 루트킷을 탐지하는 방법에 대해서 생각해보다가, eprocess 내에서 리스트 형태로 된 다른 멤버를 확인하였습니다. 해당 멤버는 메모리 관리자(**M**emory **M**anager)에서 사용하는 프로세스 리스트라고 합니다. (feat. ChatGPT)
```
12: kd> dt_eprocess ffff8287be79c100
nt!_EPROCESS
  +0x7c0 MmProcessLinks   : _LIST_ENTRY [ 0xffff8287`be778840 - 0xffff8287`be72e940 ]
```

ActiveProcessLinks와 비슷한 형태로 보이기에 이를 열거했을 때 어떤지 확인해봤습니다. 테스트 하였을 때 앞서 설명한 ActiveProcessLinks, HandleTableList이 조작된 프로세스도 정상적으로 확인되었습니다.
```
12: kd> $$>< C:\git\Windbg-Scripts\CheckMmProcessLinks
0 4 ffff8287ad34f040 System
1 0 fffff80055d48f40 Idle
2 -1 fffff80055c64dd8 t?
3 10644 ffff8287c7427080 svchost.exe
4 14296 ffff8287c74240c0 conhost.exe
5 14280 ffff8287c73a90c0 VSHiveStub.exe
6 14240 ffff8287c72bf0c0 sppsvc.exe
7 14196 ffff8287c72cc080 mscorsvw.exe
8 14136 ffff8287c72c2080 mscorsvw.exe
```

최근 분석했던 해킹툴 중에서는 MmProcessLinks까지 조작하여 숨기는 케이스는 확인하지 못했습니다. 그렇기에 저는 대부분 업무에서 해당 부분을 확인하여 루트킷을 확인하고 있습니다. (가장 간편함)

해당 스크립트는 [GitHub](https://github.com/wlsgjd/Windbg-Scripts/blob/main/CheckMmProcessLinks)에 업로드 되어있습니다.
```
$$ MmProcessLinks 프로세스 리스트
$$ 작성 버전 : Windows 10 Kernel Version 19041 MP (4 procs) Free x64 
$$ 사용 방법 : CheckMmProcessLinks
$$ 작성일 : 2022-03-16
$$ 작성자 : wlsgjd

$$ ProcessEntry
r $t0 = poi(nt!KiInitialProcess+0x7c0)-0x7c0
r $t1 = @$t0
r $t2 = 0

.while(@$t0)
{
    .printf "%d %d %p %ma\n", @$t2, poi(@$t0+440),  @$t0, @$t0 + 5a8
    r $t0 = poi(@$t0+7c0+8)-7c0
    r $t2 = @$t2+1

    .if (@$t0 == @$t1)
    {
        .break
    }
}
```

## PspCidTable
PspCidTable은 프로세스 핸들 테이블입니다. nt!PspCidTable라는 전역 변수를 통해 접근이 가능하며, 이는 Export 되지 않았기에 일부 해킹툴에서는 다음과 같이 시그니처를 통해 주소를 구해오는 것을 확인했습니다. 샘플 코드는 제 [GitHub](https://github.com/wlsgjd/MyPOC/tree/main/PCTSample)에 업로드 되어 있습니다.
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

윈도우에서는 프로세스 핸들에 접근할 때 ExpLookupHandleTableEntry라는 함수를 통해 해당 테이블에 접근합니다. 파라미터로는 nt!PspCidTable 전역 변수와 대상 프로세스의 PID를 전달합니다. 
```
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

함수 내부에서는 다음과 같은 연산을 통해 PspCidTable 내에 존재하는 프로세스 오브젝트를 반환합니다.
```
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
```

해당 함수를 스크립트 형태로 제작하면 다음과 같습니다. [GitHub](https://github.com/wlsgjd/Windbg-Scripts/blob/main/LookupPspCidTable)에 업로드 되어 있습니다.
```
$$ nt!ExpLookupHandleTableEntry - Windbg 스크립트 내부 구현
$$ 작성 버전 : Windows 10 Kernel Version 19041 MP (8 procs) Free x64 
$$ 사용 방법 : CheckPspCidTable [pid]
$$ 작성일 : 2022-02-24
$$ 작성자 : wlsgjd

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

다음과 같이 사용이 가능하며, EPROCESS 주소로 변환하려는 경우 SHR(0x10)을 추가로 연산하면 됩니다. 이를 활용하여 PspCidTable 내에 존재하는 모든 프로세스 오브젝트에 접근하면 실행 중인 프로세스를 열거할 수 있습니다.
```
0: kd> $$>a< C:\scripts\CheckPspCidTable 0x328
Case 1.
ffff8a09`934b4ca0  9a8163e1`4140acc1 // SHR(0x10) => ffff9a8163e14140
ffff8a09`934b4ca8  00000000`00000000
```

## PoolTag
프로세스는 실행 시 eprocess라는 프로세스 오브젝트를 생성합니다. 

해당 메모리는 nonpaged pool 영역에 할당되며 "Proc"이라는 풀태그를 가지고 있습니다.
현재 덤프에서 확인된 프로세스 오브젝트의 사이즈는 0xE80인데, 이 부분도 참고하면 좋을 것 같습니다.
```
12: kd> !process 0 0 wevtutil.exe
PROCESS ffff8287b5c5b0c0
    SessionId: 0  Cid: 26b8    Peb: 3613048000  ParentCid: 1408
    DirBase: 14caac000  ObjectTable: 00000000  HandleCount:   0.
    Image: wevtutil.exe

12: kd> !pool ffff8287b5c5b0c0
Pool page ffff8287b5c5b0c0 region is Nonpaged pool
*ffff8287b5c5b050 size:  e80 previous size:    0  (Allocated) *Proc
		Pooltag Proc : Process objects, Binary : nt!ps
```

이를 활용하여 "Proc" 태그를 가진 메모리를 검색하면 숨은 프로세스를 찾을 수 있습니다.
```
12: kd> !poolfind -nonpaged -tag "Proc"

Scanning large pool allocation table for tag 0x636f7250 (Proc) (ffff8287ae4a0000 : ffff8287ae6a0000)

ffff8287befc4060 : tag Proc, size     0xe70, Nonpaged pool
ffff8287b5c5f060 : tag Proc, size     0xe70, Nonpaged pool
ffff8287ad34f000 : tag Proc, size    0x1000, Nonpaged pool
ffff8287be0da0d0 : tag Proc, size     0xe70, Nonpaged pool
ffff8287c01e0050 : tag Proc, size     0xe70, Nonpaged pool
ffff8287bef8f060 : tag Proc, size     0xe70, Nonpaged pool
ffff8287c07f3030 : tag Proc, size     0xe70, Nonpaged pool
ffff8287c01e2050 : tag Proc, size     0xe70, Nonpaged pool
*ffff8287b5c5b060 : tag Proc, size     0xe70, Nonpaged pool
ffff8287c0716050 : tag Proc, size     0xe70, Nonpaged pool
ffff8287ba7850d0 : tag Proc, size     0xe70, Nonpaged pool
ffff8287ba0f1140 : tag Proc, size     0xe70, Nonpaged pool
ffff8287c0718050 : tag Proc, size     0xe70, Nonpaged pool
ffff8287bf7ab060 : tag Proc, size     0xe70, Nonpaged pool
ffff8287c3bf8050 : tag Proc, size     0xe70, Nonpaged pool
ffff8287c0ea0030 : tag Proc, size     0xe70, Nonpaged pool
ffff8287bf7db060 : tag Proc, size     0xe70, Nonpaged pool
```
```
12: kd> dt_pool_header ffff8287b5c5b060-10
nt!_POOL_HEADER
   +0x000 PreviousSize     : 0y00000000 (0)
   +0x000 PoolIndex        : 0y00000000 (0)
   +0x002 BlockSize        : 0y11101000 (0xe8)
   +0x002 PoolType         : 0y00000010 (0x2)
   +0x000 Ulong1           : 0x2e80000
   +0x004 PoolTag          : 0x636f7250 // "Proc"
   +0x008 ProcessBilled    : (null) 
   +0x008 AllocatorBackTraceIndex : 0
   +0x00a PoolTagHash      : 0
```
```
12: kd> dt_object_header ffff8287b5c5b060+30
nt!_OBJECT_HEADER
   +0x000 PointerCount     : 0n3
   +0x008 HandleCount      : 0n0
   +0x008 NextToFree       : (null) 
   +0x010 Lock             : _EX_PUSH_LOCK
   +0x018 TypeIndex        : 0x85 ''
   +0x019 TraceFlags       : 0 ''
   +0x019 DbgRefTrace      : 0y0
   +0x019 DbgTracePermanent : 0y0
   +0x01a InfoMask         : 0x88 ''
   +0x01b Flags            : 0 ''
   +0x01b NewObject        : 0y0
   +0x01b KernelObject     : 0y0
   +0x01b KernelOnlyAccess : 0y0
   +0x01b ExclusiveObject  : 0y0
   +0x01b PermanentObject  : 0y0
   +0x01b DefaultSecurityQuota : 0y0
   +0x01b SingleHandleEntry : 0y0
   +0x01b DeletedInline    : 0y0
   +0x01c Reserved         : 0
   +0x020 ObjectCreateInfo : 0xfffff800`55c6ab80 _OBJECT_CREATE_INFORMATION
   +0x020 QuotaBlockCharged : 0xfffff800`55c6ab80 Void
   +0x028 SecurityDescriptor : 0xffffa106`9eef3b6f Void
   +0x030 Body             : _QUAD
```
```
12: kd> dt_eprocess ffff8287b5c5b060+60
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : 0x00000000`000026b8 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY [ 0xffff8287`b5ee24c8 - 0xffff8287`c0c1d4c8 ]
   +0x458 RundownProtect   : _EX_RUNDOWN_REF
   +0x460 Flags2           : 0xd81f
   +0x460 JobNotReallyActive : 0y1
   +0x460 AccountingFolded : 0y1
   +0x460 NewProcessReported : 0y1
   +0x460 ExitProcessReported : 0y1
   +0x460 ReportCommitChanges : 0y1
   +0x460 LastReportMemory : 0y0
   +0x460 ForceWakeCharge  : 0y0
   +0x460 CrossSessionCreate : 0y0
   +0x460 NeedsHandleRundown : 0y0
   +0x460 RefTraceEnabled  : 0y0
   +0x460 PicoCreated      : 0y0
```

## Signature
PoolTag와 비슷하게 시그니처를 통해서도 nonpaged 영역에 존재하는 프로세스 핸들을 구할 수 있습니다. 다음과 같이 eprocess 메모리를 덤프하여 본인만의 시그니처를 출력합니다.
![](/assets/posts/2025-03-21-FindHiddenProcess/2.png)

그리고 메모리에서 검색해주면 끝입니다. 원래 시그니처는 매우 신중히 뽑아야 하지만, 저는 그냥 테스트기에 대충 뽑았습니다.
```
12: kd> !vm 0x20
System Region               Base Address    NumberOfBytes

Kasan                 :                0                0
NonPagedPool          : ffff828000000000     100000000000
```
```
12: kd> s ffff828000000000 L? 100000000000 06 a1 ff ff 00 00 00 00 64 86 00 00 00 00 00 00 00 00
ffff8287`ad34f9a4  06 a1 ff ff 00 00 00 00-64 86 00 00 00 00 00 00  ........d.......
ffff8287`b5ee29e4  06 a1 ff ff 00 00 00 00-64 86 00 00 00 00 00 00  ........d.......
ffff8287`ba785aa4  06 a1 ff ff 00 00 00 00-64 86 00 00 00 00 00 00  ........d.......
```
```
12: kd> !pool ffff8287`ad34f9a4
Pool page ffff8287ad34f9a4 region is Nonpaged pool
*ffff8287ad34f000 : large page allocation, tag is Proc, size is 0x1000 bytes
		Pooltag Proc : Process objects, Binary : nt!ps

12: kd> dt_eprocess ffff8287ad34f000+40
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : 0x00000000`00000004 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY [ 0xffff8287`ad6cf488 - 0xfffff800`55c37e80 ]
   +0x458 RundownProtect   : _EX_RUNDOWN_REF
   +0x460 Flags2           : 0xd000
   +0x460 JobNotReallyActive : 0y0
   +0x460 AccountingFolded : 0y0
   +0x460 NewProcessReported : 0y0
   ...
   +0x5a8 ImageFileName    : [15]  "System"
```

## References
- [Windows Handle Table & Object](https://shhoya.github.io/windows_hwndobject.html)
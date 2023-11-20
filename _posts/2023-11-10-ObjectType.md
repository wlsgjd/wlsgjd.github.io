---
title: 오브젝트 타입 변조 (ObjectType)
categories: [Analysis Report]
tags: [Windows Kernel]
---

## STATUS_OBJECT_TYPE_MISMATCH (C0000024)
OpenProcess를 호출하면 0xC0000024를 반환하며 실패하는 해킹툴이 발견되었습니다. 

다음과 같이 NtOpenProcess 시스콜 이후에 실패하고 있으며, 이를 봤을 때 커널 레벨에서 조작된 것을 추측할 수 있습니다.
![](/assets/posts/2023-11-10-ObjectType/1.png)

해당 값은 [MS-ERREF](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55) 문서에 나와있는데 요청한 작업과 오브젝트의 유형이 일치하지 않아 발생하는 오류입니다.

| Return value/code   |	Description |
|---------------------|----------------|
| 0xC0000024<br><br>STATUS_OBJECT_TYPE_MISMATCH | {Wrong Type} There is a mismatch between the type<br>of object that is required by the requested operation<br>and the type of object that is specified in the<br>request. |

## Kernel Memory Dump
오브젝트 타입이 변경되었는지 확인해보기 위해 커널 메모리 덤프를 수집하여 분석을 진행하였습니다.

일단 숨어있는 해킹툴 프로세스를 찾고...(ㅜㅅㅜ) 다음과 같이 !object 명령을 통해 프로세스 오브젝트의 타입을 확인해봤습니다. 
```
0: kd> !object FFFFBC015B7550C0
Object: ffffbc015b7550c0  Type: (ffffbc0155ae3bc0) Job
    ObjectHeader: ffffbc015b755090 (new version)
    HandleCount: 14  PointerCount: 4503599627585632
```
정상적인 경우라면 프로세스 오브젝트는 다음과 같이 타입이 Process로 표시되어야 합니다. 이를 통해 해당 오브젝트가 조작된 것을 확인할 수 있습니다.
```
0: kd> !object FFFFBC015B7550C0
Object: ffffbc015b7550c0  Type: (ffffbc0155ae3bc0) Process
    ObjectHeader: ffffbc015b755090 (new version)
    HandleCount: 14  PointerCount: 4503599627585632
```

## ObReferenceObjectByPointerWithTag 
유저 레벨에서 OpenProcess가 호출되면 커널 내부에서 ObReferenceObjectByPointerWithTag를 호출하게 됩니다. 

해당 함수에서는 대상 오브젝트의 타입이 PsProcessType (OBJECT_TYPE_PROCESS)와 일치하는지 검사합니다. 
![](/assets/posts/2023-11-10-ObjectType/3.png)

호출 스택은 다음과 같이 정리할 수 있습니다.

| Ring | Call Stacks                    |
|:-:|-----------------------------------|
| 3 | OpenProcess                       |
| 3 | NtOpenProcess                     |
| 0 | PsOpenProcess                     |
| 0 | ObOpenObjectByPointer             |
| 0 | ObReferenceObjectByPointerWithTag |

ObHeaderCookie, TypeIndex, 오브젝트 헤더의 주소 값을 xor 연산하여 인덱스 값을 생성하고 ObTypeIndexTable 내에서 OBJECT_TYPE 데이터를 가져와 비교합니다.
![](/assets/posts/2023-11-10-ObjectType/2.png)

현재 수집된 해킹툴 덤프에서는 다음과 같은 연산을 통해 ObTypeIndexTable에 참조하는 인덱스 값이 0x07=>0x06으로 조작된 것을 확인할 수 있습니다.
```
// ObjectHeader->TypeIndex
0: kd> db FFFFBC015B7550C0-0x18 
ffffbc01`5b7550a8  fb

FFFFBC015B7550C0-0x30 >> 8 = 00ffffbc`015b7550 = 0x50 (al)

0: kd> db ObHeaderCookie
fffff805`74cfc72c  ad

// result index
// OBJECT_TYPE_PROCESS -> OBJECT_TYPE_JOB
0xFB ^ 0xAD ^ 0x50 = 0x06 

0: kd> dqs ObTypeIndexTable
fffff805`74cfce80  00000000`00000000
fffff805`74cfce88  ffff9081`04d64000
fffff805`74cfce90  ffffbc01`55ae34e0
fffff805`74cfce98  ffffbc01`55ae3d20
fffff805`74cfcea0  ffffbc01`55ae3900
fffff805`74cfcea8  ffffbc01`55ae3a60
fffff805`74cfceb0  ffffbc01`55ae3bc0 // PsJobType
fffff805`74cfceb8  ffffbc01`55af32a0 // PsProcessType
```

## TypeIndex
ObTypeIndexTable를 참조하는 인덱스 값을 변조하려면 다음과 같이 4가지 방법이 존재하는데 1,2,4는 변경 시 다른 오브젝트에도 영향이 가거나 방법이 매우 까다롭기 때문에 사실상 ObjectHeader->TypeIndex를 통한 변조만 가능할 것 같다고 생각됩니다.
1. ObTypeIndexTable
2. ObHeaderCookie
3. ObjectHeader->TypeIndex
4. Object Memory Reallocation

OpenProcess를 호출했을 때, 커널 내부에서 ObjectType을 비교하는 방식을 C++ 코드 형태로 정리하면 대충 다음과 같습니다. 해당 함수를 통해 대상 오브젝트가 PsProcessType인지 검사합니다.
```cpp
BYTE ObHeaderCookie;
OBJECT_TYPE ObTypeIndexTable[];

NTSTATUS __stdcall ObReferenceObjectByPointerWithTag(
        PVOID Object,
        ACCESS_MASK DesiredAccess,
        POBJECT_TYPE ObjectType,
        KPROCESSOR_MODE AccessMode,
        ULONG Tag)
{
    /*
        0: kd> dt_object_header
        nt!_OBJECT_HEADER
            +0x000 PointerCount     : Int8B
            +0x008 HandleCount      : Int8B
            +0x008 NextToFree       : Ptr64 Void
            +0x010 Lock             : _EX_PUSH_LOCK
            +0x018 TypeIndex        : UChar
            +0x019 TraceFlags       : UChar
            +0x019 DbgRefTrace      : Pos 0, 1 Bit
            +0x019 DbgTracePermanent : Pos 1, 1 Bit
            +0x01a InfoMask         : UChar
            +0x01b Flags            : UChar
            +0x01b NewObject        : Pos 0, 1 Bit
            +0x01b KernelObject     : Pos 1, 1 Bit
            +0x01b KernelOnlyAccess : Pos 2, 1 Bit
            +0x01b ExclusiveObject  : Pos 3, 1 Bit
            +0x01b PermanentObject  : Pos 4, 1 Bit
            +0x01b DefaultSecurityQuota : Pos 5, 1 Bit
            +0x01b SingleHandleEntry : Pos 6, 1 Bit
            +0x01b DeletedInline    : Pos 7, 1 Bit
            +0x01c Reserved         : Uint4B
            +0x020 ObjectCreateInfo : Ptr64 _OBJECT_CREATE_INFORMATION
            +0x020 QuotaBlockCharged : Ptr64 Void
            +0x028 SecurityDescriptor : Ptr64 Void
            +0x030 Body             : _QUAD
    */
    POBJECT_HEADER ObjectHeader = Object - 0x30
    int TypeIndex = ObHeaderCookie ^ ObjectHeader->TypeIndex ^ BYTE(ObjectHeader >> 0x08);

    if (ObTypeIndexTable[TypeIndex] != ObjectType)
        return STATUS_OBJECT_TYPE_MISMATCH; // 0xC0000024

    ...
}
```

## ObjectType
오브젝트 타입 리스트는 따로 정리된게 없어보여서 직접 windbg 스크립트를 통해 추출하였습니다. 시스템 프로세스를 대상으로 오브젝트 타입을 변경하고 !object 명령을 통해 심볼명을 출력합니다.

해당 스크립트 파일은 [GitHub](https://github.com/cshelldll/Windbg-Scripts/blob/main/GetObjectTypeList)에 업로드 하였습니다.
```
$$ GetObjectTypeList
$$ 작성 버전 : Windows 10 Kernel Version 19041 MP (4 procs) Free x64 
$$ 사용 방법 : GetObjectTypeList
$$ 작성일 : 2023-11-14
$$ 작성자 : cshelldll

$$ ProcessEntry
r $t0 = poi(nt!PsActiveProcessHead)-448 
r $t1 = poi(@$t0-18) & 0xff
r $t2 = ((@$t0-30) >> 8) & 0xff
r $t3 = poi(ObHeaderCookie) & 0xff

r $t4 = 0
.while(@$t4 < 255)
{
	.printf "[%d]", @$t4
    eb (@$t0 - 0x18) (@$t2 ^ @$t3 ^ @$t4)
	!object $t0 0
	
	r $t4 = @$t4 + 1
}

// Restore ProcessType
eb (@$t0 - 0x18) (@$t2 ^ @$t3 ^ 7)
```
```
0: kd> $$>< C:\scripts\GetObjectTypeList
[0]ffffbc0155aaf040: Not a valid object (ObjectType invalid)
[1]Object: ffffbc0155aaf040  Type: (ffff908104d64000) 
    ObjectHeader: ffffbc0155aaf010 (new version)
[2]Object: ffffbc0155aaf040  Type: (ffffbc0155ae34e0) Type
    ObjectHeader: ffffbc0155aaf010 (new version)
[3]Object: ffffbc0155aaf040  Type: (ffffbc0155ae3d20) Directory
    ObjectHeader: ffffbc0155aaf010 (new version)
[4]Object: ffffbc0155aaf040  Type: (ffffbc0155ae3900) SymbolicLink
    ObjectHeader: ffffbc0155aaf010 (new version)
[5]Object: ffffbc0155aaf040  Type: (ffffbc0155ae3a60) Token
    ObjectHeader: ffffbc0155aaf010 (new version)
[6]Object: ffffbc0155aaf040  Type: (ffffbc0155ae3bc0) Job
    ObjectHeader: ffffbc0155aaf010 (new version)
[7]Object: ffffbc0155aaf040  Type: (ffffbc0155af32a0) Process
    ObjectHeader: ffffbc0155aaf010 (new version)
[8]Object: ffffbc0155aaf040  Type: (ffffbc0155af3820) Thread
    ObjectHeader: ffffbc0155aaf010 (new version)
[9]Object: ffffbc0155aaf040  Type: (ffffbc0155af2640) Partition
    ObjectHeader: ffffbc0155aaf010 (new version)
[10]Object: ffffbc0155aaf040  Type: (ffffbc0155af20c0) UserApcReserve
    ObjectHeader: ffffbc0155aaf010 (new version)
[11]Object: ffffbc0155aaf040  Type: (ffffbc0155af2220) IoCompletionReserve
    ObjectHeader: ffffbc0155aaf010 (new version)
[12]Object: ffffbc0155aaf040  Type: (ffffbc0155af2e80) ActivityReference
    ObjectHeader: ffffbc0155aaf010 (new version)
[13]Object: ffffbc0155aaf040  Type: (ffffbc0155af3400) PsSiloContextPaged
    ObjectHeader: ffffbc0155aaf010 (new version)
[14]Object: ffffbc0155aaf040  Type: (ffffbc0155af2d20) PsSiloContextNonPaged
    ObjectHeader: ffffbc0155aaf010 (new version)
[15]Object: ffffbc0155aaf040  Type: (ffffbc0155af24e0) DebugObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[16]Object: ffffbc0155aaf040  Type: (ffffbc0155af3980) Event
    ObjectHeader: ffffbc0155aaf010 (new version)
[17]Object: ffffbc0155aaf040  Type: (ffffbc0155af2380) Mutant
    ObjectHeader: ffffbc0155aaf010 (new version)
[18]Object: ffffbc0155aaf040  Type: (ffffbc0155af2900) Callback
    ObjectHeader: ffffbc0155aaf010 (new version)
[19]Object: ffffbc0155aaf040  Type: (ffffbc0155af3c40) Semaphore
    ObjectHeader: ffffbc0155aaf010 (new version)
[20]Object: ffffbc0155aaf040  Type: (ffffbc0155af27a0) Timer
    ObjectHeader: ffffbc0155aaf010 (new version)
[21]Object: ffffbc0155aaf040  Type: (ffffbc0155af3560) IRTimer
    ObjectHeader: ffffbc0155aaf010 (new version)
[22]Object: ffffbc0155aaf040  Type: (ffffbc0155af3da0) Profile
    ObjectHeader: ffffbc0155aaf010 (new version)
[23]Object: ffffbc0155aaf040  Type: (ffffbc0155af36c0) KeyedEvent
    ObjectHeader: ffffbc0155aaf010 (new version)
[24]Object: ffffbc0155aaf040  Type: (ffffbc0155af3ae0) WindowStation
    ObjectHeader: ffffbc0155aaf010 (new version)
[25]Object: ffffbc0155aaf040  Type: (ffffbc0155af2a60) Desktop
    ObjectHeader: ffffbc0155aaf010 (new version)
[26]Object: ffffbc0155aaf040  Type: (ffffbc0155af3140) Composition
    ObjectHeader: ffffbc0155aaf010 (new version)
[27]Object: ffffbc0155aaf040  Type: (ffffbc0155af3f00) RawInputManager
    ObjectHeader: ffffbc0155aaf010 (new version)
[28]Object: ffffbc0155aaf040  Type: (ffffbc0155af2bc0) CoreMessaging
    ObjectHeader: ffffbc0155aaf010 (new version)
[29]Object: ffffbc0155aaf040  Type: (ffffbc0155b1dda0) ActivationObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[30]Object: ffffbc0155aaf040  Type: (ffffbc0155b1df00) TpWorkerFactory
    ObjectHeader: ffffbc0155aaf010 (new version)
[31]Object: ffffbc0155aaf040  Type: (ffffbc0155b1d2a0) Adapter
    ObjectHeader: ffffbc0155aaf010 (new version)
[32]Object: ffffbc0155aaf040  Type: (ffffbc0155b1d980) Controller
    ObjectHeader: ffffbc0155aaf010 (new version)
[33]Object: ffffbc0155aaf040  Type: (ffffbc0155b1c4e0) Device
    ObjectHeader: ffffbc0155aaf010 (new version)
[34]Object: ffffbc0155aaf040  Type: (ffffbc0155b1c0c0) Driver
    ObjectHeader: ffffbc0155aaf010 (new version)
[35]Object: ffffbc0155aaf040  Type: (ffffbc0155b1c640) IoCompletion
    ObjectHeader: ffffbc0155aaf010 (new version)
[36]Object: ffffbc0155aaf040  Type: (ffffbc0155b1d820) WaitCompletionPacket
    ObjectHeader: ffffbc0155aaf010 (new version)
[37]Object: ffffbc0155aaf040  Type: (ffffbc0155b1d140) File
    ObjectHeader: ffffbc0155aaf010 (new version)
[38]Object: ffffbc0155aaf040  Type: (ffffbc0155b1d6c0) TmTm
    ObjectHeader: ffffbc0155aaf010 (new version)
[39]Object: ffffbc0155aaf040  Type: (ffffbc0155b1cbc0) TmTx
    ObjectHeader: ffffbc0155aaf010 (new version)
[40]Object: ffffbc0155aaf040  Type: (ffffbc0155b1c900) TmRm
    ObjectHeader: ffffbc0155aaf010 (new version)
[41]Object: ffffbc0155aaf040  Type: (ffffbc0155b1c380) TmEn
    ObjectHeader: ffffbc0155aaf010 (new version)
[42]Object: ffffbc0155aaf040  Type: (ffffbc0155b1d400) Section
    ObjectHeader: ffffbc0155aaf010 (new version)
[43]Object: ffffbc0155aaf040  Type: (ffffbc0155b1d560) Session
    ObjectHeader: ffffbc0155aaf010 (new version)
[44]Cannot find type _CM_NAME_CONTROL_BLOCK.
Object: ffffbc0155aaf040  Type: (ffffbc0155b1c220) Key
    ObjectHeader: ffffbc0155aaf010 (new version)
[45]Object: ffffbc0155aaf040  Type: (ffffbc0155b1dae0) RegistryTransaction
    ObjectHeader: ffffbc0155aaf010 (new version)
[46]Object: ffffbc0155aaf040  Type: (ffffbc0155b1c7a0) ALPC Port
    ObjectHeader: ffffbc0155aaf010 (new version)
[47]Object: ffffbc0155aaf040  Type: (ffffbc0155b1dc40) EnergyTracker
    ObjectHeader: ffffbc0155aaf010 (new version)
[48]Object: ffffbc0155aaf040  Type: (ffffbc0155b1ca60) PowerRequest
    ObjectHeader: ffffbc0155aaf010 (new version)
[49]Object: ffffbc0155aaf040  Type: (ffffbc0155b1ce80) WmiGuid
    ObjectHeader: ffffbc0155aaf010 (new version)
[50]Object: ffffbc0155aaf040  Type: (ffffbc0155bfe900) EtwRegistration
    ObjectHeader: ffffbc0155aaf010 (new version)
[51]Object: ffffbc0155aaf040  Type: (ffffbc0155bfee80) EtwSessionDemuxEntry
    ObjectHeader: ffffbc0155aaf010 (new version)
[52]Object: ffffbc0155aaf040  Type: (ffffbc0155bfe380) EtwConsumer
    ObjectHeader: ffffbc0155aaf010 (new version)
[53]Object: ffffbc0155aaf040  Type: (ffffbc0155bff2a0) CoverageSampler
    ObjectHeader: ffffbc0155aaf010 (new version)
[54]Object: ffffbc0155aaf040  Type: (ffffbc0155bff980) DmaAdapter
    ObjectHeader: ffffbc0155aaf010 (new version)
[55]Object: ffffbc0155aaf040  Type: (ffffbc0155bfea60) PcwObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[56]Object: ffffbc0155aaf040  Type: (ffffbc0155bffae0) FilterConnectionPort
    ObjectHeader: ffffbc0155aaf010 (new version)
[57]Object: ffffbc0155aaf040  Type: (ffffbc01579ee7a0) FilterCommunicationPort
    ObjectHeader: ffffbc0155aaf010 (new version)
[58]Object: ffffbc0155aaf040  Type: (ffffbc01579ee220) NdisCmState
    ObjectHeader: ffffbc0155aaf010 (new version)
[59]Object: ffffbc0155aaf040  Type: (ffffbc01579ef140) DxgkSharedResource
    ObjectHeader: ffffbc0155aaf010 (new version)
[60]Object: ffffbc0155aaf040  Type: (ffffbc01579efae0) DxgkSharedKeyedMutexObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[61]Object: ffffbc0155aaf040  Type: (ffffbc01579ef2a0) DxgkSharedSyncObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[62]Object: ffffbc0155aaf040  Type: (ffffbc01579ef400) DxgkSharedSwapChainObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[63]Object: ffffbc0155aaf040  Type: (ffffbc01579eea60) DxgkDisplayManagerObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[64]Object: ffffbc0155aaf040  Type: (ffffbc01579eee80) DxgkCurrentDxgThreadObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[65]Object: ffffbc0155aaf040  Type: (ffffbc01579eebc0) DxgkSharedProtectedSessionObjec
    ObjectHeader: ffffbc0155aaf010 (new version)
[66]Object: ffffbc0155aaf040  Type: (ffffbc01579ef560) DxgkSharedBundleObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[67]Object: ffffbc0155aaf040  Type: (ffffbc01579ef6c0) DxgkCompositionObject
    ObjectHeader: ffffbc0155aaf010 (new version)
[68]Object: ffffbc0155aaf040  Type: (ffffbc01579eed20) VRegConfigurationContext
    ObjectHeader: ffffbc0155aaf010 (new version)
```
추출하여 정리한 오브젝트 타입 목록은 다음과 같습니다. 
```cpp
typedef enum OBJECT_TYPE
{
	OBJECT_TYPE_NULL,
	OBJECT_TYPE_UNKNOWN,
	OBJECT_TYPE_TYPE,
	OBJECT_TYPE_DIRECTORY,
	OBJECT_TYPE_SYMBOLICLINK,
	OBJECT_TYPE_TOKEN,
	OBJECT_TYPE_JOB,
	OBJECT_TYPE_PROCESS,
	OBJECT_TYPE_THREAD,
	OBJECT_TYPE_PARTITION,
	OBJECT_TYPE_USERAPCRESERVE,
	OBJECT_TYPE_IOCOMPLETIONRESERVE,
	OBJECT_TYPE_ACTIVITYREFERENCE,
	OBJECT_TYPE_PSSILOCONTEXTPAGED,
	OBJECT_TYPE_PSSILOCONTEXTNONPAGED,
	OBJECT_TYPE_DEBUGOBJECT,
	OBJECT_TYPE_EVENT,
	OBJECT_TYPE_MUTANT,
	OBJECT_TYPE_CALLBACK,
	OBJECT_TYPE_SEMAPHORE,
	OBJECT_TYPE_TIMER,
	OBJECT_TYPE_IRTIMER,
	OBJECT_TYPE_PROFILE,
	OBJECT_TYPE_KEYEDEVENT,
	OBJECT_TYPE_WINDOWSTATION,
	OBJECT_TYPE_DESKTOP,
	OBJECT_TYPE_COMPOSITION,
	OBJECT_TYPE_RAWINPUTMANAGER,
	OBJECT_TYPE_COREMESSAGING,
	OBJECT_TYPE_ACTIVATIONOBJECT,
	OBJECT_TYPE_TPWORKERFACTORY,
	OBJECT_TYPE_ADAPTER,
	OBJECT_TYPE_CONTROLLER,
	OBJECT_TYPE_DEVICE,
	OBJECT_TYPE_DRIVER,
	OBJECT_TYPE_IOCOMPLETION,
	OBJECT_TYPE_WAITCOMPLETIONPACKET,
	OBJECT_TYPE_FILE,
	OBJECT_TYPE_TMTM,
	OBJECT_TYPE_TMTX,
	OBJECT_TYPE_TMRM,
	OBJECT_TYPE_TMEN,
	OBJECT_TYPE_SECTION,
	OBJECT_TYPE_SESSION,
	OBJECT_TYPE_KEY,
	OBJECT_TYPE_REGISTRYTRANSACTION,
	OBJECT_TYPE_ALPC_PORT,
	OBJECT_TYPE_ENERGYTRACKER,
	OBJECT_TYPE_POWERREQUEST,
	OBJECT_TYPE_WMIGUID,
	OBJECT_TYPE_ETWREGISTRATION,
	OBJECT_TYPE_ETWSESSIONDEMUXENTRY,
	OBJECT_TYPE_ETWCONSUMER,
	OBJECT_TYPE_COVERAGESAMPLER,
	OBJECT_TYPE_DMAADAPTER,
	OBJECT_TYPE_PCWOBJECT,
	OBJECT_TYPE_FILTERCONNECTIONPORT,
	OBJECT_TYPE_FILTERCOMMUNICATIONPORT,
	OBJECT_TYPE_NDISCMSTATE,
	OBJECT_TYPE_DXGKSHAREDRESOURCE,
	OBJECT_TYPE_DXGKSHAREDKEYEDMUTEXOBJECT,
	OBJECT_TYPE_DXGKSHAREDSYNCOBJECT,
	OBJECT_TYPE_DXGKSHAREDSWAPCHAINOBJECT,
	OBJECT_TYPE_DXGKDISPLAYMANAGEROBJECT,
	OBJECT_TYPE_DXGKCURRENTDXGTHREADOBJECT,
	OBJECT_TYPE_DXGKSHAREDPROTECTEDSESSIONOBJEC,
	OBJECT_TYPE_DXGKSHAREDBUNDLEOBJECT,
	OBJECT_TYPE_DXGKCOMPOSITIONOBJECT,
	OBJECT_TYPE_VREGCONFIGURATIONCONTEXT
}OBJECT_TYPE;
```

## ObTypeIndexTable
글을 쓰면서 정리를 해봤는데... 생각해보니 위에 방법처럼 ObjectType 리스트를 추출하는 것보다 ObTypeIndexTable를 이용하면 더 간편하게 추출이 가능할 것 같습니다.

ObTypeIndexTable에는 다음과 같이 object_type 데이터가 존재하는데 해당 부분의 Name을 하나씩 추출하면 더 간편하게 리스트업이 가능할 것 같습니다. 

필요하거나 관심있으신 분들은 직접 한번 스크립트를 제작해보시면 좋을 것 같아요.
```
0: kd> dqs ObTypeIndexTable
fffff805`74cfce80  00000000`00000000
fffff805`74cfce88  ffff9081`04d64000
fffff805`74cfce90  ffffbc01`55ae34e0
fffff805`74cfce98  ffffbc01`55ae3d20
fffff805`74cfcea0  ffffbc01`55ae3900
fffff805`74cfcea8  ffffbc01`55ae3a60
fffff805`74cfceb0  ffffbc01`55ae3bc0
fffff805`74cfceb8  ffffbc01`55af32a0
fffff805`74cfcec0  ffffbc01`55af3820
fffff805`74cfcec8  ffffbc01`55af2640
fffff805`74cfced0  ffffbc01`55af20c0
fffff805`74cfced8  ffffbc01`55af2220
fffff805`74cfcee0  ffffbc01`55af2e80
fffff805`74cfcee8  ffffbc01`55af3400
fffff805`74cfcef0  ffffbc01`55af2d20
fffff805`74cfcef8  ffffbc01`55af24e0
```
```
0: kd> dt_object_type ffffbc01`55ae34e0
nt!_OBJECT_TYPE
   +0x000 TypeList         : _LIST_ENTRY [ 0xffffbc01`55ae3490 - 0xffffbc01`579eecd0 ]
   +0x010 Name             : _UNICODE_STRING "Type"
   +0x020 DefaultObject    : 0xfffff805`74c25960 Void
   +0x028 Index            : 0x2 ''
   +0x02c TotalNumberOfObjects : 0x43
   +0x030 TotalNumberOfHandles : 0
   +0x034 HighWaterNumberOfObjects : 0x43
   +0x038 HighWaterNumberOfHandles : 0
   +0x040 TypeIndex         : _OBJECT_TYPE_INITIALIZER
   +0x0b8 TypeLock         : _EX_PUSH_LOCK
   +0x0c0 Key              : 0x546a624f
   +0x0c8 CallbackList     : _LIST_ENTRY [ 0xffffbc01`55ae35a8 - 0xffffbc01`55ae35a8 ]
```

## POC Code
분석한 내용을 바탕으로 해킹툴과 동일하게 동작하는 코드를 개발하였습니다.
전체 소스코드는 [GitHub](https://github.com/cshelldll/MyPOC/tree/main/ProcessObjType)에 업로드하였습니다.

시스템 프로세스의 오브젝트 타입이 OBJECT_TYPE_PROCESS 고정인 것을 이용하여 역연산을 통해 ObHeaderCookie 값을 획득하고, 해당 값을 통해 대상 프로세스의 TypeIndex를 변경합니다.

```cpp
BOOL GetObHeaderCookie(BYTE* ObHeaderCookie)
{
	BYTE* eprocess = 0;
	if (FindProcessByName("System", (PEPROCESS*)&eprocess))
	{
		BYTE num1 = *(eprocess - 0x18);
		BYTE num2 = (BYTE)(((ULONG64)((BYTE*)eprocess - 0x30)) >> 8);

		*ObHeaderCookie = num1 ^ num2 ^ OBJECT_TYPE_PROCESS;

		return TRUE;
	}
	return FALSE;
}

BOOL SetObjectType(BYTE* eprocess, BYTE ObjectType)
{
	BYTE ObHeaderCookie;
	GetObHeaderCookie(&ObHeaderCookie);

	BYTE* objtype = (BYTE*)eprocess - 0x18;

	BYTE num1 = (BYTE)(((ULONG64)eprocess - 0x30) >> 8);
	*objtype = ObjectType ^ ObHeaderCookie ^ num1;

	return TRUE;
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
	UNREFERENCED_PARAMETER(DriverObject);
	UNREFERENCED_PARAMETER(RegistryPath);

	PEPROCESS eprocess;
	if (FindProcessByName("notepad.exe", &eprocess))
	{
		SetObjectType((BYTE*)eprocess, OBJECT_TYPE_JOB);
	}

	return STATUS_UNSUCCESSFUL;
}
```

적용 시 다음과 같이 보호가 적용되어 OpenProcess가 실패합니다.
![](/assets/posts/2023-11-10-ObjectType/4.png)
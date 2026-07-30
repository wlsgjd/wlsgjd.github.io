---
title: ObRegisterCallbacks
date: 2026-07-30 16:39:00 +0900
categories: [Windows Internals]
tags: [Windows Internals]
---

## ObRegisterCallbacks
**ObRegisterCallbacks**은 스레드, 프로세스 및 데스크톱 핸들 작업에 대한 콜백 루틴을 등록하는 함수입니다.
```c
NTSTATUS ObRegisterCallbacks(
    _In_  POB_CALLBACK_REGISTRATION CallbackRegistration,
    _Out_ PVOID* RegistrationHandle
);
```
```c
typedef struct _OB_CALLBACK_REGISTRATION {
  USHORT                    Version;
  USHORT                    OperationRegistrationCount;
  UNICODE_STRING            Altitude;
  PVOID                     RegistrationContext;
  OB_OPERATION_REGISTRATION *OperationRegistration;
} OB_CALLBACK_REGISTRATION, *POB_CALLBACK_REGISTRATION;
```

프로세스의 경우 **PsProcessType**, 스레드의 경우 **PsThreadType** 내 콜백 리스트에 등록됩니다.
```
2: kd> dqs PsProcessType
fffff804`57afc410  ffffb502`1dcf50c0

2: kd> dt_object_type ffffb502`1dcf50c0
nt!_OBJECT_TYPE
   +0x000 TypeList         : _LIST_ENTRY [ 0xffffb502`1dcf50c0 - 0xffffb502`1dcf50c0 ]
   +0x010 Name             : _UNICODE_STRING "Process"
   +0x020 DefaultObject    : (null) 
   +0x028 Index            : 0x7 ''
   +0x02c TotalNumberOfObjects : 0xc6
   +0x030 TotalNumberOfHandles : 0x85c
   +0x034 HighWaterNumberOfObjects : 0xd6
   +0x038 HighWaterNumberOfHandles : 0x91e
   +0x040 TypeInfo         : _OBJECT_TYPE_INITIALIZER
   +0x0b8 TypeLock         : _EX_PUSH_LOCK
   +0x0c0 Key              : 0x636f7250
   +0x0c8 CallbackList     : _LIST_ENTRY [ 0xffffc78f`27685580 - 0xffffc78f`1d7062f0 ]
```
```
1: kd> dqs PsThreadType
fffff804`0defc440  ffff830f`7faff560

1: kd> dt_object_type ffff830f`7faff560
nt!_OBJECT_TYPE
   +0x000 TypeList         : _LIST_ENTRY [ 0xffff830f`7faff560 - 0xffff830f`7faff560 ]
   +0x010 Name             : _UNICODE_STRING "Thread"
   +0x020 DefaultObject    : (null) 
   +0x028 Index            : 0x8 ''
   +0x02c TotalNumberOfObjects : 0x6df1
   +0x030 TotalNumberOfHandles : 0xec3
   +0x034 HighWaterNumberOfObjects : 0x6df1
   +0x038 HighWaterNumberOfHandles : 0x107a
   +0x040 TypeInfo         : _OBJECT_TYPE_INITIALIZER
   +0x0b8 TypeLock         : _EX_PUSH_LOCK
   +0x0c0 Key              : 0x65726854
   +0x0c8 CallbackList     : _LIST_ENTRY [ 0xffffda80`6d796870 - 0xffffda80`71c3e510 ]
```

**CallbackList**는 <code>_LIST_ENTRY</code> 구조이며 <code>+0x28</code> 위치에 **PreOperation**, <code>+0x30</code> 위치에 **PostOperation** 콜백 함수가 존재합니다.<br>해당 위치는 패치가드에 트리거 되지 않아 일부 해킹툴에서 콜백을 조작하여 핸들 보호를 우회하기도 합니다.
```c
typedef struct _OB_OPERATION_REGISTRATION {
  POBJECT_TYPE                *ObjectType;
  OB_OPERATION                Operations;
  POB_PRE_OPERATION_CALLBACK  PreOperation;
  POB_POST_OPERATION_CALLBACK PostOperation;
} OB_OPERATION_REGISTRATION, *POB_OPERATION_REGISTRATION;
```
```
1: kd> dx -id 0,0,ffff830f9f887080 -r1 ((ntkrnlmp!_LIST_ENTRY *)0xffff830f7faff628)
((ntkrnlmp!_LIST_ENTRY *)0xffff830f7faff628)                 : 0xffff830f7faff628 [Type: _LIST_ENTRY *]
    [+0x000] Flink            : 0xffffda806d796870 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0xffffda8071c3e510 [Type: _LIST_ENTRY *]
```
```
1: kd> dqs 0xffffda8071c3e510+28
ffffda80`71c3e538  fffff806`17f850d0 driver+0x50d0
ffffda80`71c3e540  fffff806`17f81ff4 driver+0x1ff4
```

## References
- [ObRegisterCallbacks 함수(wdm.h)](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/ddi/wdm/nf-wdm-obregistercallbacks)
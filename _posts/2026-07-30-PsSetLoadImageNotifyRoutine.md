---
title: PsSetLoadImageNotifyRoutine
date: 2026-07-30 16:39:00 +0900
categories: [Windows Internals]
tags: [Windows Internals]
---

## PsSetLoadImageNotifyRoutine
**PsSetLoadImageNotifyRoutine**은 이미지가 로드될 때 호출되는 콜백 루틴을 등록합니다.<br>등록된 콜백은 <code>EntryPoint</code> 코드보다 우선적으로 실행됩니다.
```c
NTSTATUS PsSetLoadImageNotifyRoutine(
  [in] PLOAD_IMAGE_NOTIFY_ROUTINE NotifyRoutine
);
```
```c
PLOAD_IMAGE_NOTIFY_ROUTINE PloadImageNotifyRoutine;

VOID PloadImageNotifyRoutine(
  [in, optional] PUNICODE_STRING FullImageName,
  [in]           HANDLE ProcessId,
  [in]           PIMAGE_INFO ImageInfo
)
{...}
```

**PsSetLoadImageNotifyRoutineEx** 함수 내부에서는 **PspLoadImageNotifyRoutine** 라는 내부 콜백 리스트를 참조합니다.
```
nt!PsSetLoadImageNotifyRoutineEx:
...
fffff804`575997d1 488d0d082c5500  lea     rcx,[nt!PspLoadImageNotifyRoutine (fffff804`57aec3e0)]
fffff804`575997d8 4533c0          xor     r8d,r8d
fffff804`575997db 488d0cd9        lea     rcx,[rcx+rbx*8]
fffff804`575997df 488bd7          mov     rdx,rdi
fffff804`575997e2 e81558c1ff      call    nt!ExCompareExchangeCallBack (fffff804`571aeffc)
```

저장된 주소는 <code>^0x0f</code> 연산이 필요합니다.<br>해당 테이블은 패치가드에 트리거 되지 않아 일부 해킹툴에서는 콜백을 조작하여 감지를 우회합니다.
```
2: kd> dqs PspLoadImageNotifyRoutine
fffff804`57aec3e0  ffffb502`322971ef
fffff804`57aec3e8  ffffb502`1ee8217f
fffff804`57aec3f0  ffffb502`2320d7ef
fffff804`57aec3f8  ffffb502`33d917af

2: kd> dqs ffffb502`33d917af ^ f
ffffb502`33d917a0  00000000`00000020
ffffb502`33d917a8  fffff804`81044764 driver.sys+4767
```

**PsSetLoadImageNotifyRoutine**를 통해 등록된 콜백의 개수는 **PspLoadImageNotifyRoutineCount**를 통해서도 확인이 가능합니다.
```
nt!PsSetLoadImageNotifyRoutineEx+0x5f:
fffff804`0d9997ef f0ff05e6515900  lock inc dword ptr [nt!PspLoadImageNotifyRoutineCount (fffff804`0df2e9dc)]
fffff804`0d9997f6 8b05a44e5900    mov     eax,dword ptr [nt!PspNotifyEnableMask (fffff804`0df2e6a0)]
fffff804`0d9997fc a801            test    al,1
fffff804`0d9997fe 7509            jne     nt!PsSetLoadImageNotifyRoutineEx+0x79 (fffff804`0d999809)  Branch
```
```
1: kd> dqs PspLoadImageNotifyRoutineCount
fffff804`0df2e9dc  00000000`00000003
```

## References
- [PsSetLoadImageNotifyRoutine function (ntddk.h)](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-pssetloadimagenotifyroutine)
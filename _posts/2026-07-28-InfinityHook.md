---
title: Infinity Hook
date: 2026-07-28 18:39:00 +0900
categories: [Windows Internals]
tags: [Windows Internals]
---

## ETW (Event Tracing for Windows)
윈도우 운영체제에는 커널 또는 애플리케이션에서 발생한 이벤트를 추적할 수 있는 **ETW (Event Tracing for Windows)** 라는 기능이 존재합니다.
**Infinity Hook**은 ETW 시스템 내에서 호출되는 함수를 후킹하여 시스템 콜을 가로채는 오픈소스 도구입니다.
![](/assets/posts/2026-07-28-InfinityHook/hiding_file.gif)

## EtwpDebuggerData
후킹에 있어 가장 먼저 찾아야 하는 데이터입니다. **ntoskrnl** 이미지 영역 내 <code>2C 08 04 38 0C</code> 시그니처 검색을 통해 찾을 수 있습니다.
검색된 주소 **-0x02** 위치가 **EtwpDebuggerData** 에 해당됩니다.
```
2: kd> s nt L? 1046000 2c 08 04 38 0c
fffff807`6e010e3a  2c 08 04 38 0c f8 d8 00-70 10 98 0c 04 60 40 ac  ,..8....p....`@.

2: kd> ln fffff807`6e010e3a-2
Browse module
Set bu breakpoint

(fffff807`6e010e38)   nt!EtwpDebuggerData   |  (fffff807`6e010e60)   nt!ExpSystemIsInCmosMode
Exact matches:
```

### EtwpDebuggerDataSilo
**EtwpDebuggerData + 0x10** 위치에 함수 테이블 <code>EtwpDebuggerDataSilo</code>이 존재합니다.
```
2: kd> dq EtwpDebuggerData + 0x10
fffff807`6e010e48  ffffd98c`2790ac40 // EtwpDebuggerDataSilo[]
```

#### CkclWmiLoggerContext
**EtwpDebuggerDataSilo + 0x10** 위치에는 <code>nt!_WMI_LOGGER_CONTEXT</code> 구조를 가진 **CkclWmiLoggerContext**이 존재합니다.
```
2: kd> dq ffffd98c`2790ac40+0x10
ffffd98c`2790ac50  ffffd98c`2790e940 // CkclWmiLoggerContext
```

이 안에는 <code>GetCpuClock</code>이라는 필드가 존재합니다. 후킹에 사용되는 실질적인 데이터입니다.
```
2: kd> dt_WMI_LOGGER_CONTEXT ffffd98c`2790e940
nt!_WMI_LOGGER_CONTEXT
   +0x000 LoggerId         : 2
   +0x004 BufferSize       : 0x1000
   +0x008 MaximumEventSize : 0xfb8
   +0x00c LoggerMode       : 0x2800480
   +0x010 AcceptNewEvents  : 0n0
   +0x014 EventMarker      : [2] 0xc0130000
   +0x01c ErrorMarker      : 0xc00d0000
   +0x020 SizeMask         : 0xffff
   +0x028 GetCpuClock      : 3
```

## InfinityHook
[InfinityHook](https://github.com/everdox/InfinityHook)은 **Windows 7** 부터 **Windows 10 (Build 18363)** 까지 지원합니다.
해당 버전 커널에서는 **GetCpuClock**가 함수 포인터로 사용됩니다. 이를 조작하여 후킹 코드를 실행합니다.
```c
    if (ctx->BuildNumber <= 18363) 
    {
        // win 7 -> win10 1909
        LOG_INFO("GetCpuClock Is 0x%p", *ctx->GetCpuClock);
        *ctx->GetCpuClock = (PVOID)FakeGetCpuClock;                // replace function.
        LOG_INFO("Update GetCpuClock Is 0x%p", *ctx->GetCpuClock);
    }
```

## InfinityHookPro (Windows 10 Version 1909, Build 18363+)
[InfinityHookPro](https://github.com/i1tao/InfinityHookProLib)는 **Windows 10 (Build 19041)** 부터 **최신 버전 윈도우 (Windows 11)** 까지 지원합니다.

해당 버전 커널에서는 더 이상 **GetCpuClock** 가 함수 포인터로 사용되지 않고, 인덱스 값으로 사용됩니다.
```
1: kd> dt nt!_WMI_LOGGER_CONTEXT ffff830f`7fda5240
   +0x000 LoggerId         : 2
   +0x004 BufferSize       : 0x1000
   +0x008 MaximumEventSize : 0xfb8
   +0x00c LoggerMode       : 0x2800480
   +0x010 AcceptNewEvents  : 0n0
   +0x014 EventMarker      : [2] 0xc0130000
   +0x01c ErrorMarker      : 0xc00d0000
   +0x020 SizeMask         : 0xffff
   +0x028 GetCpuClock      : 2
```

인덱스 값에 따라 **nt!EtwpGetLoggerTimeStamp** 내부에서 호출되는 함수가 달라집니다.
```
1: kd> u EtwpGetLoggerTimeStamp
nt!EtwpGetLoggerTimeStamp:
fffff804`0d42c448 4883ec28        sub     rsp,28h
fffff804`0d42c44c 488b4128        mov     rax,qword ptr [rcx+28h]
fffff804`0d42c450 4883f803        cmp     rax,3
fffff804`0d42c454 0f87f7ca1f00    ja      nt!EtwpGetLoggerTimeStamp+0x1fcb09 (fffff804`0d628f51)
fffff804`0d42c45a 85c0            test    eax,eax
fffff804`0d42c45c 7416            je      nt!EtwpGetLoggerTimeStamp+0x2c (fffff804`0d42c474)
fffff804`0d42c45e 83e801          sub     eax,1
fffff804`0d42c461 0f85b1ca1f00    jne     nt!EtwpGetLoggerTimeStamp+0x1fcad0 (fffff804`0d628f18)
```

### [0x02] nt!HalpTimerQueryHostPerformanceCounter
**InfinityHookPro** 에서는 **GetCpuClock**  값을 **0x02** 로 변경하고 있습니다.
```c
        *ctx->GetCpuClock = (PVOID)2;
        LOG_INFO("Update GetCpuClock Is 0x%p", *ctx->GetCpuClock);
```

인덱스가 <code>0x02</code>인 경우 **nt!HalpTimerQueryHostPerformanceCounter** 가 호출됩니다.
```
1: kd> u EtwpGetLoggerTimeStamp
nt!EtwpGetLoggerTimeStamp:
fffff804`0d42c448 4883ec28        sub     rsp,28h
fffff804`0d42c44c 488b4128        mov     rax,qword ptr [rcx+28h]
fffff804`0d42c450 4883f803        cmp     rax,3
fffff804`0d42c454 0f87f7ca1f00    ja      nt!EtwpGetLoggerTimeStamp+0x1fcb09 (fffff804`0d628f51)
fffff804`0d42c45a 85c0            test    eax,eax
fffff804`0d42c45c 7416            je      nt!EtwpGetLoggerTimeStamp+0x2c (fffff804`0d42c474)
fffff804`0d42c45e 83e801          sub     eax,1
fffff804`0d42c461 0f85b1ca1f00    jne     nt!EtwpGetLoggerTimeStamp+0x1fcad0 (fffff804`0d628f18)

1: kd> u nt!EtwpGetLoggerTimeStamp+0x1fcad0
nt!EtwpGetLoggerTimeStamp+0x1fcad0:
fffff804`0d628f18 83e801          sub     eax,1
fffff804`0d628f1b 7413            je      nt!EtwpGetLoggerTimeStamp+0x1fcae8 (fffff804`0d628f30)
fffff804`0d628f1d 83f801          cmp     eax,1
fffff804`0d628f20 752f            jne     nt!EtwpGetLoggerTimeStamp+0x1fcb09 (fffff804`0d628f51)
fffff804`0d628f22 0f31            rdtsc
fffff804`0d628f24 48c1e220        shl     rdx,20h
fffff804`0d628f28 480bc2          or      rax,rdx
fffff804`0d628f2b e93e35e0ff      jmp     nt!EtwpGetLoggerTimeStamp+0x26 (fffff804`0d42c46e)
fffff804`0d628f30 488364243000    and     qword ptr [rsp+30h],0
fffff804`0d628f36 488d4c2430      lea     rcx,[rsp+30h]
fffff804`0d628f3b 488b059e7a7d00  mov     rax,qword ptr [nt!HalPrivateDispatchTable+0x450 (fffff804`0de009e0)]
fffff804`0d628f42 e839edfdff      call    nt!guard_dispatch_icall (fffff804`0d607c80)
fffff804`0d628f47 488b442430      mov     rax,qword ptr [rsp+30h]
fffff804`0d628f4c e91d35e0ff      jmp     nt!EtwpGetLoggerTimeStamp+0x26 (fffff804`0d42c46e)
fffff804`0d628f51 b93d000000      mov     ecx,3Dh
fffff804`0d628f56 cd29            int     29h

1: kd> dqs fffff804`0de009e0
fffff804`0de009e0  fffff804`0d6b6990 nt!HalpTimerQueryHostPerformanceCounter
```

**nt!HalpTimerQueryHostPerformanceCounter** 내부에서는 함수 포인터를 통해 **nt!HvlGetQpcBias** 를 호출합니다.
```
1: kd> uf HalpTimerQueryHostPerformanceCounter
nt!HalpTimerQueryHostPerformanceCounter:
fffff804`0d6b6990 48895c2408      mov     qword ptr [rsp+8],rbx
fffff804`0d6b6995 57              push    rdi
fffff804`0d6b6996 4883ec20        sub     rsp,20h
fffff804`0d6b699a 488b05af567900  mov     rax,qword ptr [nt!HalpPerformanceCounter (fffff804`0de4c050)]
fffff804`0d6b69a1 488bf9          mov     rdi,rcx
fffff804`0d6b69a4 4885c0          test    rax,rax
fffff804`0d6b69a7 743f            je      nt!HalpTimerQueryHostPerformanceCounter+0x58 (fffff804`0d6b69e8)  Branch

nt!HalpTimerQueryHostPerformanceCounter+0x19:
fffff804`0d6b69a9 83b8e400000008  cmp     dword ptr [rax+0E4h],8
fffff804`0d6b69b0 7536            jne     nt!HalpTimerQueryHostPerformanceCounter+0x58 (fffff804`0d6b69e8)  Branch

nt!HalpTimerQueryHostPerformanceCounter+0x22:
fffff804`0d6b69b2 48833dde39790000 cmp     qword ptr [nt!HalpEnlightenment+0x178 (fffff804`0de4a398)],0
fffff804`0d6b69ba 742c            je      nt!HalpTimerQueryHostPerformanceCounter+0x58 (fffff804`0d6b69e8)  Branch
```

**InfinityHookPro**는 해당 포인터 위치를 시그니처 검색을 통해 찾고 후킹 코드로 변조합니다.
```c
    //
    // Find HvlGetQpcBias.
    //
    ULONG64 RefHvlGetQpcBiasAddress = FindPatternImage(
        ctx->NtoskrnlBase,
        "\x48\x8b\x05\x00\x00\x00\x00\x48\x85\xc0\x74\x00\x48\x83\x3d\x00\x00\x00\x00\x00\x74", // before Win10 22H2 & before Win11 22621
        "xxx????xxxx?xxx?????x",
        ".text");

    if (!RefHvlGetQpcBiasAddress)
    {
        //All of these feature codes are present.
        RefHvlGetQpcBiasAddress = FindPatternImage(
            ctx->NtoskrnlBase,
            "\x48\x8b\x05\x00\x00\x00\x00\xe8\x00\x00\x00\x00\x48\x03\xd8\x48\x89\x1f",
            "xxx????x????xxxxxx",
            ".text");
        if (!RefHvlGetQpcBiasAddress)
        {
            LOG_ERROR("Find HvlGetQpcBias Failed!");
            return FALSE;
        }
    }
```

덤프를 통해 확인해보면 함수 포인터가 변경되어 있습니다.
```
1: kd> dqs nt!HalpEnlightenment+0x178
fffff804`0de4a398  fffff806`18307d40 0Ab5dgbQCYlbeUuu+0x7d40
```

후킹 함수에서는 스택을 읽고, **SSDT**에 포함된 시스템 콜인 경우 다음 동작을 수행합니다.
```c
ULONG64 FakeGetCpuClock()
{
    // 
    if (ExGetPreviousMode() == KernelMode)
    {
        __rdtsc();
    }

    PKTHREAD pCurrentThread = (PKTHREAD)__readgsqword(0x188);

    UINT32 nCallIndex = 0;
    if (g_IHookProContext.BuildNumber <= 7601)
    {
        nCallIndex = *(unsigned int*)((ULONG64)pCurrentThread + 0x1f8);
    }
    else
    {
        nCallIndex = *(unsigned int*)((ULONG64)pCurrentThread + 0x80);
    }

    void** pStackMax = (void**)__readgsqword(0x1a8);
    void** pStackFrame = (void**)_AddressOfReturnAddress();

    for (void** pStackCurrent = pStackMax; pStackCurrent > pStackFrame; --pStackCurrent)
    {
#define INFINITYHOOK_MAGIC_501802 ((unsigned long)0x501802)
#define INFINITYHOOK_MAGIC_601802 ((unsigned long)0x601802)
#define INFINITYHOOK_MAGIC_F33 ((unsigned short)0xF33)

        unsigned long* pValue1 = (unsigned long*)pStackCurrent;
        if ((*pValue1 != INFINITYHOOK_MAGIC_501802) &&
            (*pValue1 != INFINITYHOOK_MAGIC_601802))
        {
            continue;
        }

        --pStackCurrent;

        unsigned short* pValue2 = (unsigned short*)pStackCurrent;
        if (*pValue2 != INFINITYHOOK_MAGIC_F33)
        {
            continue;
        }

        for (; pStackCurrent < pStackMax; ++pStackCurrent)
        {
            // 检查是否在ssdt表内
            PULONG64 pllValue = (PULONG64)pStackCurrent;
            if (!(PAGE_ALIGN(*pllValue) >= g_IHookProContext.SystemCallTable &&
                PAGE_ALIGN(*pllValue) < (void*)((ULONG64)g_IHookProContext.SystemCallTable + (PAGE_SIZE * 2))))

            {
                continue;
            }

            // 现在已经确定是ssdt函数调用了
            // 这里是找到KiSystemServiceExit
            void** pSystemCallFunction = &pStackCurrent[9];

            InfinityCallback(nCallIndex, pSystemCallFunction);

            break;
        }
        break;
    }


    return __rdtsc();
}
```

## References
- [GitHub - InfinityHook](https://github.com/everdox/InfinityHook)
- [GitHub - InfinityHookProLib](https://github.com/i1tao/InfinityHookProLib)
- [Windows 10 19041版本的Infinity hook 原理](https://www.anquanke.com/post/id/206288)
- [About Event Tracing](https://learn.microsoft.com/en-us/windows/win32/etw/about-event-tracing)
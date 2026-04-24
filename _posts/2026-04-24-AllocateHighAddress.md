---
title: Accessing Kernel Address Space from User Mode
date: 2026-04-24 18:30:00 +0900
categories: [Tech]
tags: [Tech]
---

작성 중.

## Overview
여느때와 같이 해킹툴을 분석하다가 재밌는 기법을 발견하였습니다.<br>
다른 해킹툴과 크게 다르지 않게 d3d를 후킹하여 UI를 그려주는 해킹툴인데요.
```
0:000> u dxgi.dll+B65A
dxgi!CDXGISwapChain::PresentImplCore+0x9a:
00007ffd`4495b65a 498b8dd0000000  mov     rcx,qword ptr [r13+0D0h]
00007ffd`4495b661 488b01          mov     rax,qword ptr [rcx]
00007ffd`4495b664 488d55f8        lea     rdx,[rbp-8]
00007ffd`4495b668 488b4068        mov     rax,qword ptr [rax+68h]
00007ffd`4495b66c e89f590c00      call    dxgi!guard_dispatch_icall$thunk$10345483385596137414 (00007ffd`44a21010)

0:000> u 00007ffd`44a21010
dxgi!guard_dispatch_icall$thunk$10345483385596137414:
00007ff9`934bf010 e9ab80ca05      jmp     00007ff9`991670c0

0:000> u 00007ff9`991670c0
00007ff9`991670c0 48ffe0          jmp     rax
```

아래는 dxgi 내부에서 호출하는 가상 함수 테이블입니다. 신기한건 후킹을 통해 점프되는 메모리 위치가 커널 영역에 해당합니다. <br><code>dxgi!CD3D12Device::GetContentProtection</code> 함수가 조작되어 있습니다.
```
0:000> dqs 000001FEF035B2D0
000001fe`f035b2d0  ffff9600`003cf7a0
000001fe`f035b2d8  00007ffd`44a26b88 dxgi!TComObject<CD3D12Device>::`vftable'
000001fe`f035b2e0  00007ffd`44a26b50 dxgi!TComObject<CD3D12Device>::`vftable'
```
```
12: kd> dqs ffff9600003cf7a0
ffff9600`003cf7a0  00007ffb`368bd8a0 dxgi!TComObject<CD3D12Device>::QueryInterface
ffff9600`003cf7a8  00007ffb`368d4ca0 dxgi!TComObject<CD3D12Device>::AddRef
ffff9600`003cf7b0  00007ffb`368b3810 dxgi!TComObject<CD3D12Device>::Release
ffff9600`003cf7b8  00007ffb`368f70c0 dxgi!CD3D12Device::Present
ffff9600`003cf7c0  00007ffb`368b48e0 dxgi!WNFHelper::DoNothingWnfCallbackFn
ffff9600`003cf7c8  00007ffb`368b3f00 dxgi!CD3D12Resource::CreateSubresourceSurface
ffff9600`003cf7d0  00007ffb`368f6390 dxgi!CD3D12Device::CreateSurfaceInternal
ffff9600`003cf7d8  00007ffb`36886410 dxgi!CD3D12Device::Blt
ffff9600`003cf7e0  00007ffb`368b3f00 dxgi!CD3D12Resource::CreateSubresourceSurface
ffff9600`003cf7e8  00007ffb`368f6cc0 dxgi!CD3D12Device::GetGammaCaps
ffff9600`003cf7f0  00007ffb`368f7020 dxgi!CD3D12Device::OpenSharedResource
ffff9600`003cf7f8  00007ffb`368a9450 dxgi!CD3D12Device::IsValidScanoutFormat
ffff9600`003cf800  00007ffb`368b1170 dxgi!CD3D12Device::DeviceCompletedHWNDFullscreenTransition
> ffff9600`003cf808  ffff9600`0005e760 // hooked dxgi!CD3D12Device::GetContentProtection
ffff9600`003cf810  00007ffb`368f6be0 dxgi!CD3D12Device::Flush
ffff9600`003cf818  00007ffb`368a27f0 dxgi!CD3D12Device::AcquireResource
```

일반적으로 알려진 user/kernel space는 다음과 같습니다.<br>
후킹 함수 <code>ffff9600`0005e760</code>를 디버깅해 보면 신기하게도 코드가 보이지는 않지만 실행은 되었습니다.
| 구분 | 주소 범위 |
|------|-----------|
| User Space | 0x00000000`00000000 ~ 0x00007FFF`FFFFFFFF |
| Kernel Space | 0xFFFF8000`00000000 ~ 0xFFFFFFFF`FFFFFFFF |

## user/supervisor bit
관련하여 조사해 보니 user/supervisor bit를 통해 특정 페이지에 대한 접근 권한을 설정할 수 있다는 것을 확인하였습니다.
```
// https://www.unknowncheats.me/forum/anti-cheat-bypass/699139-allocate-memory-address-space.html
cool that youre trying to help but youre wrong. the address range for kernel and usermode is an arbitrary limit. the real way the cpu checks if you can access memory from usermode is the (user) supervisor bit in the pte of the page and not the actual address itself.

to op: you can allocate kernel memory (remap it to a hugher pml4e if you want to) and then set the user supervisor bit to 1 in the pml4e, pdpte, pde, pte and voila you will have user mode accessible kernel memory.
```

수집된 덤프 내에서도 확인해 보면 PXE/PPE/PDE/PTE 내 모든 user/supervisor bit가 1로 설정되어 있습니다.
```
12: kd> !pte ffff96000005e760
                                           VA ffff96000005e760
PXE at FFFF86C361B0D960    PPE at FFFF86C361B2C000    PDE at FFFF86C365800000    PTE at FFFF86CB000002F0
contains 0000000E4251F067  contains 0000000E64F20067  contains 0000000D2A026067  contains 0A00000E70D87967
pfn e4251f    ---DA--UWEV  pfn e64f20    ---DA--UWEV  pfn d2a026    ---DA--UWEV  pfn e70d87    -G-DA--UWEV
```

그래서 ExAllocatePool 같은 함수를 통해 할당하고 해당 비트만 수정하는줄 알았으나, 시도해 보니 커널 주소의 코드가 실행되지 않았습니다.<br>
테스트는 대충 커널 내 다음과 같이 코드를 할당하고, 유저 모드에서 실행하였습니다.
```
ffff9600`00000000:
mov rax, 1
ret
```
```
main:
mov rax, ffff9600`00000000
jmp rax
```

## CR3
관련하여 추가로 조사해 보면 
```
12: kd> !vtop d0d518000 ffff96000005e760
Amd64VtoP: Virt ffff96000005e760, pagedir 0000000d0d518000
Amd64VtoP: PML4E 0000000d0d518960
Amd64VtoP: PDPE 0000000e4251f000
Amd64VtoP: PDE 0000000e64f20000
Amd64VtoP: PTE 0000000d2a0262f0
Amd64VtoP: Mapped phys 0000000e70d87760
Virtual address ffff96000005e760 translates to physical address e70d87760.

```

## notskrnl
VirtualAlloc/ReadProcessMemory/VirtualQuery 등 프로세스 내 가상 메모리와 관련된 모든 함수는 ntoskrnl 내에서 다음과 같이 유저 영역에 해당하는지 체크합니다. 커널에 해당하는 경우 동작하지 않으며 실패를 반환합니다.

해킹툴은 이를 악용해 안티치트를 우회합니다. <br>
VirtualQuery 등을 통해 메모리 정보를 가져오는 탐지 코드에서는 할당된 메모리 영역 자체가 보이지 않을것 입니다.
```cpp
__int64 __usercall MiReadWriteVirtualMemory@<rax>(ULONG_PTR BugCheckParameter1@<rcx>, unsigned __int64 a2@<rdx>, unsigned __int64 a3@<r8>, __int64 a4@<r9>, __int64 a5, int a6)
{
  ...
    if ( v10 < a3 || v9 > 0x7FFFFFFEFFFFi64 || v10 > 0x7FFFFFFEFFFFi64 )
      return 0xC0000005i64;
  ...
}
__int64 __fastcall MmQueryVirtualMemory(__int64 a1, unsigned __int64 a2, __int64 a3, unsigned __int64 a4, unsigned __int64 a5, unsigned __int64 *a6)
{
  ...
  if ( v12 > 0x7FFFFFFEFFFFi64 )
    return 0xC000000Di64;
  ...
}
```

## References
- [https://blog.can.ac/2018/05/02/making-the-perfect-injector-abusing-windows-address-sanitization-and-cow/](https://blog.can.ac/2018/05/02/making-the-perfect-injector-abusing-windows-address-sanitization-and-cow/)
- [https://github.com/can1357/ThePerfectInjector](https://github.com/can1357/ThePerfectInjector)
- [https://www.unknowncheats.me/forum/anti-cheat-bypass/699139-allocate-memory-address-space.html](https://www.unknowncheats.me/forum/anti-cheat-bypass/699139-allocate-memory-address-space.html)
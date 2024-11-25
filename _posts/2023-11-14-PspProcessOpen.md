---
title: 프로세스 보호 (PspProcessOpen)
date: 2023-11-14 00:00:00 +/-TTTT
categories: [Analysis Report]
tags: [Windows Kernel]
---

## STATUS_UNSUCCESSFUL (0xC0000001)
OpenProcess를 호출하면 0xC0000001를 반환하며 실패하는 해킹툴이 발견되었습니다. 

다음과 같이 NtOpenProcess 시스콜 이후에 실패하고 있으며, 이를 봤을 때 커널 레벨에서 조작된 것을 추측할 수 있습니다.
![](/assets/posts/2023-11-14-PspProcessOpen/5.png)

해당 값은 [MS-ERREF](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55) 문서에 나와있는데 요청이 실패한 경우 반환되는 값입니다. 특정한 경우가 아닌, 범용적으로 사용되는 코드라 해당 내용만 가지고는 어떤 부분이 조작되었는지 짐작하기 어렵습니다.

| Return value/code   |	Description |
|---------------------|----------------|
| 0xC0000001<br><br>STATUS_UNSUCCESSFUL | {Operation Failed} The requested operation was unsuccessful. |

ntkrnlmp 내에 존재하는 OpenProcess와 관련된 반환 부분을 전부 확인하였지만 조작으로 의심되는 부분이 확인되지 않았습니다.
![](/assets/posts/2023-11-14-PspProcessOpen/6.png)

## Kernel Debugging
덤프만으로는 변조된 위치를 찾기 어렵기 때문에, 가상 환경에서 커널 디버깅을 통해 분석을 진행하였습니다. 다음과 같이 보호가 걸린 프로세스를 대상으로 OpenProcess를 시도하고 동적 분석을 진행합니다.
```cpp
#include <Windows.h>
#include <stdio.h>

int main()
{
    MessageBoxA(0, "Wait", "debug", 0);

    // dwm.exe
    HANDLE dwm = 0;
    do
    {
        dwm = OpenProcess(PROCESS_ALL_ACCESS, false, 9360);

        printf("handle : %08X\n", dwm);
    } while (!dwm);

    system("pause");
}
```

분석은 Windbg Preview를 통해 진행하였으며, 커널 디버그 모드에서는 다음과 같이 대상 프로세스를 찾고 bp를 설정합니다.
```
2: kd> !process 0 0 test.exe
PROCESS ffffce05ef58a080
    SessionId: 1  Cid: 2aa8    Peb: abecbd3000  ParentCid: 1578
    DirBase: 12809a000  ObjectTable: ffffe086d45b5b40  HandleCount: 138.
    Image: test.exe

2: kd> .process /i /r /p ffffce05ef58a080
You need to continue execution (press 'g' <enter>) for the context
to be switched. When the debugger breaks in again, you will be in
the new process context.
2: kd> g
Break instruction exception - code 80000003 (first chance)
nt!DbgBreakPointWithStatus:
fffff805`25407150 cc              int     3
14: kd> .reload /user
Loading User Symbols
...............................

14: kd> u test!main
test!main [C:\Users\JH\Downloads\ConsoleApplication5\ConsoleApplication5.cpp @ 5]:
00007ff6`fd631070 4883ec28        sub     rsp,28h
00007ff6`fd631074 4533c9          xor     r9d,r9d
00007ff6`fd631077 4c8d0522e40100  lea     r8,[test!`string' (00007ff6`fd64f4a0)]
00007ff6`fd63107e 488d1523e40100  lea     rdx,[test!`string' (00007ff6`fd64f4a8)]
00007ff6`fd631085 33c9            xor     ecx,ecx
00007ff6`fd631087 ff15f3610100    call    qword ptr [test!_imp_MessageBoxA (00007ff6`fd647280)]
00007ff6`fd63108d 33d2            xor     edx,edx
00007ff6`fd63108f 41b890240000    mov     r8d,2490h
00007ff6`fd631095 b9ffff1f00      mov     ecx,1FFFFFh
00007ff6`fd63109a ff15605f0100    call    qword ptr [test!_imp_OpenProcess (00007ff6`fd647000)]
00007ff6`fd6310a0 488bd0          mov     rdx,rax

0: kd> bp /p @$proc 00007ff6`fd63109a
```

이후에 메세지 박스를 확인하여 OpenProcess가 실행되면 중단점이 실행됩니다.
![](/assets/posts/2023-11-14-PspProcessOpen/9.png)

동적 디버깅을 통해 함수 내부로 진입해 다음과 같은 코드 위치에서 0xC0000001가 반환되는 것을 확인하였습니다.
```
14: kd> u rip-5
nt!ObpIncrementHandleCountEx+0x343:
fffff805`2564be23 e828c0dbff      call    nt!guard_dispatch_icall (fffff805`25407e50)
fffff805`2564be28 8be8            mov     ebp,eax
fffff805`2564be2a 4584ed          test    r13b,r13b
fffff805`2564be2d 0f8515010000    jne     nt!ObpIncrementHandleCountEx+0x468 (fffff805`2564bf48)
fffff805`2564be33 65488b042588010000 mov   rax,qword ptr gs:[188h]
fffff805`2564be3c 66ff88e4010000  dec     word ptr [rax+1E4h]
fffff805`2564be43 33d2            xor     edx,edx
fffff805`2564be45 498bce          mov     rcx,r14

14: kd> r
rax=00000000c0000001 rbx=ffffce05e64b0400 rcx=0000000000000001
rdx=0000000000000001 rsi=0000000000000001 rdi=ffffce05eb26f050
rip=fffff8052564be28 rsp=fffff40851bff160 rbp=ffffce05ec1bd080
 r8=ffffce05ec1bd080  r9=ffffce05eb26f080 r10=0000000000002a68
r11=ffffce05e5f030b0 r12=0000000000000001 r13=0000000000000000
r14=ffffce05eb26f060 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00040246
nt!ObpIncrementHandleCountEx+0x348:
fffff805`2564be28 8be8            mov     ebp,eax
```

해당 위치를 다시 확인해보면 다음과 같이 심볼 정보가 존재하지 않은 특정 함수로 진입합니다. 함수 내부에서는 0xC0000001를 반환하고 있으며, 해킹툴에 의하여 호출되는 함수로 추측할 수 있습니다.
```
ffffce05`e5f19000 49bbb030f0e505ceffff mov     r11, 0FFFFCE05E5F030B0h
ffffce05`e5f1900a 4d3bc1               cmp     r8, r9
ffffce05`e5f1900d 741a                 je      FFFFCE05E5F19029
ffffce05`e5f1900f 85c9                 test    ecx, ecx
ffffce05`e5f19011 7416                 je      FFFFCE05E5F19029
ffffce05`e5f19013 498b03               mov     rax, qword ptr [r11]
ffffce05`e5f19016 4c8b5008             mov     r10, qword ptr [rax+8]
ffffce05`e5f1901a 4d399140040000       cmp     qword ptr [r9+440h], r10
ffffce05`e5f19021 7506                 jne     FFFFCE05E5F19029
ffffce05`e5f19023 b8010000c0           mov     eax, 0C0000001h
ffffce05`e5f19028 c3                   ret     
```

해당 함수의 호출 스택은 다음과 같습니다.

| Ring | Call Stacks                    |
|:-:|-----------------------------------|
| 0 | 0xffffce05e5f19000                |
| 0 | nt!ObpIncrementHandleCountEx      |
| 0 | nt!ObpCreateHandle                |
| 0 | nt!ObOpenObjectByPointer          |
| 0 | nt!PsOpenProcess                  |
| 0 | nt!NtOpenProcess                  |
| 0 | nt!KiSystemServiceCopyEnd         |
| 3 | ntdll!NtOpenProcess               |
| 3 | KERNELBASE!OpenProcess            |
| 3 | test!main                         |

ObpIncrementHandleCountEx에서는 다음과 같이 함수 포인터를 구해오며 _guard_dispatch_icall를 통해 rax에 존재하는 해킹툴 함수를 호출합니다.
```
fffff805`2564bdfa 8b4c245c           mov     ecx, dword ptr [rsp+5Ch]
fffff805`2564bdfe 4c8bc5             mov     r8, rbp
*fffff805`2564be01 488b4378           mov     rax, qword ptr [rbx+78h]
fffff805`2564be05 4c8b4c2470         mov     r9, qword ptr [rsp+70h]
fffff805`2564be0a 0fb6942420010000   movzx   edx, byte ptr [rsp+120h]
fffff805`2564be12 894c2428           mov     dword ptr [rsp+28h], ecx
fffff805`2564be16 488b4c2468         mov     rcx, qword ptr [rsp+68h]
fffff805`2564be1b 48894c2420         mov     qword ptr [rsp+20h], rcx
fffff805`2564be20 418bcc             mov     ecx, r12d
fffff805`2564be23 e828c0dbff         call    ntkrnlmp!_guard_dispatch_icall (fffff80525407e50)
```

참조되는 함수 포인터의 메모리를 확인해보면 다음과 같습니다. PspProcessClose, PspProcessDelete 함수와 함께 해킹툴 코드가 위치하고 있으며, 이를 통해 특정 함수 테이블이 후킹되었음을 추측할 수 있습니다.
```
4: kd> dqs rbx+78
ffffce05`e64b0478  ffffce05`e5f19000
ffffce05`e64b0480  fffff805`256fb510 nt!PspProcessClose
ffffce05`e64b0488  fffff805`255f4c50 nt!PspProcessDelete
```

후킹된 함수는 nt!PspProcessOpen이며, 해킹툴이 적용되지 않은 클린 환경에서 동일한 테스트를 통해 확인하였습니다.
![](/assets/posts/2023-11-14-PspProcessOpen/10.png)

## Kernel Memory Dump
디버깅을 통해 확인된 정보를 실제 해킹툴이 적용된 덤프 파일에서 다시 확인해봤습니다. 후킹된 nt!PspProcessOpen 함수는 PsProcessType를 통해 다음과 같이 확인할 수 있습니다.
```
3: kd> dqs PsProcessType
fffff800`6dafc410  ffff8488`aa562b00

3: kd> dt_object_type ffff8488`aa562b00
ntdll!_OBJECT_TYPE
   +0x000 TypeList         : _LIST_ENTRY [ 0xffff8488`aa562b00 - 0xffff8488`aa562b00 ]
   +0x010 Name             : _UNICODE_STRING "Process"
   +0x020 DefaultObject    : (null) 
   +0x028 Index            : 0x7 ''
   +0x02c TotalNumberOfObjects : 0xa1
   +0x030 TotalNumberOfHandles : 0x524
   +0x034 HighWaterNumberOfObjects : 0xa4
   +0x038 HighWaterNumberOfHandles : 0x548
   +0x040 TypeInfo         : _OBJECT_TYPE_INITIALIZER
   +0x0b8 TypeLock         : _EX_PUSH_LOCK
   +0x0c0 Key              : 0x636f7250
   +0x0c8 CallbackList     : _LIST_ENTRY [ 0xffff9606`6a412b00 - 0xffff9606`6a429780 ]

3: kd> dt_OBJECT_TYPE_INITIALIZER ffff8488`aa562b00+40
ntdll!_OBJECT_TYPE_INITIALIZER
   +0x000 Length           : 0x78
   +0x002 ObjectTypeFlags  : 0xca
   +0x002 CaseInsensitive  : 0y0
   +0x002 UnnamedObjectsOnly : 0y1
   +0x002 UseDefaultObject : 0y0
   +0x002 SecurityRequired : 0y1
   +0x002 MaintainHandleCount : 0y0
   +0x002 MaintainTypeList : 0y0
   +0x002 SupportsObjectCallbacks : 0y1
   +0x002 CacheAligned     : 0y1
   +0x003 UseExtendedParameters : 0y0
   +0x003 Reserved         : 0y0000000 (0)
   +0x004 ObjectTypeCode   : 0x20
   +0x008 InvalidAttributes : 0xb0
   +0x00c GenericMapping   : _GENERIC_MAPPING
   +0x01c ValidAccessMask  : 0x1fffff
   +0x020 RetainAccess     : 0x101000
   +0x024 PoolType         : 200 ( NonPagedPoolNx )
   +0x028 DefaultPagedPoolCharge : 0x1000
   +0x02c DefaultNonPagedPoolCharge : 0xa98
   +0x030 DumpProcedure    : (null) 
   +0x038 OpenProcedure    : 0xffff8488`aa5624e0     long  +ffff8488aa5624e0
   +0x040 CloseProcedure   : 0xfffff800`6d46e6e0     void  nt!PspProcessClose+0
   +0x048 DeleteProcedure  : 0xfffff800`6d4039e0     void  nt!PspProcessDelete+0
   +0x050 ParseProcedure   : (null) 
   +0x050 ParseProcedureEx : (null) 
   +0x058 SecurityProcedure : 0xfffff800`6d4d8eb0     long  nt!SeDefaultObjectMethod+0
   +0x060 QueryNameProcedure : (null) 
   +0x068 OkayToCloseProcedure : (null) 
   +0x070 WaitObjectFlagMask : 0
   +0x074 WaitObjectFlagOffset : 0
   +0x076 WaitObjectPointerOffset : 0
```

이전에 확인된 내용과 동일하게 OpenProcedure(PspProcessOpen) 함수가 함수 테이블을 통해 후킹되어 있습니다.
```
3: kd> dt_OBJECT_TYPE_INITIALIZER ffff8488`aa562b00+40
ntdll!_OBJECT_TYPE_INITIALIZER
   +0x038 OpenProcedure    : 0xffff8488`aa5624e0     long  +ffff8488aa5624e0
   +0x040 CloseProcedure   : 0xfffff800`6d46e6e0     void  nt!PspProcessClose+0
   +0x048 DeleteProcedure  : 0xfffff800`6d4039e0     void  nt!PspProcessDelete+0
```

해킹툴이 적용되지 않은 정상 환경에서는 다음과 같이 OpenProcess가 nt!PspProcessOpen을 나타냅니다.
```
3: kd> dt_OBJECT_TYPE_INITIALIZER ffff8488`aa562b00+40
ntdll!_OBJECT_TYPE_INITIALIZER
   +0x038 OpenProcedure    : 0xfffff800`6d3fd930     long  nt!PspProcessOpen+0
   +0x040 CloseProcedure   : 0xfffff800`6d46e6e0     void  nt!PspProcessClose+0
   +0x048 DeleteProcedure  : 0xfffff800`6d4039e0     void  nt!PspProcessDelete+0
```

## Hookcode
해킹툴에서 사용하는 후킹 코드는 다음과 같으며 다양한 기능을 수행하고 있습니다. 간략하게 핵심적인 부분만 분석 진행하였습니다.
```
3: kd> $$>< C:\git\Windbg Scripts\CheckOpenProcedure
hooked openprocedure.
ffff8488`aa562b78  ffff8488`aa5624e0
ffff8488`aa562b80  fffff800`6d46e6e0 nt!PspProcessClose
ffff8488`aa562b88  fffff800`6d4039e0 nt!PspProcessDelete

ffff8488`aa5624e0 89542410        mov     dword ptr [rsp+10h],edx
ffff8488`aa5624e4 894c2408        mov     dword ptr [rsp+8],ecx
ffff8488`aa5624e8 55              push    rbp
ffff8488`aa5624e9 53              push    rbx
ffff8488`aa5624ea 56              push    rsi
ffff8488`aa5624eb 57              push    rdi
ffff8488`aa5624ec 4154            push    r12
ffff8488`aa5624ee 4155            push    r13
ffff8488`aa5624f0 4156            push    r14
ffff8488`aa5624f2 4157            push    r15
ffff8488`aa5624f4 488d6c24e9      lea     rbp,[rsp-17h]
ffff8488`aa5624f9 4881ece8000000  sub     rsp,0E8h
ffff8488`aa562500 48bb302456aa8884ffff mov rbx,0FFFF8488AA562430h
ffff8488`aa56250a 33ff            xor     edi,edi
ffff8488`aa56250c 4d8bf9          mov     r15,r9
ffff8488`aa56250f 498bf0          mov     rsi,r8
ffff8488`aa562512 448bf1          mov     r14d,ecx
ffff8488`aa562515 488b1b          mov     rbx,qword ptr [rbx]
ffff8488`aa562518 48897d6f        mov     qword ptr [rbp+6Fh],rdi
ffff8488`aa56251c ff5348          call    qword ptr [rbx+48h]
ffff8488`aa56251f 4c8b657f        mov     r12,qword ptr [rbp+7Fh]
ffff8488`aa562523 4c8be8          mov     r13,rax
ffff8488`aa562526 493bf7          cmp     rsi,r15
ffff8488`aa562529 0f854e040000    jne     ffff8488`aa56297d  Branch

ffff8488`aa56252f 4d85e4          test    r12,r12
ffff8488`aa562532 0f8445040000    je      ffff8488`aa56297d  Branch

ffff8488`aa562538 41813c2401010000 cmp     dword ptr [r12],101h
ffff8488`aa562540 0f8537040000    jne     ffff8488`aa56297d  Branch

ffff8488`aa562546 488b4b10        mov     rcx,qword ptr [rbx+10h]
ffff8488`aa56254a 4885c9          test    rcx,rcx
ffff8488`aa56254d 0f85cb000000    jne     ffff8488`aa56261e  Branch

ffff8488`aa562553 488bce          mov     rcx,rsi
ffff8488`aa562556 ff5358          call    qword ptr [rbx+58h]
ffff8488`aa562559 4885c0          test    rax,rax
ffff8488`aa56255c 0f841b040000    je      ffff8488`aa56297d  Branch

ffff8488`aa562562 4805000f0000    add     rax,0F00h
ffff8488`aa562568 488d4d6f        lea     rcx,[rbp+6Fh]
ffff8488`aa56256c 48894c2430      mov     qword ptr [rsp+30h],rcx
ffff8488`aa562571 4c8d4d97        lea     r9,[rbp-69h]
ffff8488`aa562575 0f57c0          xorps   xmm0,xmm0
ffff8488`aa562578 40887c2428      mov     byte ptr [rsp+28h],dil
ffff8488`aa56257d 488bce          mov     rcx,rsi
ffff8488`aa562580 48c744242010000000 mov   qword ptr [rsp+20h],10h
ffff8488`aa562589 4d8bc5          mov     r8,r13
ffff8488`aa56258c 48894577        mov     qword ptr [rbp+77h],rax
ffff8488`aa562590 488bd0          mov     rdx,rax
ffff8488`aa562593 0f114597        movups  xmmword ptr [rbp-69h],xmm0
ffff8488`aa562597 ff5370          call    qword ptr [rbx+70h]
ffff8488`aa56259a 85c0            test    eax,eax
ffff8488`aa56259c 0f88db030000    js      ffff8488`aa56297d  Branch

ffff8488`aa5625a2 48b800b9dcfeefcdab60 mov rax,60ABCDEFFEDCB900h
ffff8488`aa5625ac 48394597        cmp     qword ptr [rbp-69h],rax
ffff8488`aa5625b0 0f85c7030000    jne     ffff8488`aa56297d  Branch

ffff8488`aa5625b6 488b4d9f        mov     rcx,qword ptr [rbp-61h]
ffff8488`aa5625ba 49be0000fdffff7f0000 mov r14,7FFFFFFD0000h
ffff8488`aa5625c4 488d810000ffff  lea     rax,[rcx-10000h]
ffff8488`aa5625cb 493bc6          cmp     rax,r14
ffff8488`aa5625ce 0f87a5030000    ja      ffff8488`aa562979  Branch

ffff8488`aa5625d4 48894b18        mov     qword ptr [rbx+18h],rcx
ffff8488`aa5625d8 488bce          mov     rcx,rsi
ffff8488`aa5625db 48897310        mov     qword ptr [rbx+10h],rsi
ffff8488`aa5625df ff9380000000    call    qword ptr [rbx+80h]
ffff8488`aa5625e5 4c8b4d77        mov     r9,qword ptr [rbp+77h]
ffff8488`aa5625e9 488d456f        lea     rax,[rbp+6Fh]
ffff8488`aa5625ed 4889442430      mov     qword ptr [rsp+30h],rax
ffff8488`aa5625f2 488d5597        lea     rdx,[rbp-69h]
ffff8488`aa5625f6 0f57c0          xorps   xmm0,xmm0
ffff8488`aa5625f9 40887c2428      mov     byte ptr [rsp+28h],dil
ffff8488`aa5625fe 4c8bc6          mov     r8,rsi
ffff8488`aa562601 48c744242010000000 mov   qword ptr [rsp+20h],10h
ffff8488`aa56260a 498bcd          mov     rcx,r13
ffff8488`aa56260d 0f114597        movups  xmmword ptr [rbp-69h],xmm0
ffff8488`aa562611 ff5370          call    qword ptr [rbx+70h]

ffff8488`aa562614 b8010000c0      mov     eax,0C0000001h
ffff8488`aa562619 e9c1030000      jmp     ffff8488`aa5629df  Branch

ffff8488`aa56261e 483bf1          cmp     rsi,rcx
ffff8488`aa562621 0f8556030000    jne     ffff8488`aa56297d  Branch

ffff8488`aa562627 488b5318        mov     rdx,qword ptr [rbx+18h]
ffff8488`aa56262b 488d456f        lea     rax,[rbp+6Fh]
ffff8488`aa56262f 4889442430      mov     qword ptr [rsp+30h],rax
ffff8488`aa562634 4c8d4c2448      lea     r9,[rsp+48h]
ffff8488`aa562639 0f57c0          xorps   xmm0,xmm0
ffff8488`aa56263c 40887c2428      mov     byte ptr [rsp+28h],dil
ffff8488`aa562641 41bf20000000    mov     r15d,20h
ffff8488`aa562647 4d8bc5          mov     r8,r13
ffff8488`aa56264a 4c897c2420      mov     qword ptr [rsp+20h],r15
ffff8488`aa56264f 0f11442448      movups  xmmword ptr [rsp+48h],xmm0
ffff8488`aa562654 0f114587        movups  xmmword ptr [rbp-79h],xmm0
ffff8488`aa562658 ff5370          call    qword ptr [rbx+70h]
ffff8488`aa56265b 85c0            test    eax,eax
ffff8488`aa56265d 78b5            js      ffff8488`aa562614  Branch

ffff8488`aa56265f 8b442448        mov     eax,dword ptr [rsp+48h]
ffff8488`aa562663 2500ffffff      and     eax,0FFFFFF00h
ffff8488`aa562668 3d00b9dcfe      cmp     eax,0FEDCB900h
ffff8488`aa56266d 75a5            jne     ffff8488`aa562614  Branch

ffff8488`aa56266f 397c244c        cmp     dword ptr [rsp+4Ch],edi
ffff8488`aa562673 749f            je      ffff8488`aa562614  Branch

ffff8488`aa562675 397d8f          cmp     dword ptr [rbp-71h],edi
ffff8488`aa562678 749a            je      ffff8488`aa562614  Branch

ffff8488`aa56267a 488b4587        mov     rax,qword ptr [rbp-79h]
ffff8488`aa56267e 49be0000fdffff7f0000 mov r14,7FFFFFFD0000h
ffff8488`aa562688 48050000ffff    add     rax,0FFFFFFFFFFFF0000h
ffff8488`aa56268e 493bc6          cmp     rax,r14
ffff8488`aa562691 7781            ja      ffff8488`aa562614  Branch

ffff8488`aa562693 8b4c244c        mov     ecx,dword ptr [rsp+4Ch]
ffff8488`aa562697 488d5577        lea     rdx,[rbp+77h]
ffff8488`aa56269b 48897d77        mov     qword ptr [rbp+77h],rdi
ffff8488`aa56269f ff5378          call    qword ptr [rbx+78h]
ffff8488`aa5626a2 85c0            test    eax,eax
ffff8488`aa5626a4 0f886affffff    js      ffff8488`aa562614  Branch

ffff8488`aa5626aa 8b442448        mov     eax,dword ptr [rsp+48h]
ffff8488`aa5626ae 3d01b9dcfe      cmp     eax,0FEDCB901h
ffff8488`aa5626b3 0f843f020000    je      ffff8488`aa5628f8  Branch

ffff8488`aa5626b9 3d02b9dcfe      cmp     eax,0FEDCB902h
ffff8488`aa5626be 0f84fd010000    je      ffff8488`aa5628c1  Branch

ffff8488`aa5626c4 3d03b9dcfe      cmp     eax,0FEDCB903h
ffff8488`aa5626c9 0f84b3010000    je      ffff8488`aa562882  Branch

ffff8488`aa5626cf 3d04b9dcfe      cmp     eax,0FEDCB904h
ffff8488`aa5626d4 0f842b010000    je      ffff8488`aa562805  Branch

ffff8488`aa5626da 3d05b9dcfe      cmp     eax,0FEDCB905h
ffff8488`aa5626df 0f84d2000000    je      ffff8488`aa5627b7  Branch

ffff8488`aa5626e5 3d06b9dcfe      cmp     eax,0FEDCB906h
ffff8488`aa5626ea 0f8492000000    je      ffff8488`aa562782  Branch

ffff8488`aa5626f0 3d07b9dcfe      cmp     eax,0FEDCB907h
ffff8488`aa5626f5 0f8539020000    jne     ffff8488`aa562934  Branch

ffff8488`aa5626fb 488b4d77        mov     rcx,qword ptr [rbp+77h]
ffff8488`aa5626ff 488d55d7        lea     rdx,[rbp-29h]
ffff8488`aa562703 0f57c0          xorps   xmm0,xmm0
ffff8488`aa562706 0f1145d7        movups  xmmword ptr [rbp-29h],xmm0
ffff8488`aa56270a 0f1145e7        movups  xmmword ptr [rbp-19h],xmm0
ffff8488`aa56270e 0f1145f7        movups  xmmword ptr [rbp-9],xmm0
ffff8488`aa562712 ff5338          call    qword ptr [rbx+38h]
ffff8488`aa562715 488b542450      mov     rdx,qword ptr [rsp+50h]
ffff8488`aa56271a 488d442440      lea     rax,[rsp+40h]
ffff8488`aa56271f 0f57c0          xorps   xmm0,xmm0
ffff8488`aa562722 4889442428      mov     qword ptr [rsp+28h],rax
ffff8488`aa562727 458d7710        lea     r14d,[r15+10h]
ffff8488`aa56272b 48897c2440      mov     qword ptr [rsp+40h],rdi
ffff8488`aa562730 4c8d4da7        lea     r9,[rbp-59h]
ffff8488`aa562734 4c89742420      mov     qword ptr [rsp+20h],r14
ffff8488`aa562739 4533c0          xor     r8d,r8d
ffff8488`aa56273c 4883c9ff        or      rcx,0FFFFFFFFFFFFFFFFh
ffff8488`aa562740 0f1145a7        movups  xmmword ptr [rbp-59h],xmm0
ffff8488`aa562744 0f1145b7        movups  xmmword ptr [rbp-49h],xmm0
ffff8488`aa562748 0f1145c7        movups  xmmword ptr [rbp-39h],xmm0
ffff8488`aa56274c ff93a0000000    call    qword ptr [rbx+0A0h]
ffff8488`aa562752 488d4dd7        lea     rcx,[rbp-29h]
ffff8488`aa562756 8bf0            mov     esi,eax
ffff8488`aa562758 ff5340          call    qword ptr [rbx+40h]
ffff8488`aa56275b 85f6            test    esi,esi
ffff8488`aa56275d 0f88dd010000    js      ffff8488`aa562940  Branch

ffff8488`aa562763 488d456f        lea     rax,[rbp+6Fh]
ffff8488`aa562767 4889442430      mov     qword ptr [rsp+30h],rax
ffff8488`aa56276c 488d55a7        lea     rdx,[rbp-59h]
ffff8488`aa562770 40887c2428      mov     byte ptr [rsp+28h],dil
ffff8488`aa562775 4c89742420      mov     qword ptr [rsp+20h],r14

ffff8488`aa56277a 498bcd          mov     rcx,r13
ffff8488`aa56277d e9a1010000      jmp     ffff8488`aa562923  Branch

ffff8488`aa562782 837d8f18        cmp     dword ptr [rbp-71h],18h
ffff8488`aa562786 0f8565010000    jne     ffff8488`aa5628f1  Branch

ffff8488`aa56278c 488d456f        lea     rax,[rbp+6Fh]
ffff8488`aa562790 4d8bc5          mov     r8,r13
ffff8488`aa562793 4889442430      mov     qword ptr [rsp+30h],rax
ffff8488`aa562798 4c8d4b20        lea     r9,[rbx+20h]
ffff8488`aa56279c 40887c2428      mov     byte ptr [rsp+28h],dil
ffff8488`aa5627a1 48c744242018000000 mov   qword ptr [rsp+20h],18h

ffff8488`aa5627aa 488b5587        mov     rdx,qword ptr [rbp-79h]
ffff8488`aa5627ae 488b4b10        mov     rcx,qword ptr [rbx+10h]
ffff8488`aa5627b2 e974010000      jmp     ffff8488`aa56292b  Branch

ffff8488`aa5627b7 488b4d77        mov     rcx,qword ptr [rbp+77h]
ffff8488`aa5627bb 488d55a7        lea     rdx,[rbp-59h]
ffff8488`aa5627bf 0f57c0          xorps   xmm0,xmm0
ffff8488`aa5627c2 0f1145a7        movups  xmmword ptr [rbp-59h],xmm0
ffff8488`aa5627c6 0f1145b7        movups  xmmword ptr [rbp-49h],xmm0
ffff8488`aa5627ca 0f1145c7        movups  xmmword ptr [rbp-39h],xmm0
ffff8488`aa5627ce ff5338          call    qword ptr [rbx+38h]
ffff8488`aa5627d1 488b4587        mov     rax,qword ptr [rbp-79h]
ffff8488`aa5627d5 4c8d442440      lea     r8,[rsp+40h]
ffff8488`aa5627da 41b900800000    mov     r9d,8000h
ffff8488`aa5627e0 48894597        mov     qword ptr [rbp-69h],rax
ffff8488`aa5627e4 488d5597        lea     rdx,[rbp-69h]
ffff8488`aa5627e8 48897c2440      mov     qword ptr [rsp+40h],rdi
ffff8488`aa5627ed 4883c9ff        or      rcx,0FFFFFFFFFFFFFFFFh
ffff8488`aa5627f1 ff9398000000    call    qword ptr [rbx+98h]
ffff8488`aa5627f7 488d4da7        lea     rcx,[rbp-59h]
ffff8488`aa5627fb 8bf0            mov     esi,eax
ffff8488`aa5627fd ff5340          call    qword ptr [rbx+40h]
ffff8488`aa562800 e92b010000      jmp     ffff8488`aa562930  Branch

ffff8488`aa562805 488b4d77        mov     rcx,qword ptr [rbp+77h]
ffff8488`aa562809 488d55a7        lea     rdx,[rbp-59h]
ffff8488`aa56280d 0f57c0          xorps   xmm0,xmm0
ffff8488`aa562810 0f1145a7        movups  xmmword ptr [rbp-59h],xmm0
ffff8488`aa562814 0f1145b7        movups  xmmword ptr [rbp-49h],xmm0
ffff8488`aa562818 0f1145c7        movups  xmmword ptr [rbp-39h],xmm0
ffff8488`aa56281c ff5338          call    qword ptr [rbx+38h]
ffff8488`aa56281f 8b458f          mov     eax,dword ptr [rbp-71h]
ffff8488`aa562822 4c8d4d97        lea     r9,[rbp-69h]
ffff8488`aa562826 48894597        mov     qword ptr [rbp-69h],rax
ffff8488`aa56282a 488d542440      lea     rdx,[rsp+40h]
ffff8488`aa56282f 8b4593          mov     eax,dword ptr [rbp-6Dh]
ffff8488`aa562832 4533c0          xor     r8d,r8d
ffff8488`aa562835 89442428        mov     dword ptr [rsp+28h],eax
ffff8488`aa562839 4883c9ff        or      rcx,0FFFFFFFFFFFFFFFFh
ffff8488`aa56283d c744242000301000 mov     dword ptr [rsp+20h],103000h
ffff8488`aa562845 48897c2440      mov     qword ptr [rsp+40h],rdi
ffff8488`aa56284a ff9390000000    call    qword ptr [rbx+90h]
ffff8488`aa562850 488d4da7        lea     rcx,[rbp-59h]
ffff8488`aa562854 8bf0            mov     esi,eax
ffff8488`aa562856 ff5340          call    qword ptr [rbx+40h]
ffff8488`aa562859 85f6            test    esi,esi
ffff8488`aa56285b 0f88df000000    js      ffff8488`aa562940  Branch

ffff8488`aa562861 488d456f        lea     rax,[rbp+6Fh]
ffff8488`aa562865 4889442430      mov     qword ptr [rsp+30h],rax
ffff8488`aa56286a 488d542440      lea     rdx,[rsp+40h]
ffff8488`aa56286f 40887c2428      mov     byte ptr [rsp+28h],dil
ffff8488`aa562874 48c744242008000000 mov   qword ptr [rsp+20h],8
ffff8488`aa56287d e9f8feffff      jmp     ffff8488`aa56277a  Branch

ffff8488`aa562882 837d8f08        cmp     dword ptr [rbp-71h],8
ffff8488`aa562886 488b4d77        mov     rcx,qword ptr [rbp+77h]
ffff8488`aa56288a 7505            jne     ffff8488`aa562891  Branch

ffff8488`aa56288c ff5358          call    qword ptr [rbx+58h]
ffff8488`aa56288f eb03            jmp     ffff8488`aa562894  Branch

ffff8488`aa562891 ff5360          call    qword ptr [rbx+60h]

ffff8488`aa562894 48894597        mov     qword ptr [rbp-69h],rax
ffff8488`aa562898 4885c0          test    rax,rax
ffff8488`aa56289b 0f8493000000    je      ffff8488`aa562934  Branch

ffff8488`aa5628a1 488d456f        lea     rax,[rbp+6Fh]
ffff8488`aa5628a5 4889442430      mov     qword ptr [rsp+30h],rax
ffff8488`aa5628aa 488d5597        lea     rdx,[rbp-69h]
ffff8488`aa5628ae 40887c2428      mov     byte ptr [rsp+28h],dil
ffff8488`aa5628b3 48c744242008000000 mov   qword ptr [rsp+20h],8
ffff8488`aa5628bc e9b9feffff      jmp     ffff8488`aa56277a  Branch

ffff8488`aa5628c1 4c8b4c2450      mov     r9,qword ptr [rsp+50h]
ffff8488`aa5628c6 498d810000ffff  lea     rax,[r9-10000h]
ffff8488`aa5628cd 493bc6          cmp     rax,r14
ffff8488`aa5628d0 771f            ja      ffff8488`aa5628f1  Branch

ffff8488`aa5628d2 8b458f          mov     eax,dword ptr [rbp-71h]
ffff8488`aa5628d5 488d4d6f        lea     rcx,[rbp+6Fh]
ffff8488`aa5628d9 4c8b4577        mov     r8,qword ptr [rbp+77h]
ffff8488`aa5628dd 48894c2430      mov     qword ptr [rsp+30h],rcx
ffff8488`aa5628e2 c644242801      mov     byte ptr [rsp+28h],1
ffff8488`aa5628e7 4889442420      mov     qword ptr [rsp+20h],rax
ffff8488`aa5628ec e9b9feffff      jmp     ffff8488`aa5627aa  Branch

ffff8488`aa5628f1 be010000c0      mov     esi,0C0000001h
ffff8488`aa5628f6 eb38            jmp     ffff8488`aa562930  Branch

ffff8488`aa5628f8 488b542450      mov     rdx,qword ptr [rsp+50h]
ffff8488`aa5628fd 488d820000ffff  lea     rax,[rdx-10000h]
ffff8488`aa562904 493bc6          cmp     rax,r14
ffff8488`aa562907 7732            ja      ffff8488`aa56293b  Branch

ffff8488`aa562909 8b458f          mov     eax,dword ptr [rbp-71h]
ffff8488`aa56290c 488d4d6f        lea     rcx,[rbp+6Fh]
ffff8488`aa562910 48894c2430      mov     qword ptr [rsp+30h],rcx
ffff8488`aa562915 488b4d77        mov     rcx,qword ptr [rbp+77h]
ffff8488`aa562919 c644242801      mov     byte ptr [rsp+28h],1
ffff8488`aa56291e 4889442420      mov     qword ptr [rsp+20h],rax

ffff8488`aa562923 4c8b4d87        mov     r9,qword ptr [rbp-79h]
ffff8488`aa562927 4c8b4310        mov     r8,qword ptr [rbx+10h]

ffff8488`aa56292b ff5370          call    qword ptr [rbx+70h]
ffff8488`aa56292e 8bf0            mov     esi,eax

ffff8488`aa562930 85f6            test    esi,esi
ffff8488`aa562932 780c            js      ffff8488`aa562940  Branch

ffff8488`aa562934 be01000000      mov     esi,1
ffff8488`aa562939 eb05            jmp     ffff8488`aa562940  Branch

ffff8488`aa56293b be010000c0      mov     esi,0C0000001h

ffff8488`aa562940 4c8b4b18        mov     r9,qword ptr [rbx+18h]
ffff8488`aa562944 488d456f        lea     rax,[rbp+6Fh]
ffff8488`aa562948 4c8b4310        mov     r8,qword ptr [rbx+10h]
ffff8488`aa56294c 488d542448      lea     rdx,[rsp+48h]
ffff8488`aa562951 4889442430      mov     qword ptr [rsp+30h],rax
ffff8488`aa562956 498bcd          mov     rcx,r13
ffff8488`aa562959 40887c2428      mov     byte ptr [rsp+28h],dil
ffff8488`aa56295e 4c897c2420      mov     qword ptr [rsp+20h],r15
ffff8488`aa562963 89742448        mov     dword ptr [rsp+48h],esi
ffff8488`aa562967 ff5370          call    qword ptr [rbx+70h]
ffff8488`aa56296a 488b4d77        mov     rcx,qword ptr [rbp+77h]
ffff8488`aa56296e ff9388000000    call    qword ptr [rbx+88h]
ffff8488`aa562974 e99bfcffff      jmp     ffff8488`aa562614  Branch

ffff8488`aa562979 448b755f        mov     r14d,dword ptr [rbp+5Fh]

ffff8488`aa56297d 488b4b10        mov     rcx,qword ptr [rbx+10h]
ffff8488`aa562981 4885c9          test    rcx,rcx
ffff8488`aa562984 7418            je      ffff8488`aa56299e  Branch

ffff8488`aa562986 ff5368          call    qword ptr [rbx+68h]
ffff8488`aa562989 3d03010000      cmp     eax,103h
ffff8488`aa56298e 740e            je      ffff8488`aa56299e  Branch

ffff8488`aa562990 488b4b10        mov     rcx,qword ptr [rbx+10h]
ffff8488`aa562994 ff9388000000    call    qword ptr [rbx+88h]
ffff8488`aa56299a 48897b10        mov     qword ptr [rbx+10h],rdi

ffff8488`aa56299e 493bf7          cmp     rsi,r15
ffff8488`aa5629a1 7428            je      ffff8488`aa5629cb  Branch

ffff8488`aa5629a3 4585f6          test    r14d,r14d
ffff8488`aa5629a6 7423            je      ffff8488`aa5629cb  Branch

ffff8488`aa5629a8 498bcf          mov     rcx,r15
ffff8488`aa5629ab ff5350          call    qword ptr [rbx+50h]
ffff8488`aa5629ae 4885c0          test    rax,rax
ffff8488`aa5629b1 7418            je      ffff8488`aa5629cb  Branch

ffff8488`aa5629b3 488d4b20        lea     rcx,[rbx+20h]

ffff8488`aa5629b7 483901          cmp     qword ptr [rcx],rax
ffff8488`aa5629ba 0f8454fcffff    je      ffff8488`aa562614  Branch

ffff8488`aa5629c0 ffc7            inc     edi
ffff8488`aa5629c2 4883c108        add     rcx,8
ffff8488`aa5629c6 83ff03          cmp     edi,3
ffff8488`aa5629c9 72ec            jb      ffff8488`aa5629b7  Branch

ffff8488`aa5629cb 8b5567          mov     edx,dword ptr [rbp+67h]
ffff8488`aa5629ce 4d8bcf          mov     r9,r15
ffff8488`aa5629d1 4c8bc6          mov     r8,rsi
ffff8488`aa5629d4 4c89642420      mov     qword ptr [rsp+20h],r12
ffff8488`aa5629d9 418bce          mov     ecx,r14d
ffff8488`aa5629dc ff5308          call    qword ptr [rbx+8]

ffff8488`aa5629df 4881c4e8000000  add     rsp,0E8h
ffff8488`aa5629e6 415f            pop     r15
ffff8488`aa5629e8 415e            pop     r14
ffff8488`aa5629ea 415d            pop     r13
ffff8488`aa5629ec 415c            pop     r12
ffff8488`aa5629ee 5f              pop     rdi
ffff8488`aa5629ef 5e              pop     rsi
ffff8488`aa5629f0 5b              pop     rbx
ffff8488`aa5629f1 5d              pop     rbp
ffff8488`aa5629f2 c3              ret
```

### DataTable
다음과 같은 데이터 테이블을 NonPagedPool 영역에 할당하여 사용합니다. 보호 대상 목록, 서브 프로세스, 원본 함수 등 다양한 정보들이 포함되어 있습니다.
```cpp
typedef struct _MUXUE_hookTable
{
    PVOID           hookTableAddr;                     // 현재 테이블의 주소
    PVOID           fpPspProcessOpen;                  // PspProcessOpen 원본 주소
    PEPROCESS       subProcess;                        // 서브 프로세스 (통신 프로세스)
    PVOID           virtualAddr;                       // 통신 버퍼 (가상 메모리 주소)
    HANDLE          processId[PROTECTION_COUNT];       // 보호 대상 프로세스 ID

    // 후킹 코드에서 사용하는 함수는 전부 아래의 테이블을 통해 호출함.
    struct _MUXUE_FUNCTION_TABLE
    {
        PVOID fpKeStackAttachProcess;             // KeStackAttachProcess    
        PVOID fpKeUnstackDetachProcess;           // KeUnstackDetachProcess;
        PVOID fpPsGetCurrentProcess;              // PsGetCurrentProcess
        PVOID fpPsGetProcessId;                   // PsGetProcessId
        PVOID fpPsGetProcessPeb;                  // PsGetProcessPeb
        PVOID fpPsGetProcessWow64Process;         // PsGetProcessWow64Process
        PVOID fpPsGetProcessExitStatus;           // PsGetProcessExitStatus
        PVOID fpMmCopyVirtualMemory;              // MmCopyVirtualMemory
        PVOID fpPsLookupProcessByProcessId;       // PsLookupProcessByProcessId
        PVOID fpObfReferenceObject;               // ObfReferenceObject
        PVOID fpHalPutDmaAdapter;                 // HalPutDmaAdapter
        PVOID fpZwAllocateVirtualMemory;          // ZwAllocateVirtualMemory
        PVOID fpZwFreeVirtualMemory;              // ZwFreeVirtualMemory
        PVOID fpZwQueryVirtualMemory;             // ZwQueryVirtualMemory
    }ntCall;

}MUXUE_hookTable, *PMUXUE_hookTable;
```
### Protocol
후킹된 함수를 통해 유저 레벨 애플리케이션과 통신합니다. PEB 영역을 입출력 버퍼로 사용합니다.
```cpp
typedef enum _MUXUE_IO_COMMAND : DWORD32
{
    MUXUE_IO_COMMAND_NULL     = 0xFEDCB900,
    MUXUE_IO_COMMAND_1        = 0xFEDCB901,
    MUXUE_IO_COMMAND_2        = 0xFEDCB902,
    MUXUE_IO_COMMAND_3        = 0xFEDCB903,
    MUXUE_IO_COMMAND_4        = 0xFEDCB904,
    MUXUE_IO_COMMAND_5        = 0xFEDCB905,
    MUXUE_IO_COMMAND_6        = 0xFEDCB906,
    MUXUE_IO_COMMAND_7        = 0xFEDCB907,
    MUXUE_IO_COMMAND_8        = 0xFEDCB908,
}MUXUE_IO_COMMAND;

// Device I/O
if (hookTable->ntCall.fpMmCopyVirtualMemory(hookTable->subProcess, hookTable->virtualAddr, current_process, &io_buffer, 0x20, 0, &return_size) < 0)
    return STATUS_UNSUCCESSFUL;

switch ((_DWORD)io_buffer)
{
    case MUXUE_IO_COMMAND_3: ...
    case MUXUE_IO_COMMAND_4: ...
    case MUXUE_IO_COMMAND_5: ...
    case MUXUE_IO_COMMAND_6: ...
....
}
```
### SubProcess
특정 프로세스를 서브 프로세스로 전환합니다. 전환되면 보호 대상 목록에 추가되며, PEB 영역을 통해 커널 영역과 통신합니다.
```cpp
    // 서브 프로세스 지정
    PPEB peb = hookTable->ntCall.fpPsGetProcessPeb(process);
    if (peb)
    {
        dwm_process = peb + 0xF00;
        dwm_peb_ = 0;
        if (hookTable->ntCall.fpMmCopyVirtualMemory(process, peb + 0xF00, current_process, &dwm_peb_, 16, 0, &return_size) >= 0 &&
            (_QWORD)dwm_peb_ == 0x60ABCDEFFEDCB900)
        {
            if ((unsigned __int64)(*((_QWORD*)&dwm_peb_ + 1) - 0x10000) <= 0x7FFFFFFD0000)
            {
                hookTable->virtualAddr    = *((_QWORD*)&dwm_peb_ + 1);
                hookTable->subProcess    = process;
                hookTable->ntCall.fpObfReferenceObject(process);
                LOBYTE(v28) = 0;
                dwm_peb_ = 0;

                // 입력 버퍼 정리
                hookTable->ntCall.MmCopyVirtualMemory(current_process, &dwm_peb_, process, dwm_process, 16, v28, &return_size);
                return STATUS_UNSUCCESSFUL;
            }
        }
    }
```

### OpenProcedure
PspProcessOpen을 후킹하여 실행되는 코드입니다. 다른 프로세스에서 보호 프로세스에 접근하면 STATUS_UNSUCCESSFUL을 반환하고, 아닌 경우에 원본 함수를 호출합니다. 해킹툴 프로세스를 OpenProcess 시도했을 때 실패한 이유입니다.
```cpp
NTSTATUS PspProcessOpen_Hook(OB_OPEN_REASON OpenReason, BYTE AccessMode, PEPROCESS process, PVOID Object, ACCESS_MASK* GrantedAccess, ULONG HandleCount)
{
...
    if (process != Object)
    {
        if (OpenReason != ObCreateHandle)
        {
            HANDLE processId = hookTable->ntCall.fpPsGetProcessId(Object);

            if (processId)
            {
                //
                // 접근하려는 프로세스가 보호 대상이면 실패를 반환함.
                // OpenProcess가 실패하는 이유..
                // 
                // [보호 프로세스 목록]
                // 1. 서브 프로세스
                // 2. DWM
                // 

                for (int i = 0; i < PROTECTION_COUNT; ++i)
                {
                    if (hookTable->processId[i] == processId)
                    {
                        return STATUS_UNSUCCESSFUL;
                    }
                }

                return hookTable->ntCall.fpPspProcessOpen(OpenReason, AccessMode, process, Object, GrantedAccess, HandleCount);
            }
        }
    }

    return hookTable->ntCall.fpPspProcessOpen(OpenReason, AccessMode, process, Object, GrantedAccess, HandleCount);
}
```

## Windbg Script
덤프 분석 시 활용할 수 있는 스크립트를 개발하였습니다. Export 변수인 PsProcessType를 참조하여 OpenProcedure(PspProcessOpen)이 후킹되어있는지 검사합니다. 해당 코드는 [GitHub](https://github.com/wlsgjd/Windbg-Scripts/blob/main/CheckOpenProcedure)에도 업로드되어 있습니다.
```
$$ OpenProcedure 후킹 체크
$$ 작성 버전 : Windows 10 Kernel Version 19041 MP (4 procs) Free x64 
$$ 사용 방법 : CheckOpenProcedure
$$ 작성일 : 2022-03-16
$$ 작성자 : wlsgjd

r $t0 = poi(nt!PsProcessType)

$$ object_type->TypeInfo->OpenProcedure
r $t1 = poi(@$t0 + 0x78)

.if (@$t1 != nt!PspProcessOpen)
{
	.printf "hooked openprocedure.\n"
    dqs (@$t0 + 0x78) L? 3
	.printf "\n"
	u @$t1
}
.else
{
	.printf "clean openprocedure.\n"
}
```
사용 방법은 다음과 같습니다. 후킹된 경우에는 변조된 위치와 호출되는 함수 코드를 출력합니다.
```
5: kd> $$>< C:\git\Windbg Scripts\CheckOpenProcedure
hooked openprocedure.
ffffd808`35165b48  fffff803`4f9e3f68
ffffd808`35165b50  fffff803`324c4e00 nt!PspProcessClose
ffffd808`35165b58  fffff803`3245f840 nt!PspProcessDelete

fffff803`4f9e3f68 e9e1001500      jmp     fffff803`4fb3404e
fffff803`4f9e3f6d cc              int     3
fffff803`4f9e3f6e cc              int     3
fffff803`4f9e3f6f cc              int     3
fffff803`4f9e3f70 cc              int     3
fffff803`4f9e3f71 cc              int     3
fffff803`4f9e3f72 cc              int     3
fffff803`4f9e3f73 cc              int     3
```
후킹되지 않은 경우엔 다음과 같이 클린 메세지를 출력합니다.
```
2: kd> $$>< C:\git\Windbg Scripts\CheckOpenProcedure
clean openprocedure.
```

## POC Code
분석한 내용을 바탕으로 해킹툴과 동일하게 동작하는 코드를 개발하였습니다. 전체 소스코드는 [GitHub](https://github.com/wlsgjd/MyPOC/tree/main/HookProcessOpen)에 업로드 하였습니다.

후킹 코드는 드라이버가 언로드 되어도 동작해야 하기 때문에 다음과 같이 NonPagedPool 영역에 복사하였습니다.
```cpp
OB_OPEN_METHOD fpMyPspProcessOpen = ExAllocatePool(NonPagedPool, HOOK_FUNC_SIZE);
RtlCopyMemory(fpMyPspProcessOpen, MyPspProcessOpen, HOOK_FUNC_SIZE);
```
또한, 원본 함수 및 보호 대상을 포함하고 있는 데이터 테이블을 NonPagedPool 영역에 생성하였습니다. 해당 메모리 구조는 해킹툴과 동일하게 제작하였습니다.
```cpp
typedef struct _MY_HOOK_TABLE MY_HOOK_TABLE, * PMY_HOOK_TABLE;
typedef struct _MY_HOOK_TABLE
{
	MY_HOOK_TABLE*	this_ptr;
	HANDLE			dwm_pid;
	OB_OPEN_METHOD	old_OpenProcedure;
} MY_HOOK_TABLE, * PMY_HOOK_TABLE;
```
```cpp
PMY_HOOK_TABLE hook_table = ExAllocatePool(NonPagedPool, sizeof(MY_HOOK_TABLE));
hook_table->this_ptr = hook_table;
hook_table->old_OpenProcedure = processType->TypeInfo.OpenProcedure;
hook_table->dwm_pid = GetProcessId(eprocess);
```

그리고, 시그니처 형태의 정적 주소를 NonPagedPool 영역에 할당한 메모리 주소로 교체합니다. 이렇게 되면 드라이버가 언로드 되어도 새로 할당된 NonPagedPool 메모리 영역을 참조하여 실행하게 됩니다.
```cpp
#define HOOK_TABLE_SIG		(ULONG64)0x123456789
#define THIS_PTR(p)			p->this_ptr

#define GET_PROCESS_ID(p)	(*(ULONG64*)((BYTE*)p + 0x440))				// EPROCESS->UniqueProcessId

static NTSTATUS MyPspProcessOpen(OB_OPEN_REASON OpenReason, BYTE AccessMode, PEPROCESS Process, PVOID Object, ACCESS_MASK GrantedAccess, ULONG HandleCount)
{
	PMY_HOOK_TABLE hook_table = HOOK_TABLE_SIG;

	if (Process != Object)
	{
		if (OpenReason != ObCreateHandle)
		{
			if (GET_PROCESS_ID(Object) == THIS_PTR(hook_table)->dwm_pid)
			{
				return STATUS_UNSUCCESSFUL;
			}
		}
	}

	// Disabled cfg(Control Flow Guard)
	return THIS_PTR(hook_table)->old_OpenProcedure(OpenReason, AccessMode, Process, Object, GrantedAccess, HandleCount);
};
```
```cpp
PVOID SearchHookSignature64(PVOID addr, ULONG64 sig, SIZE_T size)
{
	SIZE_T max_size = size - sizeof(ULONG64);

	for (int i = 0; i <= max_size; ++i)
	{
		ULONG64* addr64 = (BYTE*)addr + i;
		if (*addr64 == sig)
			return addr64;
	}

	return 0;
};

int PatchHookTableAddress64(PVOID HookFunc, ULONG64 sig, PMY_HOOK_TABLE HookTable, SIZE_T Size)
{
	int result = 0;

	while (1)
	{
		PULONG64 my_sig = SearchHookSignature64(HookFunc, HOOK_TABLE_SIG, Size);

		if (my_sig)
		{
			*my_sig = HookTable;
			result++;
		}
		else
		{
			break;
		}

	}
	return result;
};
```
```cpp
OB_OPEN_METHOD fpMyPspProcessOpen = ExAllocatePool(NonPagedPool, HOOK_FUNC_SIZE);
RtlCopyMemory(fpMyPspProcessOpen, MyPspProcessOpen, HOOK_FUNC_SIZE);
if (PatchHookTableAddress64(fpMyPspProcessOpen, HOOK_TABLE_SIG, hook_table, HOOK_FUNC_SIZE) > 0)
{
	processType->TypeInfo.OpenProcedure = fpMyPspProcessOpen;

	DbgPrint("ProcessOpenProcedure : 0x%p => 0x%p\n",
		hook_table->old_OpenProcedure,
		processType->TypeInfo.OpenProcedure);
}
```
실행 시 다음과 같이 프로세스가 보호되어 OpenProcess가 실패합니다.
![](/assets/posts/2023-11-14-PspProcessOpen/1.png)

### CFG (Control Flow Guard)
버퍼 오버플로우와 같은 메모리 취약점을 방지하기 위해 만들어진 보안 기능이며 빌드 옵션 중 하나입니다.
```
What is Control Flow Guard?
Control Flow Guard (CFG) is a highly-optimized platform security feature that was created to combat memory corruption vulnerabilities. By placing tight restrictions on where an application can execute code from, it makes it much harder for exploits to execute arbitrary code through vulnerabilities such as buffer overflows. CFG extends previous exploit mitigation technologies such as /GS, DEP, and ASLR.

Prevent memory corruption and ransomware attacks.
Restrict the capabilities of the server to whatever is needed at a particular point in time to reduce attack surface.
Make it harder to exploit arbitrary code through vulnerabilities such as buffer overflows.
```
자세한 내용은 [Control Flow Guard for platform security](https://learn.microsoft.com/en-us/windows/win32/secbp/control-flow-guard) 페이지에서 확인할 수 있습니다.

해당 프로젝트를 빌드하게 되면 원본 프로시저(old_OpenProcedure) 호출 부분에서 포인터 호출이 아닌,  gaurd_dispatch_icall_fptr 형태의 함수가 호출되고 있습니다. 
![](/assets/posts/2023-11-14-PspProcessOpen/3.png)

이 상태에서 코드를 실행해 보면 BSOD가 발생합니다. 해당 오류는 언로드 된 드라이버 영역을 참조했을 때 발생하는 오류입니다.

자세한 내용은 [DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0xce--driver-unloaded-without-cancelling-pending-operations) 페이지에서 확인할 수 있습니다.
![](/assets/posts/2023-11-14-PspProcessOpen/8.png)

덤프를 확인해보면 다음과 같이 정적 메모리 영역이 호출되고 있습니다.
```
6: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS (ce)
A driver unloaded without cancelling timers, DPCs, worker threads, etc.
The broken driver's name is displayed on the screen and saved in
KiBugCheckDriver.
Arguments:
Arg1: fffff8048d141910, memory referenced
Arg2: 0000000000000010, value 0 = read operation, 1 = write operation
Arg3: fffff8048d141910, If non-zero, the instruction address which referenced the bad memory
	address.
Arg4: 0000000000000000, Mm internal code.

FILE_IN_CAB:  MEMORY.dmp
WRITE_ADDRESS:  fffff8048d141910 
IMAGE_NAME:  HookProcessOpen.sys
MODULE_NAME: HookProcessOpen
PROCESS_NAME:  explorer.exe

TRAP_FRAME:  ffff870919a16eb0 -- (.trap 0xffff870919a16eb0)
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=fffff80472cb6290 rbx=0000000000000000 rcx=0000000000000001
rdx=0000000000000001 rsi=0000000000000000 rdi=0000000000000000
rip=fffff8048d141910 rsp=ffff870919a17048 rbp=ffff9d01732f2080
 r8=ffff9d01732f2080  r9=ffff9d0173a430c0 r10=0000000000002484
r11=0000000000000001 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei ng nz na po cy
<Unloaded_HookProcessOpen.sys>+0x1910:
fffff804`8d141910 ??              ???
Resetting default scope

IP_MODULE_UNLOADED: 
HookProcessOpen.sys+1910
fffff804`8d141910 ??              ???

STACK_TEXT:  
ffff8709`19a16c08 fffff804`72a4a91f     : 00000000`00000050 fffff804`8d141910 00000000`00000010 ffff8709`19a16eb0 : nt!KeBugCheckEx
ffff8709`19a16c10 fffff804`7289f540     : ffff8709`19a178d8 00000000`00000010 ffff8709`19a16f30 00000000`00000000 : nt!MiSystemFault+0x18d3bf
ffff8709`19a16d10 fffff804`72a0555e     : ffff9d01`7368b080 fffff804`7289cc86 00000000`00000000 00000000`00000000 : nt!MmAccessFault+0x400
ffff8709`19a16eb0 fffff804`8d141910     : ffff9d01`69ff0052 ffff9d01`7368b6d0 fffff804`72807bae ffff9d01`7368b080 : nt!KiPageFault+0x35e
ffff8709`19a17048 ffff9d01`69ff0052     : ffff9d01`7368b6d0 fffff804`72807bae ffff9d01`7368b080 ffff9d01`73a430a0 : <Unloaded_HookProcessOpen.sys>+0x1910
```
```
// MyPspProcessOpen
6: kd> uf ffff9d01`69ff0000
ffff9d01`69ff0000 4883ec48        sub     rsp,48h
ffff9d01`69ff0004 448bd9          mov     r11d,ecx
ffff9d01`69ff0007 48b9703ffd69019dffff mov rcx,0FFFF9D0169FD3F70h
ffff9d01`69ff0011 4d3bc1          cmp     r8,r9
ffff9d01`69ff0014 741c            je      ffff9d01`69ff0032  Branch

ffff9d01`69ff0016 4585db          test    r11d,r11d
ffff9d01`69ff0019 7417            je      ffff9d01`69ff0032  Branch

ffff9d01`69ff001b 488b01          mov     rax,qword ptr [rcx]
ffff9d01`69ff001e 4c8b5008        mov     r10,qword ptr [rax+8]
ffff9d01`69ff0022 4d399140040000  cmp     qword ptr [r9+440h],r10
ffff9d01`69ff0029 7507            jne     ffff9d01`69ff0032  Branch

ffff9d01`69ff002b b8010000c0      mov     eax,0C0000001h
ffff9d01`69ff0030 eb20            jmp     ffff9d01`69ff0052  Branch

ffff9d01`69ff0032 488b01          mov     rax,qword ptr [rcx]
ffff9d01`69ff0035 8b4c2478        mov     ecx,dword ptr [rsp+78h]
ffff9d01`69ff0039 894c2428        mov     dword ptr [rsp+28h],ecx
ffff9d01`69ff003d 8b4c2470        mov     ecx,dword ptr [rsp+70h]
ffff9d01`69ff0041 488b4010        mov     rax,qword ptr [rax+10h]
ffff9d01`69ff0045 894c2420        mov     dword ptr [rsp+20h],ecx
ffff9d01`69ff0049 418bcb          mov     ecx,r11d
ffff9d01`69ff004c ff15260f0000    call    qword ptr [ffff9d01`69ff0f78] // old_OpenProcedure

ffff9d01`69ff0052 4883c448        add     rsp,48h
ffff9d01`69ff0056 c3              ret

6: kd> dqs ffff9d01`69ff0f78
ffff9d01`69ff0f78  fffff804`8d141910 <Unloaded_HookProcessOpen.sys>+0x1910
ffff9d01`69ff0f80  fffff804`8d141280 <Unloaded_HookProcessOpen.sys>+0x1280
ffff9d01`69ff0f88  fffff804`8d141930 <Unloaded_HookProcessOpen.sys>+0x1930
ffff9d01`69ff0f90  fffff804`8d141930 <Unloaded_HookProcessOpen.sys>+0x1930
```

해당 문제는 CFG 옵션을 비활성화하면 간단하게 해결됩니다. 해당 기능은 [프로젝트 설정 - C/C++ - 코드 생성] 페이지에서 '행 가드 제어' 옵션을 통해 설정이 가능합니다.
![](/assets/posts/2023-11-14-PspProcessOpen/2.png)

프로젝트 설정 후, Re-Build 하게 되면 다음과 같이 포인터 함수가 의도한대로 호출됩니다.
![](/assets/posts/2023-11-14-PspProcessOpen/4.png)

### THIS_PTR
아래에 코드는 눈으로 봤을 때 아무런 문제가 없습니다. 하나의 지역 변수에 정적 메모리 주소를 한번 참조하고, 이를 통해 동작하는 그런 코드입니다.
```cpp
static NTSTATUS MyPspProcessOpen(OB_OPEN_REASON OpenReason, BYTE AccessMode, PEPROCESS Process, PVOID Object, ACCESS_MASK GrantedAccess, ULONG HandleCount)
{
	PMY_HOOK_TABLE hook_table = HOOK_TABLE_SIG;

	if (Process != Object)
	{
		if (OpenReason != ObCreateHandle)
		{
			if (GET_PROCESS_ID(Object) == hook_table->dwm_pid)
			{
				return STATUS_UNSUCCESSFUL;
			}
		}
	}

	// Disabled cfg(Control Flow Guard)
	return hook_table->old_OpenProcedure(OpenReason, AccessMode, Process, Object, GrantedAccess, HandleCount);
};
```
그러나 빌드하여 바이너리를 확인 시, 코드 최적화에 의해 다음과 같이 정적 포인터를 최초 1번이 아닌, 참조 될 때마다(해당 코드에서는 총 2번 실행) 참조하는 것을 확인할 수 있습니다. 이 경우, 시그니처가 전부 패치되지 않아 BSOD가 발생합니다.
![](/assets/posts/2023-11-14-PspProcessOpen/11.png)

그리하여 다음과 같이 내부에서 this_ptr을 참조하도록 코드를 구성하여 최초 1회만 정적 포인터를 참조하고, 이후에는 내부 지역 변수(r11)를 통해 추가로 참조되도록 설계하였습니다. 
![](/assets/posts/2023-11-14-PspProcessOpen/12.png)
---
title: 프로세스 보호 (PspProcessOpen)
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
```
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

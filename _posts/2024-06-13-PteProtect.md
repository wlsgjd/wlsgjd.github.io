---
title: PTE 조작을 통해 실행코드 숨기기.
categories: [Anti-Cheat]
tags: [Anti-Cheat]
---

## pte-protect
해당 글에서는 PTE 조작을 통해 <U>PAGE_EXECUTE 권한이 없는 코드를 실행</U>하고, 유저 레벨에서 메모리를 숨기는 anti-cheat bypass 기술을 리뷰합니다.

PTE의 nx 비트를 변경 시, 아래 사진과 같이 PAGE_EXECUTE 권한이 존재하지 않더라도 코드 실행이 가능합니다. VAD에는 변경 전 정보가 남아 있기 때문에 원래의 보호 속성이 표기됩니다.
![](/assets/posts/2024-06-13-PteProtect/2.png)

또한 user-supervisor를 변경하면 유저 레벨에서 페이지 엑세스가 불가능하여, 덤프 시 메모리 정보가 남지 않습니다.
![](/assets/posts/2024-06-13-PteProtect/3.png)

## PTE
PTE에는 다음과 같이 메모리에 대한 물리 주소, 실행 권한 등 다양한 정보들이 포함되어 있습니다.

해당 기술은 이 중에서 2(user-supervisor), 63(nx) 비트를 조작하여 동작합니다.
![](/assets/posts/2024-06-13-PteProtect/1.png)

user-supervisor 비트를 0으로 변경 시 유저레벨에서 엑세스가 불가능하도록 합니다. 

| Bit Position(s) |	Contents |
|---------------------|----------------|
| 2 (U/S) | User/supervisor; if 0, user-mode accesses are not allowed to the 256-TByte region controlled by this entry (see Section 4.6) |

nx 비트도 0으로 변경 시 실행이 가능한 메모리가 됩니다. PAGE_EXECUTE 권한이 존재하는 코드면 해당 비트는 0입니다.

| Bit Position(s) |	Contents |
|---------------------|----------------|
| 63 (XD) |  If IA32_EFER.NXE = 1, execute-disable (if 1, instruction fetches are not allowed from the 256-TByte region controlled by this entry; see Section 4.6); otherwise, reserved (must be 0) |

## Kernel Debugging
해당 글에서는 샘플 드라이버를 제작하지 않고, windbg 커널 디버깅을 통해 PTE를 직접 조작합니다.

다음과 같이 가상 환경에서 notepad.exe를 대상으로 테스트를 진행하였습니다.
![](/assets/posts/2024-06-13-PteProtect/4.png)

먼저, PAGE_EXECUTE 권한이 존재하지 않는 페이지를 생성하고, 실행 코드를 입력합니다.

저는 .rdata 섹션의 남는 공간에 실행 코드를 삽입하여 테스트 진행하였습니다.
```
8: kd> !process 0 0 notepad.exe
PROCESS ffffbd82177ea080
    SessionId: 1  Cid: 0464    Peb: 8b8d7fd000  ParentCid: 1688
    DirBase: 19761b000  ObjectTable: ffffa90b2e13ec40  HandleCount: 265.
    Image: notepad.exe

8: kd> .process /r /p ffffbd82177ea080

8: kd> !dh notepad.exe
SECTION HEADER #2
  .rdata name
    9288 virtual size
   26000 virtual address
    9400 size of raw data
   24A00 file pointer to raw data
       0 file pointer to relocation table
       0 file pointer to line numbers
       0 number of relocations
       0 number of line numbers
40000040 flags
         Initialized Data
         (no align specified)
         Read Only
```
```
8: kd> db notepad.exe+26000+9288
00007ff7`4540f288  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`4540f298  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`4540f2a8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`4540f2b8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`4540f2c8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`4540f2d8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`4540f2e8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`4540f2f8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................

8: kd> eb notepad.exe+26000+9288 48 b9 01 00 00 00 00 00 00 00 eb f4

8: kd> u notepad.exe+26000+9288
notepad!_NULL_IMPORT_DESCRIPTOR+0x1f90:
00007ff7`4540f288 48b90100000000000000 mov rcx,1
00007ff7`4540f292 ebf4            jmp     notepad!_NULL_IMPORT_DESCRIPTOR+0x1f90 (00007ff7`4540f288)
```

그리고 !vtop 명령을 통해 물리 메모리 및 PTE 주소를 확인합니다.
```
8: kd> !vtop 19761b000 00007ff74540f288
Amd64VtoP: Virt 00007ff74540f288, pagedir 000000019761b000
Amd64VtoP: PML4E 000000019761b7f8
Amd64VtoP: PDPE 000000018aa27ee8
Amd64VtoP: PDE 000000019ca28150
Amd64VtoP: PTE 0000000185c2a078
Amd64VtoP: Mapped phys 000000018dc4e288
Virtual address 7ff74540f288 translates to physical address 18dc4e288.

8: kd> db /p 000000018dc4e288
00000001`8dc4e288  48 b9 01 00 00 00 00 00-00 00 eb f4 00 00 00 00  H...............
00000001`8dc4e298  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000001`8dc4e2a8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000001`8dc4e2b8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000001`8dc4e2c8  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
```

### NX(XD) Bit
현재 해당 페이지는 실행 권한이 존재하지 않기 때문에 NX(63)비트가 1로 되어 있습니다.
```
8: kd> dq /p 0000000185c2a078
00000001`85c2a078  80000001`8dc4e025

8: kd> .formats 80000001`8dc4e025
Evaluate expression:
  Hex:     80000001`8dc4e025
  Decimal: -9223372030181318619
  Decimal (unsigned) : 9223372043528232997
  Octal:   1000000000061561160045
  Binary:  10000000 00000000 00000000 00000001 10001101 11000100 11100000 00100101
  Chars:   .......%
  Time:    ***** Invalid FILETIME
  Float:   low -1.21334e-030 high -1.4013e-045
  Double:  -3.29713e-314
```

해당 비트를 0으로 변경하고, 변경사항이 제대로 반영되었는지 확인합니다.
```
8: kd> eq /p 0000000185c2a078 00000001`8dc4e025
```

실행 코드가 정상적으로 입력되었고, 보호 속성 또한 ReadOnly로 표시되고 있습니다.
![](/assets/posts/2024-06-13-PteProtect/5.png)

치트엔진을 통해 스레드를 생성하고, Debug Register를 통해 코드가 실행되는지 확인하였습니다.
```
Alloc(ThreadTest, 4096)
CreateThread(ThreadTest)

ThreadTest:
mov rax, notepad.exe+2F288
call rax
ret
```

입력한 코드는 실행되고 있지만, 화면에는 ReadOnly로 표시됩니다.
![](/assets/posts/2024-06-13-PteProtect/6.png)

### User/Supervisor bit
유저 레벨에서 생성된 메모리는 User/Supervisopr 비트가 기본적으로 1로 세팅되어 있습니다.
```
5: kd> dq /p 00000000280e9078
00000000`280e9078  81000001`8dc4e025

5: kd> .formats 81000001`8dc4e025
Evaluate expression:
  Hex:     81000001`8dc4e025
  Decimal: -9151314436143390683
  Decimal (unsigned) : 9295429637566160933
  Octal:   1004000000061561160045
  Binary:  10000001 00000000 00000000 00000001 10001101 11000100 11100000 00100101
  Chars:   .......%
  Time:    ***** Invalid FILETIME
  Float:   low -1.21334e-030 high -2.35099e-038
  Double:  -7.29113e-304
```

해당 비트도 0으로 변경하고, 변경된 사항을 확인합니다.
```
8: kd> eq /p 0000000185c2a078 01000001`8dc4e021
```

유저레벨에서 페이지 엑세스가 불가능하여, 다음과 같이 덤프 수집 시에도 메모리 정보가 남지 않습니다.
![](/assets/posts/2024-06-13-PteProtect/8.png)

## Anti-Cheat Detection
추가로, 앞서 소개한 해당 방식은 현재 대부분의 안티치트에 탐지되고 있다고 합니다.

NX, User/supervisor 모두 PTE에 해당 비트가 설정되어 있는지 감지하고, try-catch 등을 통해 PAGE_EXECUTE 권한이 없는 코드를 실행했을 때 Access Violation이 발생하는지 확인한다네용.
```
Funny to see 4 years old memes are still being used by some people.

Btw, this itself is detected on most major anticheats (if not all). They can easily detect changing, for example, NX bit (at least when it comes to UM) -> just try to execute every non-executable pages in try-catch and see if it does throw access violation or not. As for supervisor meme, they does detect it by walking physical pages and look if kernel memory is mapped into UM.
```

## References
- [https://snowgyu.github.io/posts/pml4](https://snowgyu.github.io/posts/pml4)
- [https://snowgyu.github.io/posts/ConvertingAddress](https://snowgyu.github.io/posts/ConvertingAddress)
- [https://www.unknowncheats.me/forum/anti-cheat-bypass/500230-pte-protect.html](https://www.unknowncheats.me/forum/anti-cheat-bypass/500230-pte-protect.html)
- [https://github.com/Zpes/pte-protect](https://github.com/Zpes/pte-protect)
---
title: 물리 메모리 주소 변환 (vtop, pte)
categories: [Windbg]
tags: [Windbg]
---

## Converting Virtual Addresses to Physical Addresses
해당 글에서는 Windbg를 통해 가상 메모리 주소를 물리 메모리 주소로 변환하는 방법을 리뷰합니다.

물리 메모리를 확인하기 위해서는 전체 메모리 덤프 파일이 필요합니다. 다음과 같은 시스템 설정을 통해 전체 메모리를 덤프합니다.
![](/assets/posts/2024-05-23-ConvertingAddress/1.png)

## Virtual address spaces
프로세서는 메모리 위치를 읽거나 쓸 때 가상 주소를 사용합니다. 이러한 작업 중에 프로세서는 가상 주소를 물리적 주소로 변환합니다.

시스템에서 가상 주소를 사용하는 이유는 다음과 같습니다.
- 페이징 작업을 통해 메모리의 연속성을 보장한다. (물리 메모리는 연속성이 보장되지 않음)
- RAM 메모리가 부족한 경우, 디스크 파일을 통해 페이징을 대신한다.
- 가상 메모리는 프로세스마다 격리되어 있기 때문에, 직접적으로 실제 메모리를 변경할 수 없다.

![](/assets/posts/2024-05-23-ConvertingAddress/2.png)

### User space and system space
32비트 프로세스는 0x00000000~0x7FFFFFFF, 64비트 프로세스는 0x000'00000000~0x7FFF'FFFFFFFF 범위가 가상 주소 공간입니다. 해당 주소 이후부터는 전부 커널 영역에 해당합니다.

![](/assets/posts/2024-05-23-ConvertingAddress/3.png)

### Paged pool and nonpaged pool
유저 레벨에서 사용하는 모든 메모리는 디스크에 페이징 할 수 있습니다. 커널 레벨에서는 nonpaged, paged pool이라는 동적 메모리가 존재합니다. 

paged로 할당된 메모리는 필요에 따라 페이징이 가능하지만, nonpaged pool로 할당된 메모리는 디스크로 페이지 아웃이 불가능합니다. 

![](/assets/posts/2024-05-23-ConvertingAddress/4.png)

## Commands
Windbg에서는 다음과 같은 명령어를 통해 가상 메모리 주소를 물리적 메모리 주소로 변환이 가능합니다.

### !pte
먼저 다음과 같은 명령을 통해 대상 프로세스를 찾고, 컨텍스트를 전환합니다.
```
14: kd> !process 0 0 dwm.exe
PROCESS ffffdf02ec514080
    SessionId: 2  Cid: 20f4    Peb: 14a7356000  ParentCid: 2278
    DirBase: 253ef0000  ObjectTable: ffffc7899afec040  HandleCount: 2719.
    Image: dwm.exe

14: kd> .process /r /p ffffdf02ec514080
Implicit process is now ffffdf02`ec514080
Loading User Symbols
................................................................
............................

************* Symbol Loading Error Summary **************
Module name            Error
myfault                The system cannot find the file specified

You can troubleshoot most symbol related issues by turning on symbol loading diagnostics (!sym noisy) and repeating the command that caused symbols to be loaded.
You should also verify that your symbol search path (.sympath) is correct.
```

이후에 다음과 같이 명령을 입력하여 pte의 pfn(Page Frame Number)를 확인합니다. 해당 덤프 파일에서는 pfn이 0x814c3c으로 확인됩니다.
```
// !pte [VirtualAddress]
14: kd> !pte dwm.exe
                                           VA 00007ff763e90000
PXE at FFFFC562B158A7F8    PPE at FFFFC562B14FFEE8    PDE at FFFFC5629FFDD8F8    PTE at FFFFC53FFBB1F480
contains 0A000007871FC867  contains 0A000007A9EFD867  contains 0A000007917FE867  contains 8100000814C3C025
pfn 7871fc    ---DA--UWEV  pfn 7a9efd    ---DA--UWEV  pfn 7917fe    ---DA--UWEV  pfn 814c3c    ----A--UR-V
```

pfn 값에 페이지의 크기(0x1000)만큼 곱하여 해당 주소에 해당하는 물리 메모리를 확인합니다.
```
14: kd> db /p 814c3c000
00000008`14c3c000  4d 5a 90 00 03 00 00 00-04 00 00 00 ff ff 00 00  MZ..............
00000008`14c3c010  b8 00 00 00 00 00 00 00-40 00 00 00 00 00 00 00  ........@.......
00000008`14c3c020  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000008`14c3c030  00 00 00 00 00 00 00 00-00 00 00 00 00 01 00 00  ................
00000008`14c3c040  0e 1f ba 0e 00 b4 09 cd-21 b8 01 4c cd 21 54 68  ........!..L.!Th
00000008`14c3c050  69 73 20 70 72 6f 67 72-61 6d 20 63 61 6e 6e 6f  is program canno
00000008`14c3c060  74 20 62 65 20 72 75 6e-20 69 6e 20 44 4f 53 20  t be run in DOS 
00000008`14c3c070  6d 6f 64 65 2e 0d 0d 0a-24 00 00 00 00 00 00 00  mode....$.......
```

가상 주소와 비교했을 때, 메모리가 동일한 것을 확인할 수 있습니다.
```
14: kd> db dwm.exe
00007ff7`63e90000  4d 5a 90 00 03 00 00 00-04 00 00 00 ff ff 00 00  MZ..............
00007ff7`63e90010  b8 00 00 00 00 00 00 00-40 00 00 00 00 00 00 00  ........@.......
00007ff7`63e90020  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`63e90030  00 00 00 00 00 00 00 00-00 00 00 00 00 01 00 00  ................
00007ff7`63e90040  0e 1f ba 0e 00 b4 09 cd-21 b8 01 4c cd 21 54 68  ........!..L.!Th
00007ff7`63e90050  69 73 20 70 72 6f 67 72-61 6d 20 63 61 6e 6e 6f  is program canno
00007ff7`63e90060  74 20 62 65 20 72 75 6e-20 69 6e 20 44 4f 53 20  t be run in DOS 
00007ff7`63e90070  6d 6f 64 65 2e 0d 0d 0a-24 00 00 00 00 00 00 00  mode....$.......
```

### !vtop
해당 명령은 DirBase를 통해 가상 메모리 주소를 물리 주소로 변환합니다. 

전달하는 가상주소는 16진수로만 이루어진 값을 전달해야합니다.
```
14: kd> !process 0 0 dwm.exe
PROCESS ffffdf02ec514080
    SessionId: 2  Cid: 20f4    Peb: 14a7356000  ParentCid: 2278
    DirBase: 253ef0000  ObjectTable: ffffc7899afec040  HandleCount: 2719.
    Image: dwm.exe
```
```
// !vtop [DirBase] [VirtualAddress]
14: kd> !vtop 253ef0000 00007ff763e90000
Amd64VtoP: Virt 00007ff763e90000, pagedir 0000000253ef0000
Amd64VtoP: PML4E 0000000253ef07f8
Amd64VtoP: PDPE 00000007871fcee8
Amd64VtoP: PDE 00000007a9efd8f8
Amd64VtoP: PTE 00000007917fe480
Amd64VtoP: Mapped phys 0000000814c3c000
Virtual address 7ff763e90000 translates to physical address 814c3c000.
```

다음과 같이 주소 공간에 특수문자가 포함되면 정상적으로 동작하지 않습니다.
```
14: kd> !vtop 253ef0000 00007ff7`63e90000 
Amd64VtoP: Virt 0000000000007ff7, pagedir 0000000253ef0000
Amd64VtoP: PML4E 0000000253ef0000
Amd64VtoP: PDPE 00000001b5e09000
Amd64VtoP: zero PDPE
Virtual address 7ff7 translation fails, error 0xD0000147.
```

물리 메모리는 다음과 같이 확인하여, 가상 주소에 존재하는 메모리와 동일한 것을 확인할 수 있습니다.
```
14: kd> db /p 814c3c000
00000008`14c3c000  4d 5a 90 00 03 00 00 00-04 00 00 00 ff ff 00 00  MZ..............
00000008`14c3c010  b8 00 00 00 00 00 00 00-40 00 00 00 00 00 00 00  ........@.......
00000008`14c3c020  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000008`14c3c030  00 00 00 00 00 00 00 00-00 00 00 00 00 01 00 00  ................
00000008`14c3c040  0e 1f ba 0e 00 b4 09 cd-21 b8 01 4c cd 21 54 68  ........!..L.!Th
00000008`14c3c050  69 73 20 70 72 6f 67 72-61 6d 20 63 61 6e 6e 6f  is program canno
00000008`14c3c060  74 20 62 65 20 72 75 6e-20 69 6e 20 44 4f 53 20  t be run in DOS 
00000008`14c3c070  6d 6f 64 65 2e 0d 0d 0a-24 00 00 00 00 00 00 00  mode....$.......
```
```
14: kd> db 00007ff763e90000
00007ff7`63e90000  4d 5a 90 00 03 00 00 00-04 00 00 00 ff ff 00 00  MZ..............
00007ff7`63e90010  b8 00 00 00 00 00 00 00-40 00 00 00 00 00 00 00  ........@.......
00007ff7`63e90020  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ff7`63e90030  00 00 00 00 00 00 00 00-00 00 00 00 00 01 00 00  ................
00007ff7`63e90040  0e 1f ba 0e 00 b4 09 cd-21 b8 01 4c cd 21 54 68  ........!..L.!Th
00007ff7`63e90050  69 73 20 70 72 6f 67 72-61 6d 20 63 61 6e 6e 6f  is program canno
00007ff7`63e90060  74 20 62 65 20 72 75 6e-20 69 6e 20 44 4f 53 20  t be run in DOS 
00007ff7`63e90070  6d 6f 64 65 2e 0d 0d 0a-24 00 00 00 00 00 00 00  mode....$.......
```

### !ptov
해당 명령어는 특정 프로세스에 대한 모든 메모리 맵을 표시합니다. 

사용 시 다음과 같은 모든 [Physical Address]-[Virtual Address]를 표시합니다.
```
14: kd> !process 0 0 dwm.exe
PROCESS ffffdf02ec514080
    SessionId: 2  Cid: 20f4    Peb: 14a7356000  ParentCid: 2278
    DirBase: 253ef0000  ObjectTable: ffffc7899afec040  HandleCount: 2719.
    Image: dwm.exe

1: kd> !ptov 253ef0000
X86PtoV: pagedir 253ef0000, PAE enabled.
15e11000 10000
549e6000 20000
...
60a000 210000
40b000 211000
...
54ad3000 25f000
548d3000 260000
...
d71000 77510000
```

## Reference
- [https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/converting-virtual-addresses-to-physical-addresses](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/converting-virtual-addresses-to-physical-addresses)
- [https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-vtop](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-vtop)
- [https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-pte](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/-pte)
- [https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/virtual-address-spaces](https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/virtual-address-spaces)
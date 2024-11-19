---
title: Physical Memory Ranges (MmPhysicalMemoryBlock)
categories: [Windows Internals]
tags: [Windows Internals]
---

## RamMap
물리 메모리 영역에 대한 정보는 [SysInternals RAMMap](https://learn.microsoft.com/ko-kr/sysinternals/downloads/rammap)를 통해 확인할 수 있습니다.

![](/assets/posts/2024-11-20-MmPhysicalMemoryBlock/1.png)

현재 제 테스트 PC에서 사용 중인 물리 메모리 영역 정보는 다음과 같습니다.

| Start       | End           | Size        |
|-------------|---------------|-------------|
| 0x1000      | 0x9F000       | 632K        |
| 0x100000    | 0x494E0000    | 1,200,000K  |
| 0x4C8B0000  | 0x5BB74000    | 248,592K    |
| 0x5EBD2000  | 0x6A000000    | 184,504K    |
| 0x100000000 | 0x880000000   | 31,457,280K |
| Total       |               | 33,091,008K |

## MmPhysicalMemoryBlock
Windbg를 통해서도 물리 메모리 영역에 대한 정보를 확인할 수 있습니다. 

MmPhysicalMemoryBlock라는 커널 변수에 물리 메모리 영역에 대한 정보가 포함되어 있습니다.
```
19: kd> dq nt!MmPhysicalMemoryBlock L? 1
fffff801`3ed1eb10  ffff9505`376eb760 // PHYSICAL_MEMORY_DESCRIPTOR
```

해당 변수는 PHYSICAL_MEMORY_DESCRIPTOR 구조체로 구성되어 있습니다.
```
19: kd> dt_PHYSICAL_MEMORY_DESCRIPTOR
nt!_PHYSICAL_MEMORY_DESCRIPTOR
   +0x000 NumberOfRuns     : Uint4B
   +0x008 NumberOfPages    : Uint8B
   +0x010 Run              : [1] _PHYSICAL_MEMORY_RUN
```

현재 테스트 중인 PC에서 사용 중인 물리 메모리 영역은 5개 입니다.
```
19: kd> dt_PHYSICAL_MEMORY_DESCRIPTOR ffff9505`376eb760
nt!_PHYSICAL_MEMORY_DESCRIPTOR
   +0x000 NumberOfRuns     : 5
   +0x008 NumberOfPages    : 0x7e3b70
   +0x010 Run              : [1] _PHYSICAL_MEMORY_RUN
```

각 영역에 대한 물리 주소 및 크기에 대한 상세 정보는 PHYSICAL_MEMORY_RUN 필드에 배열 형태로 포함되어 있습니다.
```
19: kd> dqs ffff9505`376eb760
ffff9505`376eb760  00000000`00000005 // PHYSICAL_MEMORY_DESCRIPTOR
ffff9505`376eb768  00000000`007e3b70
ffff9505`376eb770  00000000`00000001 // PHYSICAL_MEMORY_RUN[0]
ffff9505`376eb778  00000000`0000009e
ffff9505`376eb780  00000000`00000100 // PHYSICAL_MEMORY_RUN[1]
ffff9505`376eb788  00000000`000493e0
ffff9505`376eb790  00000000`0004c8b0 // PHYSICAL_MEMORY_RUN[2]
ffff9505`376eb798  00000000`0000f2c4
ffff9505`376eb7a0  00000000`0005ebd2 // PHYSICAL_MEMORY_RUN[3]
ffff9505`376eb7a8  00000000`0000b42e
ffff9505`376eb7b0  00000000`00100000 // PHYSICAL_MEMORY_RUN[4]
ffff9505`376eb7b8  00000000`00780000
```

다음과 같은 명령을 통해 더욱 간편하게 확인할 수 있습니다.
```
19: kd> dt poi(nt!MmPhysicalMemoryBlock)  nt!_PHYSICAL_MEMORY_DESCRIPTOR -a Run[0].
   +0x010 Run     : [0] 
      +0x000 BasePage : 1
      +0x008 PageCount : 0x9e
19: kd> dt poi(nt!MmPhysicalMemoryBlock)  nt!_PHYSICAL_MEMORY_DESCRIPTOR -a Run[1].
   +0x010 Run     : [1] 
      +0x000 BasePage : 0x100
      +0x008 PageCount : 0x493e0
19: kd> dt poi(nt!MmPhysicalMemoryBlock)  nt!_PHYSICAL_MEMORY_DESCRIPTOR -a Run[2].
   +0x010 Run     : [2] 
      +0x000 BasePage : 0x4c8b0
      +0x008 PageCount : 0xf2c4
19: kd> dt poi(nt!MmPhysicalMemoryBlock)  nt!_PHYSICAL_MEMORY_DESCRIPTOR -a Run[3].
   +0x010 Run     : [3] 
      +0x000 BasePage : 0x5ebd2
      +0x008 PageCount : 0xb42e
19: kd> dt poi(nt!MmPhysicalMemoryBlock)  nt!_PHYSICAL_MEMORY_DESCRIPTOR -a Run[4].
   +0x010 Run     : [4] 
      +0x000 BasePage : 0x100000
      +0x008 PageCount : 0x780000
```

BasePage는 시작 주소, PageCount는 크기를 나타내며 수집된 덤프에서 확인된 정보는 다음과 같습니다.

| BasePage    | PageCount   |
|-------------|-------------|
| 0x1000      | 0x9e        |
| 0x100000    | 0x493e0     |
| 0x4C8B0000  | 0xf2c4      |
| 0x5EBD2000  | 0xb42e      |
| 0x100000000 | 0x780000    |
| Total       | 0x7e3b70    |

해당 정보는 [!vm](https://learn.microsoft.com/ko-kr/windows-hardware/drivers/debuggercmds/-vm) 명령을 통해 확인한 Physical Memory 내용과 일치합니다.
```
19: kd> !vm 0x01
Page File: \??\C:\pagefile.sys
  Current:  33554432 Kb  Free Space:  33554424 Kb
  Minimum:  33554432 Kb  Maximum:     62396540 Kb
Page File: \??\C:\swapfile.sys
  Current:     16384 Kb  Free Space:     16376 Kb
  Minimum:     16384 Kb  Maximum:     49636512 Kb
No Name for Paging File
  Current:  95487548 Kb  Free Space:  95487528 Kb
  Minimum:  95487548 Kb  Maximum:     95487548 Kb

*Physical Memory:          8272752 (   33091008 Kb)
Available Pages:          7278088 (   29112352 Kb)
ResAvail Pages:           7830753 (   31323012 Kb)
Locked IO Pages:                0 (          0 Kb)
Free System PTEs:      4295218714 (17180874856 Kb)
```

## IDebugDataSpaces4 (dbgeng.h)
MmPhysicalMemoryBlock 커널 변수는 IDebugDataSpaces4 인터페이스를 통해 획득 가능합니다.

ReadDebuggerData 함수를 통해 DEBUG_DATA_MmPhysicalMemoryBlockAddr 값을 전달 시 반환됩니다.

```cpp
HRESULT ReadDebuggerData(
  [in]            ULONG  Index,
  [out]           PVOID  Buffer,
  [in]            ULONG  BufferSize,
  [out, optional] PULONG DataSize
);
```

| Value                                | Return Type | Description                        |
|--------------------------------------|-------------|------------------------------------|
| DEBUG_DATA_MmPhysicalMemoryBlockAddr | ULONG64     | Returns the address of the kernel variable<br>MmPhysicalMemoryBlock.        |

## References
- [Finding physical memory ranges from a kernel debugger](https://codemachine.com/articles/physical_memory_ranges_in_kernel_debugger.html)
- [IDebugDataSpaces4::ReadDebuggerData method (dbgeng.h)](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nf-dbgeng-idebugdataspaces4-readdebuggerdata)
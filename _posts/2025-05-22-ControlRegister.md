---
title: Control Registers
date: 2025-05-23 20:22:00 +0900
categories: [Intel Architecture]
tags: [Intel Architecture]
---

## Control Registers
컨트롤 레지스터는 프로세서에 대한 상태, 정보, 설정을 나타냅니다.  
![](/assets/posts/2025-05-22-ControlRegister/1.png)

### CR0
#### PG

### CR1
현재 사용되고 있지 않은은 예약된 값입니다.

### CR2
Page Fault가 발생한 메모리 주소를 나타냅니다. 페이지 폴트는 다음과 같은 경우에 발생합니다.  

**1. Access Violation**  
아래 예제에서는 NULL 주소에 엑세스하여 Page Fault가 발생합니다.  
```cpp
int* addr_value = NULL;

// Access Violation.
*addr_value = 123;
```

**2. Protection Violation**
아래 예제에서는 READONLY 메모리에 write를 시도하여 Page Fault가 발생합니다.  
('static const' 키워드로 변수를 선언하면 .rdata 섹션에 매핑됨)  
```cpp
static const value = 0; // .rdata (PAGE_READONLY)

// Protection Violation.
value = 123;
```

### CR3
DirectoryTableBase의 물리 메모리 주소를 나타냅니다. DirBase를 통해 가상 메모리 주소를 물리 메모리 주소로 변환합니다.  
CR4.LA57 설정에 따라 PML4 또는 PML5 페이징 방식이 사용됩니다.  

해당 값은 PCB->DirectoryTableBase 값과 동일합니다.  
```
14: kd> dt_eprocess ffffdf02ec514080 -r1
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
      +0x000 Header           : _DISPATCHER_HEADER
      +0x018 ProfileListHead  : _LIST_ENTRY [ 0xffffdf02`ec514098 - 0xffffdf02`ec514098 ]
      +0x028 DirectoryTableBase : 0x00000002`53ef0000
```

### CR4
#### VMXE
#### PAE
#### LA57
```
5.5 4-LEVEL PAGING AND 5-LEVEL PAGING
Because the operation of 4-level paging and 5-level paging is very similar, they are described together in this
section. The following items highlight the distinctions between the two paging modes:
• A logical processor uses 4-level paging if CR0.PG = 1, CR4.PAE = 1, IA32_EFER.LME = 1, and CR4.LA57 = 0.
4-level paging translates 48-bit linear addresses to 52-bit physical addresses.13 Although 52 bits corresponds
to 4 PBytes, linear addresses are limited to 48 bits; at most 256 TBytes of linear-address space may be
accessed at any given time.
• A logical processor uses 5-level paging if CR0.PG = 1, CR4.PAE = 1, IA32_EFER.LME = 1, and CR4.LA57 = 1.
5-level paging translates 57-bit linear addresses to 52-bit physical addresses. Thus, 5-level paging supports a
linear-address space sufficient to access the entire physical-address space.
```

### CR8

## References
### Documents
- [Intel® 64 and IA-32 Architectures Software Developer’s Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

### Web Links
- [Intel 4-level paging](https://wlsgjd.github.io/posts/pml4)
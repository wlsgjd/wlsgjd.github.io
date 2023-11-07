---
title: 프로세스 보호 (EPROCESS Protection)
categories: [Analysis Report]
tags: [OpenProcess, EPROCESS, Protection, PspProcessOpen, ObOpenObjectByPointer, PsTestProtectedProcessIncompatibility]
---

## OpenProcess
TerminateProcess, Read/WriteProcessMemory, CreateRemoteThread 등 외부 프로세스를 제어하기 위해서는 OpenProcess를 통한 핸들 획득이 필요합니다. 운영체제에서는 사용자로부터 이러한 접근 행위를 방지하고자 커널 레벨의 보호 기능이 구현되어 있습니다. 

일부 시스템 프로세스에는 적용되어 있으며, 해킹툴에서도 이러한 기능을 이용하여 안티치트로부터 특정 프로세스를 보호합니다.

## csrss.exe
시스템 프로세스 중 csrss에는 앞서 설명한 운영체제 보호 기능이 적용되어 있습니다. 해당 프로세스를 대상으로 OpenProcess를 시도할 경우, 다음과 같이 실패하며 STATUS_ACCESS_DENIED(0xC0000022)를 반환합니다.
![](/assets/posts/2023-11-06-EProtection/1.png)

해당 값은 [MS-ERREF](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55) 문서에 나와있는데 권한으로 인한 액세스 오류입니다.

| Return value/code   |	Description |
|---------------------|----------------|
| 0xC0000022<br><br>STATUS_ACCESS_DENIED |  {Access Denied} A process has requested access to<br>an object but has not been granted those access<br>rights. |

## PspProcessOpen
유저 레벨에서 OpenProcess를 실행하게 되면 커널 내부에서 다음과 같은 순서대로 PspProcessOpen이 호출되며 해당 함수를 통해 STATUS_ACCESS_DENIED(0xC0000022)를 반환합니다.
![](/assets/posts/2023-11-06-EProtection/4.png)

| Ring | Call Stacks                    |
|:-:|-----------------------------------|
| 3 | OpenProcess                       |
| 3 | NtOpenProcess                     |
| 0 | PsOpenProcess                     |
| 0 | ObOpenObjectByPointer             |
| 0 | ObpCreateHandle                   |
| 0 | PspProcessOpen                    |

## PsTestProtectedProcessIncompatibility
PspProcessOpen 함수 내부에서는 PsTestProtectedProcessIncompatibility를 통해 대상 프로세스가 보호 중인지 체크합니다. EPROCESS + 0x87A에 위치한 데이터를 통해 보호 여부를 확인합니다.
![](/assets/posts/2023-11-06-EProtection/5.png)

## EPROCESS
해당 위치(0x87A)에는 Protection이라는 필드가 존재합니다. 해당 값에 따라 PsTestProtectedProcessIncompatibility의 반환값이 바뀌며 운영체제로부터 프로세스 보호가 적용됩니다.
```
0: kd> dt_eprocess
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : Ptr64 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY
   +0x458 RundownProtect   : _EX_RUNDOWN_REF
   +0x460 Flags2           : Uint4B
   +0x460 JobNotReallyActive : Pos 0, 1 Bit
   +0x460 AccountingFolded : Pos 1, 1 Bit
   +0x460 NewProcessReported : Pos 2, 1 Bit
   +0x460 ExitProcessReported : Pos 3, 1 Bit
   +0x460 ReportCommitChanges : Pos 4, 1 Bit
   +0x460 LastReportMemory : Pos 5, 1 Bit
   ...
   +0x87a Protection       : _PS_PROTECTION
```

csrss.exe에는 다음과 같이 Protection 필드가 0x61으로 설정되어 있습니다.
```
0: kd> dt_eprocess ffffa08f741e91c0
nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : 0x00000000`00000298 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY [ 0xffffa08f`742864c8 - 0xffffa08f`741e24c8 ]
   +0x458 RundownProtect   : _EX_RUNDOWN_REF
   ...
   +0x87a Protection       : _PS_PROTECTION

0: kd> dx -id 0,0,ffffa08f6c667040 -r1 (*((ntkrnlmp!_PS_PROTECTION *)0xffffa08f741e9a3a))
(*((ntkrnlmp!_PS_PROTECTION *)0xffffa08f741e9a3a))                 [Type: _PS_PROTECTION]
    [+0x000] Level            : 0x61 [Type: unsigned char]
    [+0x000 ( 2: 0)] Type             : 0x1 [Type: unsigned char]
    [+0x000 ( 3: 3)] Audit            : 0x0 [Type: unsigned char]
    [+0x000 ( 7: 4)] Signer           : 0x6 [Type: unsigned char]
```


## POC Code
분석된 내용을 바탕으로 프로세스 보호 도구를 개발하였습니다. 시스템 프로세스와 동일하게 Protection 필드를 0x61으로 변경하며, 전체 소스코드는 [GitHub](https://github.com/cshelldll/MyPOC/tree/main/EProtection)에 업로드 하였습니다.
```cpp
void Protect(PEPROCESS eprocess, BOOL bActive)
{
	/*
		0: kd> dt_eprocess
		ntdll!_EPROCESS
		   +0x000 Pcb              : _KPROCESS
		   +0x438 ProcessLock      : _EX_PUSH_LOCK
		   +0x440 UniqueProcessId  : Ptr64 Void
		   +0x448 ActiveProcessLinks : _LIST_ENTRY
		   +0x458 RundownProtect   : _EX_RUNDOWN_REF
		   +0x460 Flags2           : Uint4B

		   ...
		   +0x87a Protection       : _PS_PROTECTION
	*/

	BYTE* protection = (BYTE*)eprocess + 0x87A;
	*protection = bActive ? 0x61 : 0x00;
}
```
작업관리자를 통해 종료를 시도 시, 보호가 적용된 것을 확인할 수 있습니다. 
![](/assets/posts/2023-11-06-EProtection/2.png)

반대로 보호 중인 프로세스를 해제하려는 경우 해당 값을 0x00으로 변경하면 됩니다. 시스템 프로세스 또한 다음과 같이 보호가 해제되어 OpenProcess가 성공적으로 동작하였습니다.
![](/assets/posts/2023-11-06-EProtection/6.png)
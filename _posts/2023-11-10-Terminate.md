---
title: 스레드 생성 보호 (ProcessExiting)
categories: [Analysis Report]
tags: [Windows Kernel]
---

## STATUS_PROCESS_IS_TERMINATING (C000010A)
해당 핵툴은 원격 스레드 생성 시, 0xC000010A를 반환하며 실패합니다.
![](/assets/posts/2023-11-10-ProcessExiting/3.png)

해당 값은 [MS-ERREF](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55) 문서에 나와있는데 이전에 프로세스가 종료를 시도했음을 나타냅니다. 현재 종료 중이거나, 이미 종료된 상태를 의미하는 것 같습니다.

| Return value/code   |	Description |
|---------------------|----------------|
| 0xC000010A<br><br>STATUS_PROCESS_IS_TERMINATING | An attempt was made to duplicate an object handle<br>into or out of an exiting process. |

## POC Code
분석한 내용을 바탕으로 해킹툴과 동일하게 동작하는 코드를 개발하였습니다. 전체 소스코드는 [GitHub](https://github.com/cshelldll/MyPOC/tree/main/ProcessExiting)에 업로드 하였습니다. 
```cpp
void SetProcessExiting(PEPROCESS eprocess, BOOL bExiting, DWORD ExitStatus)
{
	/*
	1: kd> dt_eprocess
	ntdll!_EPROCESS
	   ++0x464 Flags            : Uint4B
	   +0x464 CreateReported   : Pos 0, 1 Bit
	   +0x464 NoDebugInherit   : Pos 1, 1 Bit
	   +0x464 ProcessExiting   : Pos 2, 1 Bit
	   +0x464 ProcessDelete    : Pos 3, 1 Bit
	   +0x464 ManageExecutableMemoryWrites : Pos 4, 1 Bit
	   +0x464 VmDeleted        : Pos 5, 1 Bit
	   +0x464 OutswapEnabled   : Pos 6, 1 Bit
	   +0x464 Outswapped       : Pos 7, 1 Bit
	   +0x464 FailFastOnCommitFail : Pos 8, 1 Bit
	   +0x464 Wow64VaSpace4Gb  : Pos 9, 1 Bit
	   +0x464 AddressSpaceInitialized : Pos 10, 2 Bits
	   +0x464 SetTimerResolution : Pos 12, 1 Bit
	   +0x464 BreakOnTermination : Pos 13, 1 Bit
	   +0x464 DeprioritizeViews : Pos 14, 1 Bit
	   +0x464 WriteWatch       : Pos 15, 1 Bit
	   +0x464 ProcessInSession : Pos 16, 1 Bit
	   +0x464 OverrideAddressSpace : Pos 17, 1 Bit
	   +0x464 HasAddressSpace  : Pos 18, 1 Bit
	   +0x464 LaunchPrefetched : Pos 19, 1 Bit
	   +0x464 Background       : Pos 20, 1 Bit
	   +0x464 VmTopDown        : Pos 21, 1 Bit
	   +0x464 ImageNotifyDone  : Pos 22, 1 Bit
	   +0x464 PdeUpdateNeeded  : Pos 23, 1 Bit
	   +0x464 VdmAllowed       : Pos 24, 1 Bit
	   +0x464 ProcessRundown   : Pos 25, 1 Bit
	   +0x464 ProcessInserted  : Pos 26, 1 Bit
	   +0x464 DefaultIoPriority : Pos 27, 3 Bits
	   +0x464 ProcessSelfDelete : Pos 30, 1 Bit
	   +0x464 SetTimerResolutionLink : Pos 31, 1 Bit

	   ...

	   // +0x7d4 ExitStatus       : Int4B
	*/
	EprocessFlags*	Flags = (EprocessFlags*)((BYTE*)eprocess + 0x464);
	EprocessFlags	org_flags = *Flags;

	DWORD* pExitStatus = (DWORD*)((BYTE*)eprocess + 0x7D4);

	if (bExiting)
	{
		Flags->ProcessExiting		= TRUE;
		Flags->ProcessDelete		= TRUE;
		Flags->ProcessInSession		= FALSE;
		Flags->ProcessRundown		= TRUE;
		Flags->ProcessSelfDelete	= TRUE;

		*pExitStatus = ExitStatus;
	}
	else
	{
		IfReverseFlag(Flags->ProcessExiting,	FALSE);
		IfReverseFlag(Flags->ProcessDelete,		FALSE);
		IfReverseFlag(Flags->ProcessInSession,	TRUE);
		IfReverseFlag(Flags->ProcessRundown,	FALSE);
		IfReverseFlag(Flags->ProcessSelfDelete, FALSE);
	}
		
	DbgPrint("flags : 0x%08X => 0x%08X\n", org_flags, *Flags);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPat)
{
	UNREFERENCED_PARAMETER(DriverObject);
	UNREFERENCED_PARAMETER(RegistryPat);

	PEPROCESS eprocess;
	HANDLE pid;
	if (FindProcessByName("notepad.exe", &eprocess))
	{
		pid = GetProcessId(eprocess);

		DbgPrint("PID : %d\n", pid);
		DbgPrint("EPROCESS : 0x%p\n", eprocess);

		SetProcessExiting(eprocess, TRUE, 0);
	}
	return STATUS_UNSUCCESSFUL;
}
```

적용 시, 프로세스가 종료되지 않습니다.
![](/assets/posts/2023-11-10-ProcessExiting/1.png)

또한 원격 스레드 생성이 실패합니다.
![](/assets/posts/2023-11-10-ProcessExiting/2.png)
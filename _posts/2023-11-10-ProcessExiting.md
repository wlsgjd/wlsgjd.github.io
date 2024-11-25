---
title: 프로세스 종료 위장 (ProcessExiting)
date: 2023-11-10 00:00:00 +/-TTTT
categories: [Analysis Report]
tags: [Windows Kernel]
---

## STATUS_PROCESS_IS_TERMINATING (C000010A)
해당 핵툴은 원격 스레드 생성 시, NtCreateThread 시스템콜 단계에서 0xC000010A를 반환하며 실패합니다. 또한 프로세스가 종료되지 않습니다.
![](/assets/posts/2023-11-10-ProcessExiting/3.png)

해당 값은 [MS-ERREF](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55) 문서에 나와있는데 이전에 프로세스가 종료를 시도했음을 나타냅니다. 현재 종료 중이거나, 이미 종료된 상태를 의미하는 것 같습니다.

| Return value/code   |	Description |
|---------------------|----------------|
| 0xC000010A<br><br>STATUS_PROCESS_IS_TERMINATING | An attempt was made to duplicate an object handle<br>into or out of an exiting process. |

## EPROCSS
해킹툴은 종료되지 않고 정상적으로 실행되고 있습니다. 이러한 오류를 반환하는 것으로 봐서는 커널 레벨에서 프로세스가 마치 종료된 것처럼 보이게 조작한 것으로 추측됩니다.

EPROCESS에는 다양한 정보들이 저장되어 있습니다. 이 중에서 flags라는 필드에는 프로세스가 종료됐는지에 대한 플래그 값이 존재합니다. 해당 값은 NtTerminateProcess를 통해 실행 중인 프로세스가 종료될 때 활성화됩니다. 
```
6: kd> dt_eprocess
ntdll!_EPROCESS
   +0x464 Flags            : Uint4B
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
```

## Kernel Memory Dump
위에서 언급한 flags 값이 변조되었는지 확인해보기 위해 해킹툴이 실행된 환경과 정상적인 환경의 덤프를 각각 수집하여 비교를 하였습니다. 프로세스 종료 상태를 나타내는 ProcessExiting, ProcessDelete, ProcessSelfDelete 등 일부 멤버들이 해킹툴 적용 시 변경되었습니다.
```
6: kd> dt_eprocess ffffb70ebe160080
ntdll!_EPROCESS
   +0x464 Flags            : 0x144d0c01 -> 0x564c0c0d
   +0x464 CreateReported   : 0y1
   +0x464 NoDebugInherit   : 0y0
   +0x464 ProcessExiting   : 0y1               // False -> True
   +0x464 ProcessDelete    : 0y1               // False -> True
   +0x464 ManageExecutableMemoryWrites : 0y0
   +0x464 VmDeleted        : 0y0
   +0x464 OutswapEnabled   : 0y0
   +0x464 Outswapped       : 0y0
   +0x464 FailFastOnCommitFail : 0y0
   +0x464 Wow64VaSpace4Gb  : 0y0
   +0x464 AddressSpaceInitialized : 0y11
   +0x464 SetTimerResolution : 0y0
   +0x464 BreakOnTermination : 0y0
   +0x464 DeprioritizeViews : 0y0
   +0x464 WriteWatch       : 0y0
   +0x464 ProcessInSession : 0y0               // True -> False
   +0x464 OverrideAddressSpace : 0y0
   +0x464 HasAddressSpace  : 0y1
   +0x464 LaunchPrefetched : 0y1
   +0x464 Background       : 0y0
   +0x464 VmTopDown        : 0y0
   +0x464 ImageNotifyDone  : 0y1
   +0x464 PdeUpdateNeeded  : 0y0
   +0x464 VdmAllowed       : 0y0
   +0x464 ProcessRundown   : 0y1               // False -> True
   +0x464 ProcessInserted  : 0y1
   +0x464 DefaultIoPriority : 0y010
   +0x464 ProcessSelfDelete : 0y1              // False -> True
   +0x464 SetTimerResolutionLink : 0y0
```

또한 프로세스 종료 시 반환되는 리턴 값(ExitCode) 또한 변경되어 있습니다. 해당 값은 프로세스가 실행 중인 경우 STILL_ACTIVE(0x103), 아닌 경우에는 [TerminateProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminateprocess) 호출 시 전달된 uExitCode 값입니다.
```cpp
BOOL TerminateProcess(
  [in] HANDLE hProcess,
  [in] UINT   uExitCode
);
```
```
6: kd> dt_eprocess ffffb70ebe160080
ntdll!_EPROCESS
   +0x7d4 ExitStatus       : 0n1193046          // 0n259 -> 0n1193046
```

## NtTerminateProcess
유저 레벨에서 ExitProcess를 호출하는 경우, 커널 내부에서 NtTerminateProcess가 호출됩니다. 

해당 함수에서는 다음과 같이 flags와 ExitStatus 값을 초기화하여 프로세스가 종료 중이라는 것을 나타냅니다. 
![](/assets/posts/2023-11-10-ProcessExiting/4.png)

ProcessDelete, ProcessSelfDelete이 활성화되면 PspTerminateAllThreads를 통해 실행 중인 모든 스레드를 종료합니다. 해당 함수의 호출 스택은 다음과 같습니다.

| Ring | Call Stacks                    |
|:-:|-----------------------------------|
| 3 | ExitProcess                       |
| 3 | RtlExitUserProcess                |
| 3 | NtTerminateProcess                |
| 0 | NtTerminateProcess                |
| 0 | PspTerminateAllThreads            |

해당 해킹툴은 이 과정에서 초기화되는 flags와 동일하게 값을 변조하여 마치 프로세스가 중료 중 또는 이미 종료된 것처럼 보이게 조작하였습니다. 

그렇기 때문에 당연히 종료된 프로세스에는 스레드 생성이 불가능한 것으로 보입니다.

## POC Code
분석한 내용을 바탕으로 해킹툴과 동일하게 동작하는 코드를 개발하였습니다. 전체 소스코드는 [GitHub](https://github.com/wlsgjd/MyPOC/tree/main/ProcessExiting)에 업로드 하였습니다. 
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

적용 시 프로세스가 종료되지 않습니다.
![](/assets/posts/2023-11-10-ProcessExiting/1.png)

또한 원격 스레드 생성이 실패합니다.
![](/assets/posts/2023-11-10-ProcessExiting/2.png)

## NtQueryInformationProcess
해당 해킹툴을 유저 레벨에서 탐지할 방법이 없는지 연구를 진행하였습니다. 유저 레벨에서 EPROCESS를 읽는 대표적인 방법은 [NtQueryInformationProcess](https://learn.microsoft.com/ko-kr/windows/win32/api/winternl/nf-winternl-ntqueryinformationprocess)가 존재합니다. 
```cpp
__kernel_entry NTSTATUS NtQueryInformationProcess(
  [in]            HANDLE           ProcessHandle,
  [in]            PROCESSINFOCLASS ProcessInformationClass,
  [out]           PVOID            ProcessInformation,
  [in]            ULONG            ProcessInformationLength,
  [out, optional] PULONG           ReturnLength
);
```

해당 함수에서 flags(+0x464)를 반환하는 유형이 존재하는지 분석해봤습니다.
![](/assets/posts/2023-11-10-ProcessExiting/5.png)

호출 시 전달하는 ProcessInformationClass 값에 따라 반환되는 유형이 다르며, 해당 함수를 통해 확인가능한 flags 멤버는 다음과 같습니다. 

|Code|   ProcessInformationClass | Return Flags                        |
|:-:|----------------------------|-------------------------------------|
|0 | ProcessBasicInformation     | ProcessSelfDelete<br>ProcessDelete  |
|29| ProcessIoPriority           | DefaultIoPriority                   |
|33| ProcessBreakOnTermination   | BreakOnTermination                  |
|31| ProcessDebugFlags           | NoDebugInherit                      |
|46| ProcessMemoryAllocationMode | VmTopDown                           |

## PROCESS_EXTENDED_BASIC_INFORMATION
ProcessBasicInformation는 문서화되지 않은 PROCESS_EXTENDED_BASIC_INFORMATION이라는 확장 형태가 존재합니다. 

다음과 같이 전달받은 ProcessInformationLength에 따라 다르게 동작하며 EXTENDED 버전은 더 많은 정보를 제공합니다.
![](/assets/posts/2023-11-10-ProcessExiting/6.png)

ProcessSelfDelete, ProcessDelete에 대한 값 또한 PROCESS_EXTENDED_BASIC_INFORMATION를 통해서 확인이 가능합니다. 다음과 같이 해당 값이 활성화된 경우 0x04 값을 반환합니다.
![](/assets/posts/2023-11-10-ProcessExiting/7.png)

이를 활용하여 flags롤 조작한 핵툴을 유저 레벨에서 탐지할 수 있습니다. 전체 소스코드는 [GitHub](https://github.com/wlsgjd/MyPOC/tree/main/ExitingDetector)에 업로드하였습니다. 해당 프로그램은 x64를 기준으로 제작되었기 때문에 x86 파일에 대해서는 따로 포팅 작업이 필요합니다.
```cpp
typedef struct _PROCESS_EXTENDED_BASIC_INFORMATION
{
	SIZE_T Size;
	PROCESS_BASIC_INFORMATION BasicInfo;
	union
	{
		ULONG Flags;
		struct
		{
			ULONG IsProtectedProcess : 1;
			ULONG IsWow64Process : 1;
			ULONG IsProcessDeleting : 1;
			ULONG IsCrossSessionCreate : 1;
			ULONG IsFrozen : 1;
			ULONG IsBackground : 1;
			ULONG IsStronglyNamed : 1;
			ULONG IsSecureProcess : 1;
			ULONG IsSubsystemProcess : 1;
			ULONG SpareBits : 23;
		} DUMMYSTRUCTNAME;
	} DUMMYUNIONNAME;
} PROCESS_EXTENDED_BASIC_INFORMATION, *PPROCESS_EXTENDED_BASIC_INFORMATION;

int main()
{
    HANDLE hProcess = OpenProcess(PROCESS_QUERY_INFORMATION, false, GetProcessIdByName(_T("notepad.exe")));
    NtQueryInformationProcessT NtQueryInformationProcess = (NtQueryInformationProcessT)GetProcAddress(LoadLibraryA("ntdll.dll"), "NtQueryInformationProcess");

    PROCESS_EXTENDED_BASIC_INFORMATION info = { 0x00, };
    ULONG len;
    NtQueryInformationProcess(hProcess, ProcessBasicInformation, &info, sizeof(info), &len);

    if (info.IsProcessDeleting)
    {
        printf("ProcessSelfDelete, ProcessDelete : True\n");

        DWORD exitcode = 0;
        GetExitCodeProcess(hProcess, &exitcode);
        if (exitcode == STILL_ACTIVE)
        {
            printf("detected !!\n");
        }
    }

    ULONG mode = 0;
    NtQueryInformationProcess(hProcess, ProcessMemoryAllocationMode, &mode, sizeof(mode), &len);
    if (mode)
    {
        printf("VmTopMode : True\n");
    }

    printf("success. \n");
    system("pause");
}
```

다음과 같이 flags를 조작하여 보호된 프로세스를 정상적으로 탐지합니다.
![](/assets/posts/2023-11-10-ProcessExiting/8.png)
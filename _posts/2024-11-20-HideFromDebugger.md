---
title: HideFromDebugger
date: 2024-11-20 00:00:00 +/-TTTT
categories: [Anti-Debugging]
tags: [Anti-Debugging]
---

## STATUS_SINGLE_STEP
최근 진행했던 과제에서 디버깅 시 프로세스가 종료되는 Anti-Debugging 기법을 확인하였습니다. 

당시에 Debug Register를 통해 디버깅을 시도하였습니다. DebugAcitveProcess를 통해 Debug Object 또한 정상적으로 생성되는 부분을 확인하였고, 디버깅 과정에서 문제될 요소가 없었습니다.
![](/assets/posts/2024-11-20-HideFromDebugger/2.png)

그러나, 중단점이 설치된 코드가 실행되면 프로세스가 종료되었습니다. 이벤트 뷰어를 통해 확인해보니 [STATUS_SINGLE_STEP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55)를 반환하며 종료되었습니다. 
![](/assets/posts/2024-11-20-HideFromDebugger/1.png)


중단점이 설치된 코드를 실행하는 과정에서 예외가 발생한 것으로 보입니다.

| Return value/code   |	Description |
|---------------------|----------------|
| 0x80000004<br><br>STATUS_SINGLE_STEP |  {EXCEPTION} Single Step A single step or trace operation has just been completed. |

## HideFromDebugger
해당 기법은 모든 스레드를 대상으로 동작하지는 않았습니다. 임의로 생성된 스레드를 디버깅 할 때는 BP가 정상적으로 동작하였습니다.
![](/assets/posts/2024-11-20-HideFromDebugger/3.png)

따라서, 특정 스레드에만 보호가 적용된 것을 인지하였고, 이와 관련하여 TEB 및 EThread를 먼저 확인해봤습니다. (VEH, SEH와 같은 Exception Handler는 디버거가 연결된 경우, 디버거에 제어권이 우선적으로 넘어가기에 추가로 확인하지 않습니다.)

TEB에는 디버깅 관련된 데이터는 일부 존재하나, 안티디버깅과 관련된 기능들은 제공되지 않습니다.
<br>또한, 다른 정상 스레드와 비교해봐도 변경점이 존재하지 않았습니다.
```
0:000> dt_teb
ntdll!_TEB
   +0x000 NtTib            : _NT_TIB
   +0x01c EnvironmentPointer : Ptr32 Void
   +0x020 ClientId         : _CLIENT_ID
   +0x028 ActiveRpcHandle  : Ptr32 Void
   +0x02c ThreadLocalStoragePointer : Ptr32 Void
   +0x030 ProcessEnvironmentBlock : Ptr32 _PEB
   +0x034 LastErrorValue   : Uint4B
   +0x038 CountOfOwnedCriticalSections : Uint4B
   +0x03c CsrClientThread  : Ptr32 Void
   +0x040 Win32ThreadInfo  : Ptr32 Void
   +0x044 User32Reserved   : [26] Uint4B
   +0x0ac UserReserved     : [5] Uint4B
   +0x0c0 WOW32Reserved    : Ptr32 Void
   +0x0c4 CurrentLocale    : Uint4B
   +0x0c8 FpSoftwareStatusRegister : Uint4B
   +0x0cc ReservedForDebuggerInstrumentation : [16] Ptr32 Void
   +0x10c SystemReserved1  : [26] Ptr32 Void
   +0x174 PlaceholderCompatibilityMode : Char
   +0x175 PlaceholderHydrationAlwaysExplicit : UChar
   +0x176 PlaceholderReserved : [10] Char
   +0x180 ProxiedProcessId : Uint4B
   +0x184 _ActivationStack : _ACTIVATION_CONTEXT_STACK
   +0x19c WorkingOnBehalfTicket : [8] UChar
   +0x1a4 ExceptionCode    : Int4B
   +0x1a8 ActivationContextStackPointer : Ptr32 _ACTIVATION_CONTEXT_STACK
   +0x1ac InstrumentationCallbackSp : Uint4B
   +0x1b0 InstrumentationCallbackPreviousPc : Uint4B
   +0x1b4 InstrumentationCallbackPreviousSp : Uint4B
   +0x1b8 InstrumentationCallbackDisabled : UChar
   +0x1b9 SpareBytes       : [23] UChar
   +0x1d0 TxFsContext      : Uint4B
   +0x1d4 GdiTebBatch      : _GDI_TEB_BATCH
   +0x6b4 RealClientId     : _CLIENT_ID
   +0x6bc GdiCachedProcessHandle : Ptr32 Void
   +0x6c0 GdiClientPID     : Uint4B
   +0x6c4 GdiClientTID     : Uint4B
   +0x6c8 GdiThreadLocalInfo : Ptr32 Void
   +0x6cc Win32ClientInfo  : [62] Uint4B
   +0x7c4 glDispatchTable  : [233] Ptr32 Void
   +0xb68 glReserved1      : [29] Uint4B
   +0xbdc glReserved2      : Ptr32 Void
   +0xbe0 glSectionInfo    : Ptr32 Void
   +0xbe4 glSection        : Ptr32 Void
   +0xbe8 glTable          : Ptr32 Void
   +0xbec glCurrentRC      : Ptr32 Void
   +0xbf0 glContext        : Ptr32 Void
   +0xbf4 LastStatusValue  : Uint4B
   +0xbf8 StaticUnicodeString : _UNICODE_STRING
   +0xc00 StaticUnicodeBuffer : [261] Wchar
   +0xe0c DeallocationStack : Ptr32 Void
   +0xe10 TlsSlots         : [64] Ptr32 Void
   +0xf10 TlsLinks         : _LIST_ENTRY
   +0xf18 Vdm              : Ptr32 Void
   +0xf1c ReservedForNtRpc : Ptr32 Void
   +0xf20 DbgSsReserved    : [2] Ptr32 Void
   +0xf28 HardErrorMode    : Uint4B
   +0xf2c Instrumentation  : [9] Ptr32 Void
   +0xf50 ActivityId       : _GUID
   +0xf60 SubProcessTag    : Ptr32 Void
   +0xf64 PerflibData      : Ptr32 Void
   +0xf68 EtwTraceData     : Ptr32 Void
   +0xf6c WinSockData      : Ptr32 Void
   +0xf70 GdiBatchCount    : Uint4B
   +0xf74 CurrentIdealProcessor : _PROCESSOR_NUMBER
   +0xf74 IdealProcessorValue : Uint4B
   +0xf74 ReservedPad0     : UChar
   +0xf75 ReservedPad1     : UChar
   +0xf76 ReservedPad2     : UChar
   +0xf77 IdealProcessor   : UChar
   +0xf78 GuaranteedStackBytes : Uint4B
   +0xf7c ReservedForPerf  : Ptr32 Void
   +0xf80 ReservedForOle   : Ptr32 Void
   +0xf84 WaitingOnLoaderLock : Uint4B
   +0xf88 SavedPriorityState : Ptr32 Void
   +0xf8c ReservedForCodeCoverage : Uint4B
   +0xf90 ThreadPoolData   : Ptr32 Void
   +0xf94 TlsExpansionSlots : Ptr32 Ptr32 Void
   +0xf98 MuiGeneration    : Uint4B
   +0xf9c IsImpersonating  : Uint4B
   +0xfa0 NlsCache         : Ptr32 Void
   +0xfa4 pShimData        : Ptr32 Void
   +0xfa8 HeapData         : Uint4B
   +0xfac CurrentTransactionHandle : Ptr32 Void
   +0xfb0 ActiveFrame      : Ptr32 _TEB_ACTIVE_FRAME
   +0xfb4 FlsData          : Ptr32 Void
   +0xfb8 PreferredLanguages : Ptr32 Void
   +0xfbc UserPrefLanguages : Ptr32 Void
   +0xfc0 MergedPrefLanguages : Ptr32 Void
   +0xfc4 MuiImpersonation : Uint4B
   +0xfc8 CrossTebFlags    : Uint2B
   +0xfc8 SpareCrossTebBits : Pos 0, 16 Bits
   +0xfca SameTebFlags     : Uint2B
   +0xfca SafeThunkCall    : Pos 0, 1 Bit
   +0xfca InDebugPrint     : Pos 1, 1 Bit
   +0xfca HasFiberData     : Pos 2, 1 Bit
   +0xfca SkipThreadAttach : Pos 3, 1 Bit
   +0xfca WerInShipAssertCode : Pos 4, 1 Bit
   +0xfca RanProcessInit   : Pos 5, 1 Bit
   +0xfca ClonedThread     : Pos 6, 1 Bit
   +0xfca SuppressDebugMsg : Pos 7, 1 Bit
   +0xfca DisableUserStackWalk : Pos 8, 1 Bit
   +0xfca RtlExceptionAttached : Pos 9, 1 Bit
   +0xfca InitialThread    : Pos 10, 1 Bit
   +0xfca SessionAware     : Pos 11, 1 Bit
   +0xfca LoadOwner        : Pos 12, 1 Bit
   +0xfca LoaderWorker     : Pos 13, 1 Bit
   +0xfca SkipLoaderInit   : Pos 14, 1 Bit
   +0xfca SkipFileAPIBrokering : Pos 15, 1 Bit
   +0xfcc TxnScopeEnterCallback : Ptr32 Void
   +0xfd0 TxnScopeExitCallback : Ptr32 Void
   +0xfd4 TxnScopeContext  : Ptr32 Void
   +0xfd8 LockCount        : Uint4B
   +0xfdc WowTebOffset     : Int4B
   +0xfe0 ResourceRetValue : Ptr32 Void
   +0xfe4 ReservedForWdf   : Ptr32 Void
   +0xfe8 ReservedForCrt   : Uint8B
   +0xff0 EffectiveContainerId : _GUID
   +0x1000 LastSleepCounter : Uint8B
   +0x1008 SpinCallCount    : Uint4B
   +0x1010 ExtendedFeatureDisableMask : Uint8B
```

EThread 에서는 HideFromDebugger라는 디버깅과 관련된 것으로 보이는 필드가 존재하여 확인해보니 0x01으로 활성화 되어 있었습니다.
<br>임의로 생성된 스레드에서는 해당 기능이 비활성화 되어 있으며, 보호가 적용된 스레드에만 HideFromDebugger가 활성화 되어 있었습니다.
![](/assets/posts/2024-11-20-HideFromDebugger/4.png)

### NtSetInformationThread
해당 기능에 대하여 조사해보니 HideFromDebugger는 NtSetInformationThread를 통해 호출가능한 안티디버깅 기능으로 확인되었습니다.
<br>문서화되지 않은 값, ThreadHideFromDebugger(0x11)를 전달하여 실행합니다.

다음과 같이 실행하면 현재 스레드가 보호됩니다.
```cpp
#define THREAD_HIDE_FROM_DEBUGGER 0x11

typedef NTSTATUS(*PNTSETINFORMATIONTHREAD)(
    IN HANDLE ThreadHandle,
    IN _THREAD_INFORMATION_CLASS ThreadInformationClass,
    IN PVOID ThreadInformation,
    IN ULONG ThreadInformationLength
    );

void main() {
    PNTSETINFORMATIONTHREAD pNtSetInformationThread;
    HMODULE h_ntdll = GetModuleHandleA("ntdll.dll");
    pNtSetInformationThread = (PNTSETINFORMATIONTHREAD)GetProcAddress(h_ntdll, "NtSetInformationThread");
    
    pNtSetInformationThread(GetCurrentThread(), (_THREAD_INFORMATION_CLASS)THREAD_HIDE_FROM_DEBUGGER, NULL, NULL);

    printf("Done!\n");
    system("pause");

}
```

해당 함수는 단방향성으로 동작하기 때문에 반대로 비활성화하는 것은 불가능합니다.
<br>따라서, 해당 안티디버깅을 우회하려면 NtSetInformationThread가 호출되기 전에 가로채거나, 커널 레벨에서 HideFromDebugger 값을 직접 조작해야합니다.
![](/assets/posts/2024-11-20-HideFromDebugger/5.png)

### DbgkForwardException
Debug Register가 활성화되면 커널 내부에서는 IDT를 통해 0x01번에 해당하는 KiDebugTrapOrFault가 호출됩니다.
![](/assets/posts/2024-11-20-HideFromDebugger/7.png)

이후에 KiDispatchException, DbgkForwardException가 순서대로 호출되며, HideFromDebugger 안티디버깅 기능은 DbgkForwardException 내에 분기문을 통해 실행됩니다.
<br>해당 값이 활성화되어 있는 경우, 프로세스가 즉시 종료됩니다.
![](/assets/posts/2024-11-20-HideFromDebugger/6.png)

## Proof of Concept
HideFromDebugger를 비활성화 하는 도구를 개발하였습니다. 실행 중인 모든 스레드의 HideFromDebugger 값을 0으로 변경합니다.
<br>해당 기능을 비활성화 시 디버깅이 정상적으로 동작하였으며, 전체 소스코드는 [GitHub](https://github.com/wlsgjd/MyPOC/tree/main/UnhideDebugger)에 업로드하였습니다.
```c
BOOL UnhideFromDebugger(PEPROCESS eprocess)
{
	// EPROCESS->ThreadListHead
	PLIST_ENTRY thread_list_head = (PVOID)((uintptr_t)eprocess + 0x5e0); 
	PLIST_ENTRY cur_list = thread_list_head;

	DbgPrint("UnhideFromDebugger\n");

	do
	{
		// ETHREAD->ThreadListEntry
		PETHREAD ethread = (PETHREAD)((uintptr_t)cur_list - 0x538);

		// ETHREAD->CrossThreadFlags
		CrossThreadFlags* flags = (CrossThreadFlags*)((uintptr_t)ethread + 0x560);

		if (flags->HideFromDebugger)
		{
			DbgPrint("- 0x%p: TRUE -> FALSE\n", ethread);
			flags->HideFromDebugger = FALSE;
		}

		cur_list = cur_list->Blink;
	} while (cur_list != thread_list_head);

	return TRUE;
}
```

## References
- [NtSetInformationThread를 이용한 Anti-Debugging](https://ezbeat.tistory.com/371)
- [Hidding Threads From Debuggers](https://waleedassar.blogspot.com/2012/11/hidding-threads-from-debuggers.html)
- [Hardware breakpoints and exceptions on Windows](https://ling.re/hardware-breakpoints/)
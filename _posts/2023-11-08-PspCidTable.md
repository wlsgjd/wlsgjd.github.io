---
title: 프로세스 보호 (PspCidTable)
categories: [Analysis Report]
tags: [PsLookupProcessByProcessId, PspReferenceCidTableEntry, ExpLookupHandleTableEntry, PspCidTable, EPROCESS, OpenProcess]
---

## STATUS_INVALID_CID (0xC000000B)
OpenProcess를 호출하면 다음과 같이 0xC000000B를 반환하며 실패하는 해킹툴이 발견되었습니다.
![](/assets/posts/2023-11-08-PspCidTable/1.png)

다음과 같이 NtOpenProcess syscall 이후에 실패된 것으로 보아, 커널 레벨에서 조작된 것을 추측할 수 있습니다.
![](/assets/posts/2023-11-08-PspCidTable/2.png)

해당 값은 [MS-ERREF](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55) 문서에 나와있는데 클라이언트 ID가 잘못되었음을 나타냅니다. 해당 내용만 봐서는 딱히 짐작이 되질 않습니다.

| Return value/code   |	Description |
|---------------------|----------------|
| 0xC000000B<br><br>STATUS_INVALID_CID |  An invalid client ID was specified. |

## PsLookupProcessByProcessId
유저 레벨에서 OpenProcess를 호출할 경우, 커널 내부에서 PsLookupProcessByProcessId를 호출하게 됩니다. 해당 함수에서는 실패 시 STATUS_INVALID_CID를 리턴하고 있습니다.
![](/assets/posts/2023-11-08-PspCidTable/3.png)

호출 스택은 다음과 같이 정리할 수 있습니다.

| Ring | Call Stacks                    |
|:-:|-----------------------------------|
| 3 | OpenProcess                       |
| 3 | NtOpenProcess                     |
| 0 | PsOpenProcess                     |
| 0 | *PsLookupProcessByProcessId       |
| 0 | PspReferenceCidTableEntry         |
| 0 | ExpLookupHandleTableEntry         |

내부에서 사용되는 함수들은 다음과 같은 호출 구조를 가진 것으로 추측합니다. 문서화되지 않은 부분이라 정확하진 않습니다.
```cpp
BOOL PspReferenceCidTableEntry(ULONG pid, PEPROCESS* process);
void* ExpLookupHandleTableEntry(void* PspCidTable, ULONG pid);
```

PsLookupProcessByProcessId 내부에서는 PspReferenceCidTableEntry라는 함수를 호출하며 PID 및 리턴받을 EPROCESS 포인터를 전달합니다.

## ExpLookupHandleTableEntry
ExpLookupHandleTableEntry 함수는 PspCidTable이라는 프로세스 오브젝트 테이블을 전달 받으며, 특정한 연산을 통해 PID를 기반으로 인덱스를 생성하고 데이터를 반환합니다. 해시 테이블과 비슷한 개념이라고 생각하면 될 것 같습니다.
![](/assets/posts/2023-11-08-PspCidTable/4.png)

일반적인 경우에 내부 연산은 다음과 같습니다.
```cpp
psp
```


## PspReferenceCidTableEntry
PspReferenceCidTableEntry는 ExpLookupHandleTableEntry에서 전달받은 값을 한번 더 연산작업을 통해 전달받는 EPROCESS 값으로 변환합니다.

## Kernel Memory Dump
위에서 분석된 내용들을 증명하기 위해 해킹툴이 적용된 환경에서 커널 메모리 덤프를 수집하여 추가로 진행하였습니다.


## Windbg Script

## POC Code
분석된 내용을 바탕으로 PspCidTable를 조작하는 도구를 개발하였습니다. 전체 소스코드는 [GitHub](https://github.com/cshelldll/MyPOC/tree/main/PCTSample)에 업로드 하였습니다.
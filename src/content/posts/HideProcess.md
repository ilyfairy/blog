---
title: "驱动隐藏进程(断链)"
published: 2022-03-24T17:02:39+08:00
draft: false
tags: ["CSharp", "EFCore", "PostgreSql", "教程"]
category: '教程'
---


# 驱动隐藏进程

## 开始

在Windows中, 进程之间都是由一个个链表链起来的 每一个进程都是一个EPROCESS结构  
以下是 `Windows10 19041 x64` 的eprocess结构  

``` asm
#win10
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
   +0x460 ForceWakeCharge  : Pos 6, 1 Bit
   +0x460 CrossSessionCreate : Pos 7, 1 Bit
   +0x460 NeedsHandleRundown : Pos 8, 1 Bit
   +0x460 RefTraceEnabled  : Pos 9, 1 Bit
   +0x460 PicoCreated      : Pos 10, 1 Bit
   +0x460 EmptyJobEvaluated : Pos 11, 1 Bit
   +0x460 DefaultPagePriority : Pos 12, 3 Bits
   +0x460 PrimaryTokenFrozen : Pos 15, 1 Bit
   +0x460 ProcessVerifierTarget : Pos 16, 1 Bit
   +0x460 RestrictSetThreadContext : Pos 17, 1 Bit
   +0x460 AffinityPermanent : Pos 18, 1 Bit
   +0x460 AffinityUpdateEnable : Pos 19, 1 Bit
   +0x460 PropagateNode    : Pos 20, 1 Bit
   +0x460 ExplicitAffinity : Pos 21, 1 Bit
   +0x460 ProcessExecutionState : Pos 22, 2 Bits
   +0x460 EnableReadVmLogging : Pos 24, 1 Bit
   +0x460 EnableWriteVmLogging : Pos 25, 1 Bit
   +0x460 FatalAccessTerminationRequested : Pos 26, 1 Bit
   +0x460 DisableSystemAllowedCpuSet : Pos 27, 1 Bit
   +0x460 ProcessStateChangeRequest : Pos 28, 2 Bits
   +0x460 ProcessStateChangeInProgress : Pos 30, 1 Bit
   +0x460 InPrivate        : Pos 31, 1 Bit
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
   +0x468 CreateTime       : _LARGE_INTEGER
   +0x470 ProcessQuotaUsage : [2] Uint8B
   +0x480 ProcessQuotaPeak : [2] Uint8B
   +0x490 PeakVirtualSize  : Uint8B
   +0x498 VirtualSize      : Uint8B
   +0x4a0 SessionProcessLinks : _LIST_ENTRY
   +0x4b0 ExceptionPortData : Ptr64 Void
   +0x4b0 ExceptionPortValue : Uint8B
   +0x4b0 ExceptionPortState : Pos 0, 3 Bits
   +0x4b8 Token            : _EX_FAST_REF
   +0x4c0 MmReserved       : Uint8B
   +0x4c8 AddressCreationLock : _EX_PUSH_LOCK
   +0x4d0 PageTableCommitmentLock : _EX_PUSH_LOCK
   +0x4d8 RotateInProgress : Ptr64 _ETHREAD
   +0x4e0 ForkInProgress   : Ptr64 _ETHREAD
   +0x4e8 CommitChargeJob  : Ptr64 _EJOB
   +0x4f0 CloneRoot        : _RTL_AVL_TREE
   +0x4f8 NumberOfPrivatePages : Uint8B
   +0x500 NumberOfLockedPages : Uint8B
   +0x508 Win32Process     : Ptr64 Void
   +0x510 Job              : Ptr64 _EJOB
   +0x518 SectionObject    : Ptr64 Void
   +0x520 SectionBaseAddress : Ptr64 Void
   +0x528 Cookie           : Uint4B
   +0x530 WorkingSetWatch  : Ptr64 _PAGEFAULT_HISTORY
   +0x538 Win32WindowStation : Ptr64 Void
   +0x540 InheritedFromUniqueProcessId : Ptr64 Void
   +0x548 OwnerProcessId   : Uint8B
   +0x550 Peb              : Ptr64 _PEB
   +0x558 Session          : Ptr64 _MM_SESSION_SPACE
   +0x560 Spare1           : Ptr64 Void
   +0x568 QuotaBlock       : Ptr64 _EPROCESS_QUOTA_BLOCK
   +0x570 ObjectTable      : Ptr64 _HANDLE_TABLE
   +0x578 DebugPort        : Ptr64 Void
   +0x580 WoW64Process     : Ptr64 _EWOW64PROCESS
   +0x588 DeviceMap        : Ptr64 Void
   +0x590 EtwDataSource    : Ptr64 Void
   +0x598 PageDirectoryPte : Uint8B
   +0x5a0 ImageFilePointer : Ptr64 _FILE_OBJECT
   +0x5a8 ImageFileName    : [15] UChar
   +0x5b7 PriorityClass    : UChar
   +0x5b8 SecurityPort     : Ptr64 Void
   +0x5c0 SeAuditProcessCreationInfo : _SE_AUDIT_PROCESS_CREATION_INFO
   +0x5c8 JobLinks         : _LIST_ENTRY
   +0x5d8 HighestUserAddress : Ptr64 Void
   +0x5e0 ThreadListHead   : _LIST_ENTRY
   +0x5f0 ActiveThreads    : Uint4B
   +0x5f4 ImagePathHash    : Uint4B
   +0x5f8 DefaultHardErrorProcessing : Uint4B
   +0x5fc LastThreadExitStatus : Int4B
   +0x600 PrefetchTrace    : _EX_FAST_REF
   +0x608 LockedPagesList  : Ptr64 Void
   +0x610 ReadOperationCount : _LARGE_INTEGER
   +0x618 WriteOperationCount : _LARGE_INTEGER
   +0x620 OtherOperationCount : _LARGE_INTEGER
   +0x628 ReadTransferCount : _LARGE_INTEGER
   +0x630 WriteTransferCount : _LARGE_INTEGER
   +0x638 OtherTransferCount : _LARGE_INTEGER
   +0x640 CommitChargeLimit : Uint8B
   +0x648 CommitCharge     : Uint8B
   +0x650 CommitChargePeak : Uint8B
   +0x680 Vm               : _MMSUPPORT_FULL
   +0x7c0 MmProcessLinks   : _LIST_ENTRY
   +0x7d0 ModifiedPageCount : Uint4B
   +0x7d4 ExitStatus       : Int4B
   +0x7d8 VadRoot          : _RTL_AVL_TREE
   +0x7e0 VadHint          : Ptr64 Void
   +0x7e8 VadCount         : Uint8B
   +0x7f0 VadPhysicalPages : Uint8B
   +0x7f8 VadPhysicalPagesLimit : Uint8B
   +0x800 AlpcContext      : _ALPC_PROCESS_CONTEXT
   +0x820 TimerResolutionLink : _LIST_ENTRY
   +0x830 TimerResolutionStackRecord : Ptr64 _PO_DIAG_STACK_RECORD
   +0x838 RequestedTimerResolution : Uint4B
   +0x83c SmallestTimerResolution : Uint4B
   +0x840 ExitTime         : _LARGE_INTEGER
   +0x848 InvertedFunctionTable : Ptr64 _INVERTED_FUNCTION_TABLE
   +0x850 InvertedFunctionTableLock : _EX_PUSH_LOCK
   +0x858 ActiveThreadsHighWatermark : Uint4B
   +0x85c LargePrivateVadCount : Uint4B
   +0x860 ThreadListLock   : _EX_PUSH_LOCK
   +0x868 WnfContext       : Ptr64 Void
   +0x870 ServerSilo       : Ptr64 _EJOB
   +0x878 SignatureLevel   : UChar
   +0x879 SectionSignatureLevel : UChar
   +0x87a Protection       : _PS_PROTECTION
   +0x87b HangCount        : Pos 0, 3 Bits
   +0x87b GhostCount       : Pos 3, 3 Bits
   +0x87b PrefilterException : Pos 6, 1 Bit
   +0x87c Flags3           : Uint4B
   +0x87c Minimal          : Pos 0, 1 Bit
   +0x87c ReplacingPageRoot : Pos 1, 1 Bit
   +0x87c Crashed          : Pos 2, 1 Bit
   +0x87c JobVadsAreTracked : Pos 3, 1 Bit
   +0x87c VadTrackingDisabled : Pos 4, 1 Bit
   +0x87c AuxiliaryProcess : Pos 5, 1 Bit
   +0x87c SubsystemProcess : Pos 6, 1 Bit
   +0x87c IndirectCpuSets  : Pos 7, 1 Bit
   +0x87c RelinquishedCommit : Pos 8, 1 Bit
   +0x87c HighGraphicsPriority : Pos 9, 1 Bit
   +0x87c CommitFailLogged : Pos 10, 1 Bit
   +0x87c ReserveFailLogged : Pos 11, 1 Bit
   +0x87c SystemProcess    : Pos 12, 1 Bit
   +0x87c HideImageBaseAddresses : Pos 13, 1 Bit
   +0x87c AddressPolicyFrozen : Pos 14, 1 Bit
   +0x87c ProcessFirstResume : Pos 15, 1 Bit
   +0x87c ForegroundExternal : Pos 16, 1 Bit
   +0x87c ForegroundSystem : Pos 17, 1 Bit
   +0x87c HighMemoryPriority : Pos 18, 1 Bit
   +0x87c EnableProcessSuspendResumeLogging : Pos 19, 1 Bit
   +0x87c EnableThreadSuspendResumeLogging : Pos 20, 1 Bit
   +0x87c SecurityDomainChanged : Pos 21, 1 Bit
   +0x87c SecurityFreezeComplete : Pos 22, 1 Bit
   +0x87c VmProcessorHost  : Pos 23, 1 Bit
   +0x87c VmProcessorHostTransition : Pos 24, 1 Bit
   +0x87c AltSyscall       : Pos 25, 1 Bit
   +0x87c TimerResolutionIgnore : Pos 26, 1 Bit
   +0x87c DisallowUserTerminate : Pos 27, 1 Bit
   +0x880 DeviceAsid       : Int4B
   +0x888 SvmData          : Ptr64 Void
   +0x890 SvmProcessLock   : _EX_PUSH_LOCK
   +0x898 SvmLock          : Uint8B
   +0x8a0 SvmProcessDeviceListHead : _LIST_ENTRY
   +0x8b0 LastFreezeInterruptTime : Uint8B
   +0x8b8 DiskCounters     : Ptr64 _PROCESS_DISK_COUNTERS
   +0x8c0 PicoContext      : Ptr64 Void
   +0x8c8 EnclaveTable     : Ptr64 Void
   +0x8d0 EnclaveNumber    : Uint8B
   +0x8d8 EnclaveLock      : _EX_PUSH_LOCK
   +0x8e0 HighPriorityFaultsAllowed : Uint4B
   +0x8e8 EnergyContext    : Ptr64 _PO_PROCESS_ENERGY_CONTEXT
   +0x8f0 VmContext        : Ptr64 Void
   +0x8f8 SequenceNumber   : Uint8B
   +0x900 CreateInterruptTime : Uint8B
   +0x908 CreateUnbiasedInterruptTime : Uint8B
   +0x910 TotalUnbiasedFrozenTime : Uint8B
   +0x918 LastAppStateUpdateTime : Uint8B
   +0x920 LastAppStateUptime : Pos 0, 61 Bits
   +0x920 LastAppState     : Pos 61, 3 Bits
   +0x928 SharedCommitCharge : Uint8B
   +0x930 SharedCommitLock : _EX_PUSH_LOCK
   +0x938 SharedCommitLinks : _LIST_ENTRY
   +0x948 AllowedCpuSets   : Uint8B
   +0x950 DefaultCpuSets   : Uint8B
   +0x948 AllowedCpuSetsIndirect : Ptr64 Uint8B
   +0x950 DefaultCpuSetsIndirect : Ptr64 Uint8B
   +0x958 DiskIoAttribution : Ptr64 Void
   +0x960 DxgProcess       : Ptr64 Void
   +0x968 Win32KFilterSet  : Uint4B
   +0x970 ProcessTimerDelay : _PS_INTERLOCKED_TIMER_DELAY_VALUES
   +0x978 KTimerSets       : Uint4B
   +0x97c KTimer2Sets      : Uint4B
   +0x980 ThreadTimerSets  : Uint4B
   +0x988 VirtualTimerListLock : Uint8B
   +0x990 VirtualTimerListHead : _LIST_ENTRY
   +0x9a0 WakeChannel      : _WNF_STATE_NAME
   +0x9a0 WakeInfo         : _PS_PROCESS_WAKE_INFORMATION
   +0x9d0 MitigationFlags  : Uint4B
   +0x9d0 MitigationFlagsValues : <anonymous-tag>
   +0x9d4 MitigationFlags2 : Uint4B
   +0x9d4 MitigationFlags2Values : <anonymous-tag>
   +0x9d8 PartitionObject  : Ptr64 Void
   +0x9e0 SecurityDomain   : Uint8B
   +0x9e8 ParentSecurityDomain : Uint8B
   +0x9f0 CoverageSamplerContext : Ptr64 Void
   +0x9f8 MmHotPatchContext : Ptr64 Void
   +0xa00 DynamicEHContinuationTargetsTree : _RTL_AVL_TREE
   +0xa08 DynamicEHContinuationTargetsLock : _EX_PUSH_LOCK
   +0xa10 DynamicEnforcedCetCompatibleRanges : _PS_DYNAMIC_ENFORCED_ADDRESS_RANGES
   +0xa20 DisabledComponentFlags : Uint4B
```

打开windbg,输入`dt _eprocess`就可以列出当前操作系统的eprocess结构  
其中, `ActiveProcessLinks`是一个`_LIST_ENTRY`结构,也就是双向链表,它指向了上一个和下一个进程(eprocess)的ActiveProcessLinks的地址  
在这里,位于地址+0x448的地方就是ActiveProcessLinks  

``` asm
nt!_LIST_ENTRY
   +0x000 Flink            : Ptr64 _LIST_ENTRY
   +0x008 Blink            : Ptr64 _LIST_ENTRY
```

Flink指向上一个
Blink指向下一个

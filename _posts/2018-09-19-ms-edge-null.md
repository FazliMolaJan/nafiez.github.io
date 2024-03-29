---
layout: post
title:  "Microsoft Edge 11.00.16299.492 - NULL Pointer Dereference"
date:   2018-09-19 03:39:03 +0700
categories: [security, null]
---

Description
-----------
The issue was found using Domato by Google (modified version). I was using EdgeWinDBG.cmd by skylined to debug the browser. Issue has 
been reported to Microsoft (CRM:0461055787) and does not meet  the security bar. I didn't get chance to provide POC here, need to 
minimize the code before I can share.

Did some reversing on Windows 10 and figure out that there's no chance to trigger for Use-After-Free or even Heap corruption. This could 
be by design to protect Edge browser being exploited. I did found similar information by Skylined in his Twitter, https://twitter.com/berendjanwever/status/1001366547871551488.

Initial triage crash:
```
(13f0.27e4): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
edgehtml!Fetch::OriginTuple::OriginTuple+0x4d:
00007ffb`d47d4625 488b01          mov     rax,qword ptr [rcx] ds:00000000`00000000=????????????????
```

We can see the pointer of [RCX] is moving to RAX, RCX contained NULL value. We can observe this in register:
```
1:107> r
rax=0000000c927f6818 rbx=0000000c927f66c0 rcx=0000000000000000
rdx=0000000c927f6800 rsi=0000000000000002 rdi=0000000c927f61d0
rip=00007ffbd47d4625 rsp=0000000c927f6790 rbp=0000000c927f67d0
 r8=0000000000000000  r9=000001bbd8fff008 r10=000000000000003d
r11=0000000c927f67c0 r12=00007ffbd49568b0 r13=000001bbd4cc08d0
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na pe nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010200
edgehtml!Fetch::OriginTuple::OriginTuple+0x4d:
00007ffb`d47d4625 488b01          mov     rax,qword ptr [rcx] ds:00000000`00000000=????????????????
```
Our heap:
```
1:107> !heap -i @rsi
Detailed information for block entry 0000000000000002
Assumed heap       : 0x0000000000000000 (Use !heap -i NewHeapHandle to change)
Header content     : 0x00000000 0x00000000
Owning segment     : 0x0000000000000000 (offset 0)
Block flags        : 0x0 (free )
Total block size   : 0x0 units (0x0 bytes)
Previous block size: 0x0 units (0x0 bytes)
Block CRC          : OK - 0x0  
List corrupted: (Blink->Flink = 0000000000000000) != (Block = 0000000000000012)
Free list entry    : CORRUPTED
Next block         : 0x0000000000000002
```
Full Dump with the information of NULL issue:
```
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************

*** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\WINDOWS\SYSTEM32\igd10umd64.dll - 
GetUrlPageData2 (WinHttp) failed: 12002.

DUMP_CLASS: 2
DUMP_QUALIFIER: 0
FAULTING_IP: 
edgehtml!Fetch::OriginTuple::OriginTuple+4d
00007ffb`d47d4625 488b01          mov     rax,qword ptr [rcx]

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ffbd47d4625 (edgehtml!Fetch::OriginTuple::OriginTuple+0x000000000000004d)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000001
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: 0000000000000000
Attempt to read from address 0000000000000000

FAULTING_THREAD:  000027e4
DEFAULT_BUCKET_ID:  NULL_POINTER_READ
PROCESS_NAME:  MicrosoftEdgeCP.exe
ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.
EXCEPTION_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.
EXCEPTION_CODE_STR:  c0000005
EXCEPTION_PARAMETER1:  0000000000000000
EXCEPTION_PARAMETER2:  0000000000000000

FOLLOWUP_IP: 
edgehtml!Fetch::OriginTuple::OriginTuple+4d
00007ffb`d47d4625 488b01          mov     rax,qword ptr [rcx]

READ_ADDRESS:  0000000000000000 
BUGCHECK_STR:  NULL_POINTER_READ
WATSON_BKT_PROCSTAMP:  5b1a1a5d
WATSON_BKT_PROCVER:  11.0.16299.492
PROCESS_VER_PRODUCT:  Microsoft Edge
WATSON_BKT_MODULE:  edgehtml.dll
WATSON_BKT_MODSTAMP:  ba357c2f
WATSON_BKT_MODOFFSET:  3f4625
WATSON_BKT_MODVER:  11.0.16299.492
MODULE_VER_PRODUCT:  Microsoft Edge Web Platform
BUILD_VERSION_STRING:  10.0.16299.431 (WinBuild.160101.0800)
MODLIST_WITH_TSCHKSUM_HASH:  42f18e3edf038aad47960e4f898f1ef770f8d16e
MODLIST_SHA1_HASH:  150320058b018006e4c5235ff9ffd75f299003a3
NTGLOBALFLAG:  0
APPLICATION_VERIFIER_FLAGS:  0
PRODUCT_TYPE:  1
SUITE_MASK:  272
ANALYSIS_SESSION_HOST:  DESKTOP-N8NGOGJ
ANALYSIS_SESSION_TIME:  07-06-2018 14:50:36.0856
ANALYSIS_VERSION: 10.0.14321.1024 amd64fre

THREAD_ATTRIBUTES: 
OS_LOCALE:  ENM

PROBLEM_CLASSES: 

NULL_POINTER_READ
    Tid    [0x27e4]
    Frame  [0x00]: edgehtml!Fetch::OriginTuple::OriginTuple

LAST_CONTROL_TRANSFER:  from 00007ffbd4802abd to 00007ffbd47d4625
STACK_TEXT:  
0000000c`927f6790 00007ffb`d4802abd : 000001bb`f0537220 000001bb`ed5aff80 0000000c`927f68c8 00000000`00000000 : edgehtml!Fetch::OriginTuple::OriginTuple+0x4d
0000000c`927f6800 00007ffb`d51e72a5 : 00000000`02000001 00000000`00000000 00007ffb`d39a0000 0000000c`927f6a20 : edgehtml!Fetch::Utils::s_GetSerializedOrigin+0x21
0000000c`927f6890 00007ffb`d4d4eb32 : 000001bb`d4c2ce58 00000000`02000001 0000000c`927f6a29 00000000`02000001 : edgehtml!CUrl::Var_get_origin+0x61
0000000c`927f6950 00007ffb`d49568d5 : 000001bb`d4c2af70 00000000`00000001 000001bb`f1d3fa40 00000000`00000000 : edgehtml!CFastDOM::CURL::Trampoline_Get_origin+0x3e
0000000c`927f6990 00007ffb`d3be6ac1 : 000001bb`eddac150 00000000`02000001 0000000c`927f6aa0 000001b3`8038c350 : edgehtml!CFastDOM::CURL::Profiler_Get_origin+0x25
0000000c`927f69c0 00007ffb`d3b4b04e : 000001bb`eddac150 00000000`02000001 000001bb`ed5b98c0 000001bb`d4cc08d0 : chakra!Js::JavascriptExternalFunction::ExternalFunctionThunk+0x1d1
0000000c`927f6a90 00007ffb`d3aa1c12 : 0000000c`927f6b90 00000000`00000035 00000000`0000ffff 00070000`00000000 : chakra!<lambda_fc0e308ae07e4768a4bfa4a43a1dda76>::operator()+0xee
0000000c`927f6ad0 00007ffb`d3b5f91f : 000001bb`d4c2af70 000001bb`eddac150 00000000`00000640 0000000c`927f6b90 : chakra!ThreadContext::ExecuteImplicitCall<<lambda_fc0e308ae07e4768a4bfa4a43a1dda76> >+0xa2
0000000c`927f6b30 00007ffb`d3b6163a : 00000000`0000000f 000001bb`d85f1540 000001bb`ed5b98c0 000001bb`ed517b88 : chakra!Js::DictionaryTypeHandlerBase<unsigned short>::GetPropertyFromDescriptor<0,int>+0x20f
0000000c`927f6c00 00007ffb`d3a7f4b0 : 000001bb`ed554e40 000001bb`d85f1540 000001bb`ed5b98c0 0000000c`0000067c : chakra!Js::DictionaryTypeHandlerBase<unsigned short>::GetProperty+0x11a
0000000c`927f6c70 00007ffb`d3a9b939 : 000001bb`d85f1540 000001bb`ed5b98c0 0000000c`0000067c 0000000c`927f6dc0 : chakra!Js::CustomExternalObject::GetPropertyQuery+0x150
0000000c`927f6d70 00007ffb`d3b6e0ef : 000001bb`ed5b98c0 000001bb`ed5b98c0 000001bb`0000067c 000001bb`d4cc08d0 : chakra!Js::JavascriptOperators::GetProperty+0xa9
0000000c`927f6e00 00007ffb`d3b76700 : 0000000c`927f7120 000001bb`ed5b98c0 000001b3`8015db37 00007ffb`d3a9ecab : chakra!Js::InterpreterStackFrame::OP_GetProperty_NoFastPath<Js::OpLayoutT_ElementCP<Js::LayoutSizePolicy<1> > const >+0xdf
0000000c`927f6ec0 00007ffb`d3b748fc : 0000000c`927f7120 000001b3`8015db36 0000000c`927f6f50 00000000`00000000 : chakra!Js::InterpreterStackFrame::ProcessUnprofiledMediumLayoutPrefix+0x340
0000000c`927f6f20 00007ffb`d3b74788 : 0000000c`927f7120 00000000`00000000 00000000`00000001 00000000`00000000 : chakra!Js::InterpreterStackFrame::ProcessUnprofiled+0x8c
0000000c`927f6f80 00007ffb`d3b731f4 : 0000000c`927f7120 0000000c`927f7120 0000000c`927f7120 00000000`00000002 : chakra!Js::InterpreterStackFrame::Process+0x1a8
0000000c`927f6fe0 00007ffb`d3b756f6 : 0000000c`927f7120 000001b3`8015db33 000001b3`8015db33 00000000`00000000 : chakra!Js::InterpreterStackFrame::OP_TryCatch+0x64
0000000c`927f7030 00007ffb`d3b74788 : 0000000c`927f7120 00000000`00000000 00000000`00000000 00000000`00000000 : chakra!Js::InterpreterStackFrame::ProcessUnprofiled+0xe86
0000000c`927f7090 00007ffb`d3b7b9b0 : 0000000c`927f7120 0000000c`927f7120 0000000c`927fb930 00000000`00000000 : chakra!Js::InterpreterStackFrame::Process+0x1a8
0000000c`927f70f0 00007ffb`d3b7ad02 : 000001bb`d85a1ec0 0000000c`927fbb10 000001bb`e92b0fb2 0000000c`927fbb28 : chakra!Js::InterpreterStackFrame::InterpreterHelper+0x9d0
0000000c`927fbae0 000001bb`e92b0fb2 : 0000000c`927fbb60 00000000`00000001 0000000c`927fbb50 0000000c`927fbed0 : chakra!Js::InterpreterStackFrame::InterpreterThunk+0x52
0000000c`927fbb30 00007ffb`d3c1ade6 : 000001bb`d85a1ec0 00000000`10000001 000001bb`d86540b0 00000000`00000001 : 0x000001bb`e92b0fb2
0000000c`927fbb60 00007ffb`d3a911bc : 000001bb`d4c2af70 00000000`00000008 000001bb`e91735a0 000001b3`80b52070 : chakra!amd64_CallFunction+0x86
0000000c`927fbbb0 00007ffb`d3b6f7ae : 000001bb`d85a1ec0 00007ffb`d3c1af50 0000000c`927fbc30 000001bb`d4c2af70 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
0000000c`927fbc10 00007ffb`d3b74cfe : 0000000c`927fbd50 000001bb`d8539c97 000001bb`8090cd00 0000000c`927fbd50 : chakra!Js::InterpreterStackFrame::OP_CallI<Js::OpLayoutDynamicProfile<Js::OpLayoutT_CallI<Js::LayoutSizePolicy<0> > > >+0xee
0000000c`927fbc60 00007ffb`d3b74788 : 0000000c`927fbd50 00000000`00000000 00000000`00000000 00000000`00000000 : chakra!Js::InterpreterStackFrame::ProcessUnprofiled+0x48e
0000000c`927fbcc0 00007ffb`d3b7b9b0 : 0000000c`927fbd50 0000000c`927fbd50 0000000c`927fbee0 00007ffb`d3c07d00 : chakra!Js::InterpreterStackFrame::Process+0x1a8
0000000c`927fbd20 00007ffb`d3b7ad02 : 000001bb`d85a17a0 0000000c`927fc0c0 000001bb`e92b0fba 0000000c`927fc0d8 : chakra!Js::InterpreterStackFrame::InterpreterHelper+0x9d0
0000000c`927fc090 000001bb`e92b0fba : 0000000c`927fc110 00000000`00000002 0000000c`927fc100 0000000c`927fc5e8 : chakra!Js::InterpreterStackFrame::InterpreterThunk+0x52
0000000c`927fc0e0 00007ffb`d3c1ade6 : 000001bb`d85a17a0 00000000`00000002 000001bb`d86540b0 000001bb`e919bf80 : 0x000001bb`e92b0fba
0000000c`927fc110 00007ffb`d3a911bc : 000001bb`d4c2af70 00000000`00000010 000001bb`f14b9c40 00007ffb`d3b80cd9 : chakra!amd64_CallFunction+0x86
0000000c`927fc160 00007ffb`d3bde9f9 : 000001bb`d85a17a0 00007ffb`d3c1af50 0000000c`927fc200 000001bb`d4cc08d0 : chakra!Js::JavascriptFunction::CallFunction<1>+0x9c
0000000c`927fc1c0 00007ffb`d3bde8d2 : 000001bb`d85a17a0 0000000c`927fc308 000001bb`d4cc08d0 000001bb`d85f0200 : chakra!Js::JavascriptFunction::CallRootFunctionInternal+0x105
0000000c`927fc2b0 00007ffb`d3bde815 : 000001bb`d85a17a0 0000000c`927fc370 000001bb`d4cc08d0 000001b3`8090c600 : chakra!Js::JavascriptFunction::CallRootFunction+0x7e
0000000c`927fc330 00007ffb`d3a8147b : 000001bb`d85a17a0 0000000c`927fc3c0 00000000`00000000 0000000c`927fc3d0 : chakra!ScriptSite::CallRootFunction+0x6d
0000000c`927fc390 00007ffb`d3a844c6 : 000001bb`f14ba940 000001bb`d85a17a0 0000000c`927fc460 00000000`00000000 : chakra!ScriptSite::Execute+0x12b
0000000c`927fc420 00007ffb`d4721961 : 000001bb`d4c44370 000001bb`d85a17a0 0000000c`00000002 0000000c`927fc5e8 : chakra!ScriptEngineBase::Execute+0xb6
0000000c`927fc4c0 00007ffb`d4721888 : 00000001`ce5aeb6f 00007ffb`d4721ac5 000001bb`ed989b90 00000000`80004005 : edgehtml!CJScript9Holder::ExecuteCallbackDirect+0x3d
0000000c`927fc510 00007ffb`d479b3e8 : 000001bb`ee03ff00 0000000c`927fc621 000001bb`ee03ca18 000001bb`f1dd7500 : edgehtml!CJScript9Holder::ExecuteCallback+0x50
0000000c`927fc550 00007ffb`d479b115 : 000001bb`ee03ca18 000001bb`d85a17a0 000001bb`ed989b90 000001bb`ee03ca10 : edgehtml!CListenerDispatch::InvokeVar+0x258
0000000c`927fc670 00007ffb`d479ad47 : 000001bb`ee03ca18 000001bb`ed989b90 000001bb`ee03ca10 0000000c`927fc930 : edgehtml!CListenerDispatch::Invoke+0xbd
0000000c`927fc700 00007ffb`d479a95e : 000001bb`ed989b90 000001b3`d49b8110 0000000c`927fc930 000001bb`ed989b01 : edgehtml!CEventMgr::_InvokeListeners+0x307
0000000c`927fc860 00007ffb`d47c3a64 : 000001bb`ed989b90 0000000c`927fc930 000001bb`ed989b90 000001b3`d49dc3c0 : edgehtml!CEventMgr::_InvokeListenersOnWindow+0x66
0000000c`927fc890 00007ffb`d47c346a : 000001bb`ed989b90 000001b3`d49b8110 0000000c`927fcc00 00007ffb`ffffffff : edgehtml!CEventMgr::Dispatch+0x5d4
0000000c`927fcba0 00007ffb`d4702689 : 00007ffb`d5409290 000001b3`d49b8110 000001bb`ee020720 00000000`ffffffff : edgehtml!CEventMgr::DispatchEvent+0x6e
0000000c`927fcbf0 00007ffb`d470149d : 000001b3`d4a08598 000001bb`f1df7200 000001bb`ed93b3d8 00000000`08000000 : edgehtml!COmWindowProxy::Fire_onload+0x171
0000000c`927fcd00 00007ffb`d46fc6c7 : 00000000`ffffff01 000001b3`00000000 000001bb`f1df7200 000001bb`f1df7200 : edgehtml!CMarkup::OnLoadStatusDone+0x421
0000000c`927fcdc0 00007ffb`d47c8cac : 000001bb`f1d7f900 0000000c`927fcef0 000001b3`d49f8000 00000000`00000000 : edgehtml!CMarkup::OnLoadStatus+0x11f
0000000c`927fcdf0 00007ffb`d48d9f5b : 000001b3`d4a28110 000001bb`ed98cea0 000001b3`d2c82ec0 00000001`ce5ac9d3 : edgehtml!CProgSink::DoUpdate+0x3e0
0000000c`927fd270 00007ffb`d4789ef8 : 00000004`b107ae74 00000000`0027a306 00000000`000f4240 00007ffb`d4788b59 : edgehtml!GWndAsyncTask::Run+0x1b
0000000c`927fd2a0 00007ffb`d48b30a5 : 00000004`b107ae74 000001bb`f08fe050 000001bb`f1d7f900 00000000`00000002 : edgehtml!HTML5TaskScheduler::RunReadiedTask+0x208
0000000c`927fd370 00007ffb`d483f689 : 000001b3`d49d8170 0000000c`927fd760 00000000`00000000 00000000`00000159 : edgehtml!HTML5TaskScheduler::RunReadiedTasks+0x115
0000000c`927fd3e0 00007ffb`d4788b92 : 00000001`ce5aca44 000001b3`d49d0000 00000000`00008002 000001b3`d49d0000 : edgehtml!NormalPriorityAtInputEventLoopDriver::DriveRegularPriorityTaskExecution+0xa9
0000000c`927fd410 00007ffb`f326b85d : 00000000`0003074a 00000000`00000001 00000000`00000002 00007ffb`d28a1552 : edgehtml!GlobalWndProc+0x162
0000000c`927fd470 00007ffb`f326b1ef : 000001b3`d32a8a30 00007ffb`d4788a30 00000000`0003074a 0000000c`906d3800 : user32!UserCallWinProcCheckWow+0x2ad
0000000c`927fd5e0 00007ffb`cb508d9e : 0000000c`927fd730 00000000`00000000 000001b3`d2c32030 000001b3`d2c32030 : user32!DispatchMessageWorker+0x19f
0000000c`927fd660 00007ffb`cb4d792e : 00000000`00000001 00000000`00000001 000001b3`d2cf0af0 00000000`00000000 : EdgeContent!CBrowserTab::_TabWindowThreadProc+0x42e
0000000c`927ff8e0 00007ffb`d6711878 : 00000000`00000000 000001b3`d2d27140 00000000`00000000 00000000`00000000 : EdgeContent!LCIETab_ThreadProc+0x32e
0000000c`927ffa10 00007ffb`f3461fe4 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : edgeIso!_IsoThreadProc_WrapperToReleaseScope+0x48
0000000c`927ffa40 00007ffb`f503cb31 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : KERNEL32!BaseThreadInitThunk+0x14
0000000c`927ffa70 00000000`00000000 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x21

THREAD_SHA1_HASH_MOD_FUNC:  b51e4d0c493545aaedb68388dcb1ed0ff19cd982
THREAD_SHA1_HASH_MOD_FUNC_OFFSET:  312f2fee2463a7669ccd59ced383c0b0adf6bc2d
THREAD_SHA1_HASH_MOD:  847cf92fda0432c4a527683f91ab1f501224c188
FAULT_INSTR_CODE:  48018b48
SYMBOL_STACK_INDEX:  0
SYMBOL_NAME:  edgehtml!Fetch::OriginTuple::OriginTuple+4d
FOLLOWUP_NAME:  MachineOwner
MODULE_NAME: edgehtml
IMAGE_NAME:  edgehtml.dll
DEBUG_FLR_IMAGE_TIMESTAMP:  0
STACK_COMMAND:  ~107s ; kb
BUCKET_ID:  NULL_POINTER_READ_edgehtml!Fetch::OriginTuple::OriginTuple+4d
PRIMARY_PROBLEM_CLASS:  NULL_POINTER_READ_edgehtml!Fetch::OriginTuple::OriginTuple+4d
FAILURE_EXCEPTION_CODE:  c0000005
FAILURE_IMAGE_NAME:  edgehtml.dll
BUCKET_ID_IMAGE_STR:  edgehtml.dll
FAILURE_MODULE_NAME:  edgehtml
BUCKET_ID_MODULE_STR:  edgehtml
FAILURE_FUNCTION_NAME:  Fetch::OriginTuple::OriginTuple
BUCKET_ID_FUNCTION_STR:  Fetch::OriginTuple::OriginTuple
```

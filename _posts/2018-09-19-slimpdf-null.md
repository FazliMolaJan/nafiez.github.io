---
layout: post
title:  "SlimPDF Reader - NULL Pointer Dereference"
date:   2018-09-19 03:39:03 +0700
categories: [security, null]
---

Vulnerability Description
-------------------------
Few months back, I found a bug in SlimPDF Reader. The bug leads to NULL Pointer dereference. I reported the issue on May 2018 and seems 
update from the developer, https://support.investintech.com/hc/en-us/requests/14369.

Affected software: https://www.investintech.com/download/InstallSlimPDFReader.exe

Initial Analysis
----------------
To trigger the NULL pointer, open PDF with SlimPDF Reader. Upon opening the PDF, an exception occured.
```
(f84.cdc): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
*** WARNING: Unable to verify checksum for C:\Program Files (x86)\Investintech.com Inc\SlimPDF Reader 1.0\SlimPDF Reader.exe
*** ERROR: Module load completed but symbols could not be loaded for C:\Program Files (x86)\Investintech.com Inc\SlimPDF Reader 1.0\SlimPDF Reader.exe
SlimPDF_Reader+0x3457c:
0043457c 394608          cmp     dword ptr [esi+8],eax ds:002b:00000008=????????
```
Looking at the exception code c0000005, it is a code for an access violation. That means that the program is accessing a memory address to which hasn't right to do. 
```
0:000:x86> .exr -1
ExceptionAddress: 0043457c (SlimPDF_Reader+0x0003457c)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 00000000
   Parameter[1]: 00000008
Attempt to read from address 00000008
0:000:x86> r
eax=00000002 ebx=023274a8 ecx=04357020 edx=00000005 esi=00000000 edi=0231e928
eip=0043457c esp=0019f6e0 ebp=0019f860 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00210246
SlimPDF_Reader+0x3457c:
0043457c 394608          cmp     dword ptr [esi+8],eax ds:002b:00000008=????????
```

If we see the registers above, we can see ESI contained NULL value. We can see this via memory:
```
0:000:x86> dc esi
00000000  ???????? ???????? ???????? ????????  ????????????????
00000010  ???????? ???????? ???????? ????????  ????????????????
00000020  ???????? ???????? ???????? ????????  ????????????????
00000030  ???????? ???????? ???????? ????????  ????????????????
00000040  ???????? ???????? ???????? ????????  ????????????????
00000050  ???????? ???????? ???????? ????????  ????????????????
00000060  ???????? ???????? ???????? ????????  ????????????????
00000070  ???????? ???????? ???????? ????????  ????????????????
```

Root Cause
----------
Inspecting the issue leads to the failure parsing the stream object thus causing NULL pointer dereference. Crash part:
```
.text:0043456D                 mov     eax, [ebx+8]		; [ebx+8] containing invalid value
.text:00434570                 cmp     eax, 2			; compare eax = 2, 
.text:00434573                 mov     esi, [ebx+28h]	; [ebx+28h] containing our PDF stream, pointer to 
.text:00434576                 jnz     loc_434826		; zero flag set here, so no jump
.text:0043457C                 cmp     [esi+8], eax		; continue executing and crash
```
We can see the object stream in EBX as in following:
```
0:000:x86> dps 023274a8
023274a8  ffffffff
023274ac  ffffffff
023274b0  00000002
023274b4  41414141
023274b8  41410033
023274bc  41414141
023274c0  41414141
023274c4  41414141
023274c8  00000001
023274cc  0000000f
023274d0  00000000
023274d4  00000000
023274d8  00000000
023274dc  41414141
023274e0  bdd3747d
023274e4  80002241
023274e8  ffffffff
023274ec  ffffffff
023274f0  00000000
023274f4  41414141
023274f8  41414100
023274fc  41414141
02327500  41414141
02327504  41414141
02327508  00000000
0232750c  0000000f
02327510  00000000
02327514  00000000
02327518  00000000
0232751c  41414141
02327520  bddb7445
02327524  80002341
```

EAX containing pointer to an invalid memory:
```
0:000:x86> db poi (ebx+8)
00000002  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000012  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000022  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000032  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000042  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000052  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000062  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
00000072  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
```

Crash dump details:
```
0:000:x86> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************

GetUrlPageData2 (WinHttp) failed: 12002.

KEY_VALUES_STRING: 1

TIMELINE_ANALYSIS: 1

Timeline: !analyze.Start
    Name: <blank>
    Time: 2018-05-28T05:55:00.250Z
    Diff: 250 mSec

Timeline: Dump.Current
    Name: <blank>
    Time: 2018-05-28T05:55:00.0Z
    Diff: 0 mSec

Timeline: Process.Start
    Name: <blank>
    Time: 2018-05-28T04:16:48.0Z
    Diff: 5892000 mSec

Timeline: OS.Boot
    Name: <blank>
    Time: 2018-05-26T16:13:30.0Z
    Diff: 135690000 mSec

DUMP_CLASS: 2
DUMP_QUALIFIER: 0

FAULTING_IP: 
SlimPDF_Reader+3457c
0043457c 394608          cmp     dword ptr [esi+8],eax

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 0043457c (SlimPDF_Reader+0x0003457c)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 00000000
   Parameter[1]: 00000008
Attempt to read from address 00000008

FAULTING_THREAD:  00000cdc
DEFAULT_BUCKET_ID:  NULL_CLASS_PTR_READ
PROCESS_NAME:  SlimPDF Reader.exe

FOLLOWUP_IP: 
SlimPDF_Reader+3457c
0043457c 394608          cmp     dword ptr [esi+8],eax

READ_ADDRESS:  00000008 
ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.
EXCEPTION_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.
EXCEPTION_CODE_STR:  c0000005
EXCEPTION_PARAMETER1:  00000000
EXCEPTION_PARAMETER2:  00000008
WATSON_BKT_PROCSTAMP:  4ecd831a
WATSON_BKT_PROCVER:  1.0.1.12
PROCESS_VER_PRODUCT:  SlimPDF Reader
WATSON_BKT_MODULE:  SlimPDF Reader.exe
WATSON_BKT_MODSTAMP:  4ecd831a
WATSON_BKT_MODOFFSET:  3457c
WATSON_BKT_MODVER:  1.0.1.12
MODULE_VER_PRODUCT:  SlimPDF Reader
BUILD_VERSION_STRING:  10240.17443.amd64fre.th1.170602-2340
MODLIST_WITH_TSCHKSUM_HASH:  2b36701f96731def5752c269357e7e5a2679befb
MODLIST_SHA1_HASH:  d231aecbd4b1f32bc2f593851f16d87c2da75c72
NTGLOBALFLAG:  400
PROCESS_BAM_CURRENT_THROTTLED: 0
PROCESS_BAM_PREVIOUS_THROTTLED: 0
APPLICATION_VERIFIER_FLAGS:  0
PRODUCT_TYPE:  1
SUITE_MASK:  272
DUMP_TYPE:  fe
ANALYSIS_SESSION_HOST:  DESKTOP-1IBEKMI
ANALYSIS_SESSION_TIME:  05-28-2018 13:55:00.0250
ANALYSIS_VERSION: 10.0.17134.1 amd64fre
THREAD_ATTRIBUTES: 
OS_LOCALE:  ENU

PROBLEM_CLASSES: 
    ID:     [0n309]
    Type:   [@ACCESS_VIOLATION]
    Class:  Addendum
    Scope:  BUCKET_ID
    Name:   Omit
    Data:   Omit
    PID:    [Unspecified]
    TID:    [0xcdc]
    Frame:  [0] : SlimPDF_Reader

    ID:     [0n281]
    Type:   [INVALID_POINTER_READ]
    Class:  Primary
    Scope:  BUCKET_ID
    Name:   Add
    Data:   Omit
    PID:    [Unspecified]
    TID:    [0xcdc]
    Frame:  [0] : SlimPDF_Reader

    ID:     [0n306]
    Type:   [NULL_CLASS_PTR_READ]
    Class:  Primary
    Scope:  DEFAULT_BUCKET_ID (Failure Bucket ID prefix)
            BUCKET_ID
    Name:   Add
    Data:   Omit
    PID:    [0xf84]
    TID:    [0xcdc]
    Frame:  [0] : SlimPDF_Reader

    ID:     [0n156]
    Type:   [ZEROED_STACK]
    Class:  Addendum
    Scope:  BUCKET_ID
    Name:   Add
    Data:   Omit
    PID:    [0xf84]
    TID:    [0xcdc]
    Frame:  [0] : SlimPDF_Reader

BUGCHECK_STR:  APPLICATION_FAULT_NULL_CLASS_PTR_READ_INVALID_POINTER_READ_ZEROED_STACK
PRIMARY_PROBLEM_CLASS:  APPLICATION_FAULT
LAST_CONTROL_TRANSFER:  from 0044211b to 0043457c

STACK_TEXT:  
WARNING: Stack unwind information not available. Following frames may be wrong.
0019f860 0044211b 0231d3b0 0231deb8 00000000 SlimPDF_Reader+0x3457c
0019f8e4 00443702 0231d270 022f7ef8 022f7f28 SlimPDF_Reader+0x4211b
0019f914 0041be0d 00000001 00000000 0231d3b0 SlimPDF_Reader+0x43702
0019f984 0041d20b 00000001 022df730 3f0119b6 SlimPDF_Reader+0x1be0d
0019f9d4 0045fbf0 00000001 00000001 3f0119b6 SlimPDF_Reader+0x1d20b
0019fa78 004cee26 0000007c 004cdea8 022b5340 SlimPDF_Reader+0x5fbf0
0019fb60 004afa7e 52fbf7b5 0000000f 022e1c38 SlimPDF_Reader+0xcee26
0019fbf4 004ab964 0000000f 00000000 0052df50 SlimPDF_Reader+0xafa7e
0019fc14 004ae035 0000000f 00000000 00000000 SlimPDF_Reader+0xab964
0019fc7c 004ae0c2 00000000 0003084a 0000000f SlimPDF_Reader+0xae035
0019fc9c 74904923 0003084a 0000000f 00000000 SlimPDF_Reader+0xae0c2
0019fcc8 748e4790 004ae08e 0003084a 0000000f USER32!_InternalCallWinProc+0x2b
0019fd70 748e4370 004ae08e 00000000 0000000f USER32!UserCallWinProcCheckWow+0x1f0
0019fdd0 748eb179 00eed8f0 00000000 0000000f USER32!DispatchClientMessage+0xf0
0019fe10 777fad66 0019fe2c 00000020 0019fe90 USER32!__fnDWORD+0x49
0019fe48 7490509c 748e419b 008aafd0 787b6351 ntdll_77790000!KiUserCallbackDispatcher+0x36
0019fe4c 748e419b 008aafd0 787b6351 008aafa0 USER32!NtUserDispatchMessage+0xc
0019fea0 748e3e50 78629df1 00000000 004b1d6c USER32!DispatchMessageWorker+0x33b
0019feac 004b1d6c 008aafd0 008aafd0 00599b10 USER32!DispatchMessageW+0x10
00000000 00000000 00000000 00000000 00000000 SlimPDF_Reader+0xb1d6c

STACK_COMMAND:  ~0s ; .cxr ; kb
THREAD_SHA1_HASH_MOD_FUNC:  23c6551acd70e2ee68c3fe17230a519c8d2edbd8
THREAD_SHA1_HASH_MOD_FUNC_OFFSET:  c0fb1c21cd18236bfb477e6f4ab150f37a3c72a8
THREAD_SHA1_HASH_MOD:  fca436a783f277e0e687fc89df0c8a6506b6fcd9
FAULT_INSTR_CODE:  75084639
SYMBOL_STACK_INDEX:  0
SYMBOL_NAME:  SlimPDF_Reader+3457c
FOLLOWUP_NAME:  MachineOwner
MODULE_NAME: SlimPDF_Reader
IMAGE_NAME:  SlimPDF_Reader.exe
DEBUG_FLR_IMAGE_TIMESTAMP:  4ecd831a
FAILURE_BUCKET_ID:  NULL_CLASS_PTR_READ_c0000005_SlimPDF_Reader.exe!Unknown
BUCKET_ID:  APPLICATION_FAULT_NULL_CLASS_PTR_READ_INVALID_POINTER_READ_ZEROED_STACK_SlimPDF_Reader+3457c
FAILURE_EXCEPTION_CODE:  c0000005
FAILURE_IMAGE_NAME:  SlimPDF_Reader.exe
BUCKET_ID_IMAGE_STR:  SlimPDF_Reader.exe
FAILURE_MODULE_NAME:  SlimPDF_Reader
BUCKET_ID_MODULE_STR:  SlimPDF_Reader
FAILURE_FUNCTION_NAME:  Unknown
BUCKET_ID_FUNCTION_STR:  Unknown
BUCKET_ID_OFFSET:  3457c
BUCKET_ID_MODTIMEDATESTAMP:  4ecd831a
BUCKET_ID_MODCHECKSUM:  0
BUCKET_ID_MODVER_STR:  1.0.1.12
BUCKET_ID_PREFIX_STR:  APPLICATION_FAULT_NULL_CLASS_PTR_READ_INVALID_POINTER_READ_ZEROED_STACK_
FAILURE_PROBLEM_CLASS:  APPLICATION_FAULT
FAILURE_SYMBOL_NAME:  SlimPDF_Reader.exe!Unknown
WATSON_STAGEONE_URL:  http://watson.microsoft.com/StageOne/SlimPDF Reader.exe/1.0.1.12/4ecd831a/SlimPDF Reader.exe/1.0.1.12/4ecd831a/c0000005/0003457c.htm?Retriage=1
TARGET_TIME:  2018-05-28T05:55:11.000Z
OSBUILD:  10240
OSSERVICEPACK:  17113
SERVICEPACK_NUMBER: 0
OS_REVISION: 0
OSPLATFORM_TYPE:  x64
OSNAME:  Windows 10
OSEDITION:  Windows 10 WinNt SingleUserTS
USER_LCID:  0
OSBUILD_TIMESTAMP:  2016-09-07 11:54:11
BUILDDATESTAMP_STR:  170602-2340
BUILDLAB_STR:  th1
BUILDOSVER_STR:  10.0.10240.17443.amd64fre.th1.170602-2340
ANALYSIS_SESSION_ELAPSED_TIME:  81ce
ANALYSIS_SOURCE:  UM
FAILURE_ID_HASH_STRING:  um:null_class_ptr_read_c0000005_slimpdf_reader.exe!unknown
FAILURE_ID_HASH:  {46980e42-0101-b5c8-decb-c547562bd18a}
Followup:     MachineOwner
---------
```

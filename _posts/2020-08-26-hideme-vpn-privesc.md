---
layout: post
title:  "hide.me VPN Windows Client Privilege Escalation Vulnerability"
date:   2020-08-26 12:30:00 +0800
categories: [security, privesc]
---

Overview
--------
hide.me is one of the fastest VPN service provider in the VPN industry, committed to develop applications and services that provides an open access of the internet, as well as protect users' online privacy. With VPN servers in 72 different locations, fully-managed and maintained, hide.me offers a comprehensive reach to its users and empowers them to access blocked and restricted content on the internet. hide.me offers easy-to-install apps for Windows, macOS, Android and iOS with best-in-class Internet security. hide.me continues to grow its customer base and service offerings as an ever-greater number of internet users are adopting the use of VPN.

hide.me VPN Service creates a log file in "C:\Users\<user>\AppData\Roaming\Hide.me\" by default. The files are created, accessed and manipulated by privileged (SYSTEM) processes of hide.me VPN service. The log and folder have permissive access rights that allow unprivileged users to add/remove files and change properties. Affected version of hide.me VPN Windows Client 3.4.3. Version below might be affected too, but untested. The vulnerability has been fixed on version 3.5.0.

Vulnerability Analysis
----------------------
A process running with higher privileges (e.g. SYSTEM) and perform operations on files like the rest of the processes that has access to user-controlled files or directories without restrictions could lead to a security issue. The operations could be abuse by leveraging the privileged process to perform unwanted activity. This issue well-known as logical vulnerability. Most privilege programs will not manipulate unprivileged user-access files indirectly, however most of it perform operations on a files that located somewhere a user can have access to it. Here's a list of interesting locations based on this [blog](https://offsec.almond.consulting/intro-to-file-operation-abuse-on-Windows.html):
- The user's own files & directories, including its AppData and Temp folders, that some privileged process may use if you're lucky or running an VPN services
- The Public user's files & directories: idem
- Directories created in C:\ with default ACL: by default, directories created at the root of partitions do have a permissive ACL that allows write access for users
- Subdirectories of C:\ProgramData with default ACL: by default, users can create files and directories, but not modify existing ones. This is often the first place to look at.
- Subdirectories of C:\Windows\Temp: by default, users can create files and directories, but not modify existing ones and read files / access directories created by other users.

Files and folder permission can be check using multiple way such as Powershell (Get-Acl) or just use explorer Security tab. Logs files are created by SYSTEM processes, and are made writable to the user. Arbitrary file creation can be achieved by abusing the log file creation: an unprivileged user can replace these log files by pseudo-symbolic links to arbitrary files. When a log is generated, a privileged hide.me VPN process will create the log file and set its access rights, offering write access. The permission for hide.me VPN as in following screenshot:

![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/hide_vpn/acl_1.png "ACL issue")

We can observe the file operations by using Process Monitor tool. Once we find the file operations performed on user-controllable files & directories, we will need to figure a way to exploit these operations. James Forshaw came up with several techniques to abuse Windows filesystem and path resolution features, and even released a Symbolic Link Testing Toolkit for researchers to use as Proof-of-Concept. There are three types of symbolic links you can access from a low privileged user:
- Object Manager Symbolic Links
- Registry Key Symbolic Links 
- NTFS Mount Points

In this case, I used NTFS junctions to exploit the vulnerability. Junctions are an NTFS feature that allows directories to be set as mount points for a filesystem, can also be set up to resolve to another directory (on the same or another filesystem). Junctions can be created by unprivileged users. It works on any volumes and we can redirect from one to another. An existing directory can be turned into a junction if you have write access to it, but it must be empty. NTFS junctions are implemented with reparse points and, while built-in tools won't let you do it, they can be made to resolve to arbitrary paths with an implementation that sets custom reparse points. In our case we used CreateMountPoint to create junction. Example of NTFS junctions by @clavoillotte:

![Screenshot broadcast](https://offsec.almond.consulting/images/intro-to-file-operation-abuse-on-Windows/ntfs_junctions.png "NTFS junctions")

Exploitation
------------
To exploit the issue, unprivileged user can perform the below behavior:
- Delete all files in "C:\Users\<user>\AppData\Roaming\Hide.me\" folder
- Create a mount point from "C:\Users\<user>\AppData\Roaming\Hide.me\" that points to "C:\"
- The hide.me VPN service can be restarted or wait until the computer gets rebooted. Once rebooted / restarted, it will create a mount point to "C:\". 

This results in a vulnerability that can be exploited to create arbitrary files with arbitrary content. We can use the symlink technique to divert a specific log file  to create an arbitrary file with an attacker-chosen name, e.g. a DLL in the program's directory. Example:

![Screenshot broadcast](https://offsec.almond.consulting/images/intro-to-file-operation-abuse-on-Windows/product_x_exploit_junction.png "NTFS Junction Exploitation")

The exploitation can be achieved as in following steps:

Delete all files inside the folder. Powershell command used to delete files:
```
Remove-Item -Force "C:\Users\<user>\AppData\Roaming\Hide.me\*"
```
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/hide_vpn/delete_2.png "File deletion")

Using CreateMounPoint tool to mount to "C:\". Once created, we have to dump the reparse by using DumpReparsePoint.exe tool. Command used:
```
CreateMountPoint.exe "C:\Users\<user>\AppData\Roaming\Hide.me\" "C:\"

DumpReparsePoint.exe "C:\Users\<user>\AppData\Roaming\Hide.me\"
```
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/hide_vpn/mount_3.png "NTFS Junctions")

Successful exploitation will arbitrary create the log file to "C:\". We can see the ACL set to "Authenticated Users" with Modify permission which means allows authenticated users to perform modifications on the file created. We could exploit the vulnerability in multiple way to achieve privilege escalation. Since we have permission on the log file, we can replace the log file with new executable. 
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/hide_vpn/file_created_4.png "Successful exploitation")

So now assuming that we got our executable place on "C:\". We have to figure out how are we going to run the "Program.exe". One way to abuse the privilege escalation is by leveraging Windows Host Services (svchost.exe). The service trying to load non-exisiting executable from "C:\" with file named "Program.exe". I have reported this issue previously to MSRC however it didn't meet the bar of servicing due to required to have Admin access to dropped files into "C:\". So here we leverage this issue to elevate our process during boot up. We crafted a simple C code to spawn CMD.exe during boot :)
```
#include <Windows.h>

int main()
{
	WinExec("cmd", 0);
	return 0;
}
```

I used Process Monitor (Sysinternals tool) to observe boot-up process and filter it to "C:\Program.exe". 
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/hide_vpn/load_file.png "Loaded")

By right svchost.exe will execute our "Program.exe" and this executable will execute CMD.exe. We can see this process being called as "cmd" (refer back to our code). This is where we can achieve privilege escalation :)
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/hide_vpn/privesc.png "Privilege escalations")


Disclosure timeline
-------------------
The vulnerability was reported in August 2020. Timeline of disclosure:

- 2020-08-02 - Asking for proper channel to report security vulnerability via support channel.
- 2020-08-02 - Support replying to sent all the relevant details to them, coordinated via support. Vulnerability details sent to them. 
- 2020-08-11 - Seek for update. No response from vendor.
- 2020-08-15 - Seek for update again. No response from vendor.
- 2020-08-20 - Vendor replied saying the vulnerability has been fixed on latest version 3.5.0. They awarded bounty for the issue reported.
- 2020-08-26 - Vulnerability writeup

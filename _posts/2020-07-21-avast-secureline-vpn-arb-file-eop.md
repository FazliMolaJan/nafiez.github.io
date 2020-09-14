---
layout: post
title:  "CVE-2020-25289 - AVAST SecureLine VPN - Arbitrary File Creation Vulnerability"
date:   2020-07-21 11:43:00 +0800
categories: [security, arbitrary file, eop]
---

Overview
--------
Avast SecureLine VPN is an application that enables you to connect to the internet via secure Avast VPN servers using an encrypted tunnel to protect your online activity from eavesdropping. Avast SecureLine VPN can be used any time you want to connect to the internet with extra security and privacy. This is especially recommended when you are connected to a public or unsecured wireless network.

AVAST SecureLine VPN Service creates a log file in "C:\ProgramData\AVAST Software\SecureLine\log\" by default. The files are created, accessed and manipulated by privileged (SYSTEM) processes of AVAST SecureLine service. The log and folder have permissive access rights that allow unprivileged users to add/remove files and change properties. Affected version of AVAST SecureLine VPN 5.5.522.0. Version below might be affected too, but untested. The vulnerability has been fixed on version 5.6.4982.470.

Vulnerability Analysis
----------------------
A process running with higher privileges (e.g. SYSTEM) and perform operations on files like the rest of the processes that has access to user-controlled files or directories without restrictions could lead to a security issue. The operations could be abuse by leveraging the privileged process to perform unwanted activity. This issue well-known as logical vulnerability. Most privilege programs will not manipulate unprivileged user-access files indirectly, however most of it perform operations on a files that located somewhere a user can have access to it. Here's a list of interesting locations based on this [blog](https://offsec.almond.consulting/intro-to-file-operation-abuse-on-Windows.html):
- The user's own files & directories, including its AppData and Temp folders, that some privileged process may use if you're lucky or running an AV
- The Public user's files & directories: idem
- Directories created in C:\ with default ACL: by default, directories created at the root of partitions do have a permissive ACL that allows write access for users
- Subdirectories of C:\ProgramData with default ACL: by default, users can create files and directories, but not modify existing ones. This is often the first place to look at.
- Subdirectories of C:\Windows\Temp: by default, users can create files and directories, but not modify existing ones and read files / access directories created by other users.

Files and folder permission can be check using multiple way such as Powershell (Get-Acl) or just use explorer Security tab. Logs files are created by SYSTEM processes, and are made writable to the user. Arbitrary file creation can be achieved by abusing the log file creation: an unprivileged user can replace these log files by pseudo-symbolic links to arbitrary files. When a log is generated, a privileged AVAST SecureLine VPN process will create the log file and set its access rights, offering write access. The permission for AVAST SecureLine as in following screenshot:

![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/acl.png "ACL issue")

We can observe the file operations by using Process Monitor tool. Once we find the file operations performed on user-controllable files & directories, we will need to figure a way to exploit these operations. James Forshaw came up with several techniques to abuse Windows filesystem and path resolution features, and even released a Symbolic Link Testing Toolkit for researchers to use as Proof-of-Concept. There's a plenty of technique that can be use to abuse this kind of vulnerability, including:
- NTFS Junctions
- Hard Links
- Object Manager Symbolic Links
- Opportunistic Locks

There are three types of symbolic links you can access from a low privileged user:
- Object Manager Symbolic Links
- Registry Key Symbolic Links 
- NTFS Mount Points

In this case, I used Object Manager symbolic links to exploit the vulnerability. As mentioned by @clavoillotte in his [blog](https://offsec.almond.consulting/intro-to-file-operation-abuse-on-Windows.html), an unprivileged users can create symbolic links in Windows Object Manager which, as its name suggests, manages objects such as processes, sections and files. The Object Manager uses symlinks, e.g. for drive letters and named pipes that are associated with the corresponding devices. Users can create object symlinks in writable object directories such as \RPC CONTROL\, and these symbolic links can point to arbitrary paths – including paths on the filesystem – whether that path currently exists or not. Object symlinks are especially interesting when combined with an NTFS junction. Indeed, as an unprivileged user we can chain a mount point that resolves to the \RPC Control\ directory with an object manager symlink in that directory. Example:

![Screenshot broadcast](https://offsec.almond.consulting/images/intro-to-file-operation-abuse-on-Windows/object_manager_symbolic_links.png "Object Manager")

Exploitation
------------
To exploit the issue, unprivileged user can perform the below behavior:
- Delete all files in "C:\ProgramData\AVAST Software\SecureLine\log\"
- Create a pseudo-symlink named "C:\ProgramData\AVAST Software\SecureLine\log\vpn_engine.log" that points to "C:\Windows\System32\pwned.dll"
- The SecureLine service can be restarted or wait until the computer gets rebooted. Once rebooted / restarted, it will create an arbitrary file on the "C:\Windows\System32\" folder.

This results in a vulnerability that can be exploited to create arbitrary files with arbitrary content. We can use the symlink technique to divert a specific log file  to create an arbitrary file with an attacker-chosen name, e.g. a DLL in the program's directory. Example:

![Screenshot broadcast](https://offsec.almond.consulting/images/intro-to-file-operation-abuse-on-Windows/product_x_exploit_symlink.png "Symlink Exploit")

The exploitation can be achieved as in following steps:

Delete all files inside the folder. Powershell command used to delete files:
```
Remove-Item -Force "C:\ProgramData\AVAST Software\SecureLine\log\*"
```
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/delete_logs.png "File deleted")

Using CreateSymlink tool by James Forshaw to create a symbolic link. Command used:
```
CreateSymlink.exe "C:\ProgramData\AVAST Software\SecureLine\log\vpn_engine.log" C:\Windows\System32\pwned.dll
```
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/createsymlink.png "Create Symbolic Link")

Successful exploitation will create a file in our target folder
![Screenshot broadcast](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/success_arb_create.png "Successful exploitation")

I have written a Proof-of-Concept that exploit this vulnerability and the messy code can be found [here](https://raw.githubusercontent.com/nafiez/Vulnerability-Research/master/avast_secureline_vpn_poc.c). To compile the exploit, it is required compile it with [Symbolic Link Toolkit](https://github.com/googleprojectzero/symboliclink-testing-tools) by James Forshaw. However the PoC required an Admin rights in order to run since the start and stop service are required an Admin privilege to perform so. I leave the rest to you to figure another way to exploit this issue :). Following are the demo video (GIF):

![Demo POC](https://raw.githubusercontent.com/nafiez/nafiez.github.io/master/static/img/avast-pwned-eop.gif)

Disclosure timeline
-------------------
The bug was found few days before I fly to South Korea for POC conference and only tooks me few hours to find the bug and exploit it. The vulnerability was reported back in 2019, on November. I follow the standard coordinated disclosure by AVAST. Timeline of disclosure

- 2019-11-02 - Reported to AVAST Security Team (via email)
- 2019-11-05 - Vendor response that they will look into the problem.
- 2020-01-23 - Follow up with vendor. They ack and informing there's a changes happened internally so it might take some time to get a fix.  
- 2020-03-02 - Follow up again with vendor. They focus on AV released first then follow by the rest of their products.
- 2020-05-18 - Follow with vendor. They have been informing the patch for VPN product will be release end of May. However it didn't go well. So shift to June release.
- 2020-06-26 - Follow up again with vendor. Vendor informed me that the fixed version will be release on Monday (which is June 29, 2020). 
- 2020-07-14 - Verify the fixed has been shipped on version 5.6.4982.470. Follow up with vendor for CVE and blog post. Vendor said no CVE assign, however can request myself. For blog post, vendor good with it since the fixed has been shipped. The fixed version can be found [here](https://bits.avcdn.net/productfamily_VPN/insttype_PRO/platform_WIN/installertype_ONLINE/build_RELEASE/cookie_mmm_scl_003_999_a4g_m)
- 2020-07-22 - CVE requested. Pending for CVE.
- 2020-09-14 - CVE assigned, CVE-2020-25289. 

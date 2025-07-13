+++ 
draft = false
date = 2020-01-11T22:47:27+10:00
title = "Intro to Windows Organs"
description = ""
slug = ""
authors = ["Joe goldberg"]
tags = []
categories = ["Windos Internals"]
externalLink = ""
series = []
+++

Hey traveller,
Over the years, Windows has evolved from a simple graphical shell into a complex operating system that powers everything from personal laptops to global enterprise infrastructure. As of February 2024, Windows dominates the global desktop operating system (OS) market with a 72% share. Over 90% of malware is designed to target Windows systems. Understanding how Windows works is a crucial pillar in knowing how to respond to threats or emulate adversaries. In simple terms, if you don't understand what you're defending, you're just watching — not guarding. 


<img src="/images/windows-arti/who-will-win.jpg" alt=""
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

## Lord of the Rings

The image below explains the concept of CPU privilege rings, including user mode and kernel mode, which apply across all operating systems.
Rings are different privilege levels that the CPU uses to distinguish what a piece of code is allowed to do. Ring 0 and Ring 3 are the most important for now:
- Ring 0 (Kernel Mode) has full access to the system.
- Ring 3 (User Mode) is restricted and used for applications

Rings 1 and 2 are rarely used in modern operating systems. Some OS designs may use them for specific tasks or drivers, but most operating systems skip these rings entirely.

<img src="/images/windows-arti/rings.jpg" alt=""
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">


## Ring 3 (User mod) VS Ring 0 (kernel mod) 

Ring 3, also known as user mode, is where us peasants live with the least privileges. We can’t do much without asking for permission from Ring 0, the boss. Basically, all your applications like Notepad, WannaCry, or even your browser — run in user mode. On the other hand, Ring 0, aka kernel mode, has the highest level of privileges. Any code running in kernel mode can execute any CPU instruction and access any part of memory — this is where things like rootkits operate. If code in Ring 0 crashes or gets compromised, the whole system can go down with it. 

So, whenever you try to perform any privileged action (remember, we're just peasants), we have to ask the kernel to do it on our behalf. The kernel checks whether we’re allowed to do that action and only then proceeds.

<img src="/images/windows-arti/rings1.jpg" alt=""
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

## what is a DLL VERY FAST ? 

A DLL (Dynamic-Link Library) is a file that contains code and data that can be used by multiple programs at the same time in Windows. It makes it easier for developers to import and use functions that are already written, instead of coding everything from scratch. 

For example, instead of writing your own function to open a file, you can just use the [CreateFileA](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea) function, which is exported from a DLL called kernel32.dll.
 

Just a quick note: to use any function that's exported by a DLL, the DLL that contains the function must be loaded into memory. When you boot your fancy Windows OS, it automatically loads a few essential DLLs , without them, Windows wouldn't function. These include ntdll.dll, kernel32.dll, advapi32.dll, user32.dll, and others.

## Architecture of Windows

<img src="/images/windows-arti/windows-arti.jpg" alt=""
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

So Bob the Builder, since you’re here, Let’s talk about the architecture of Windows in general.

But before we dive in, here are a few definitions to clarify what we’re dealing with:

- User processes: Basically, these are processes that do the work on your behalf when you take any action — like opening a file. Think of application processes like chrome.exe or notepad.exe.
- Service processes: These represent background services installed on your machine, like an SQL database, FTP server, or any custom service you’ve installed.
- System processes: These are used by the system itself to perform critical tasks. A great example is lsass.exe, which is responsible for authenticating users.
- Subsystem DLLs: These are essential DLLs provided by Windows, like kernel32.dll and user32.dll. (⚠️ Note: ntdll.dll is not considered a subsystem DLL — it lives in its own universe.)
- csrss.exe process: To simplify it. It’s a critical system process. If your favorite color is blue, try killing the process… 
- ntdll.dll: This is the only one allowed to speak directly to the majestic kernel-mode.
It contains Native API functions that allow communication between user mode and kernel mode. (Don’t worry you’ll fully understand Native APIs once I walk you through an example.)

     
     
And to make things easier for me (and hopefully less painful for you), I won’t explain everything raw and hope that GRC people say “I understand.”
They’ll just say “Compliant” anyway. 


Let’s say you open a file called CISSP_IS_NOT_technical.txt.

1- Your user process (for example, notepad.exe) wants to open the file. It calls the [CreateFileA](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea) function, which is exported by kernel32.dll (a subsystem DLL). (And please, don’t panic. Yes, the function is called "CreateFileA" even if you're just reading a file. That’s because the function’s behavior depends on the dwDesiredAccess parameter, which you can set to just read the file.)

2- kernel32.dll doesn’t talk to the kernel directly. Instead, it passes the request to ntdll.dll

3- ntdll.dll finds the equivalent Native API function, which is NtCreateFile. It sets up all the required arguments in a very low-level structure format (you don't see this in normal WinAPI calls).

4- ntdll.dll then performs a syscall. This is how it translates the function request from user mode into a kernel-mode request.

(I'll just touch the surface of the kernel to keep things simple )

5- the Windows kernel kicks in. It performs the necessary checks , like whether the peasant has permission to open the file, and then grants or denies access.


In Summary :

kernel32.dll → passes to ntdll.dll → calls NtCreateFile → syscall → kernel handles the rest.

Note: Sometimes you’ll see that a system process skips the subsystem DLL step and communicates directly with ntdll.dll to use a Native API.

But remember — only ntdll.dll is allowed to perform the syscall.
If any other code tries to perform a syscall directly and communicate with the kernel, that’s considered an Indicator of Compromise (IOC).

We’ll dive deeper into syscalls and how they work in future blog posts.


Anyway, that's enough typing for me . I'm going to watch _YOU_ Season 5.

<img src="/images/windows-arti/joe.gif" alt="joe goldberg"
       style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">
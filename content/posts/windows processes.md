+++ 
draft = true
date = 2025-07-11T22:47:27+10:00
title = "windows Processes"
description = ""
slug = ""
authors = ["Raymond Reddington"]
tags = []
categories = ["Windows Internals"]
externalLink = ""
series = []
+++


Hi there continue the juorny to into windows internals gonna dive a little bet deep into processes.

Why are we diving into processes? Because it's fun (at least for me). Understanding processes is important for anyone working in cyberspace — unless you're a GRC person (in which case, why on earth are you looking at Windows processes? You probably barely know how to use Task Manager).

But seriously, knowing how processes work is crucial for detecting malicious activity. In fact, you can solve entire cases using just memory forensics. Tools like Volatility allow you to analyze memory dumps or VM snapshots to identify what processes were spawned during an incident; no disk forensics needed.  

Imagine you're managing a company where each department has a head or boss, and under that boss are employees. In this analogy, the boss or head of the department represents a process, and the employees are the threads that do the actual work.

A process is a set of resources used to execute a program — typically, this means an executable file (EXE). To avoid confusion later on, if you run a script such as malware.ps1 (PowerShell) or malware.py (Python), the system will actually start the corresponding interpreter — powershell.exe or python.exe. The threads within that process then execute the script code. 

## Process Components 

- virtual address space: 

- executable image ( file exe or dll or ps1 etc) 

- handle table 

- access token ( security token) 

-  threads  ( at least one thread of excution often called the primary thread (I like to call it main thread) ) 

- a unique process identifier (PID)

- environment variables 

-  a priority class

- minimum and maximum working set sizes


## Process Environment Block (PEB)

There are three important structures associated with a process:
- PEB (resides in the user mod)   
- KPROCESS (resides in kernel mod)
- EPROCESS (resides in kernel mod)

TL;DR — What is a structure?
A structure is a predefined layout in memory that holds related data — in this context, it stores key information about a process.

For now, I’ll explain only the PEB, so I can proudly say “I’ve conquered the peasant mode.”
And believe me , you do not want to see the map that connects all the important structures together now. Your eyes will burn.

 Every process that spawns on your system has a PEB structure — except for System.exe, because it was sent down by the almighty kernel to oversee the land of the peasants ( I will talk more about the process in the windows core processes section ). 

The Process Environment Block (PEB) is a data structure in Windows that contains information about a process such as its parameters, startup information, allocated heap information, and loaded DLLs . It’s primarily used by the Windows loader and user-mode components to manage the process during execution. You can think of it as the process’s “personal file”, recording everything the OS needs to keep track of that process from where it came from to what it's carrying.



## Windows Core Processes






## Process Types 
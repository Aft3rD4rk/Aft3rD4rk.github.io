+++ 
draft = false
date = 2025-07-11T22:47:27+10:00
title = "Hooking apis where ever I please"
description = ""
slug = ""
authors = ["Joe goldberg"]
tags = []
categories = ["Malware"]
externalLink = ""
series = []
+++


Today we're gonna hook windows api functions

<img src="/images/hooking/hook.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

API hooking is a technique that modifies the behavior of a Windows API function. For example, you can change the functionality of MessageBox to display a different message.

Hooking is a two-edged sword in Windows. It’s used by EDRs and AVs to intercept API calls and inspect them before they even reach the kernel (essentially a userland defensive mechanism). At the same time, it can also be abused by attackers as a technique to bypass EDR/AV detection by altering the functionality of an API, making it do something else.


 

There are three types of hooking in Windows:

- Trampolines
- Inline Hooking
- Hardware Breakpoint Hooking (I’ll cover this in the future. has complately diffrent implementation).


So before we explore Trampolines and Inline Hooking, let me drop this image here so we can compare the difference between a normal flow of execution (no hooks) and a hooked scenario

<img src="/images/hooking/nohooks.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

     1- Your code jumps to the address of the API function to execute it. 
     2- It then returns to where you left off in your code (thanks to the RET assembly instruction).

<H1> Trampolines </H1>

<img src="/images/hooking/Trampolines.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

Trampoline hooking works by modifying the beginning of a function so that execution jumps to another address within the process’s address space. This is typically done by writing a small piece of code (shellcode) at the start of the target function. When the hooked function is called, this trampoline executes first (since it sits at the function’s entry point) and redirects execution to a different function address of our choosing.then it returns to to our code.


<H1> Inline Hooking </H1>

<img src="/images/hooking/Inline-Hooking.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

nline Hooking is similar to Trampoline Hooking, but the difference lies in the execution flow. With inline hooking, execution eventually returns to the original (hooked) function, allowing normal execution to continue after the custom code runs.

While it is more difficult to implement and maintain, inline hooking is generally more stealthy and efficient than trampoline hooking. And, as anyone who knows me can guess, I’m going to implement Inline Hooking because it’s much more fun. 

Here’s the flow:

    1- Our code jumps to the address of the API function to execute it.
    2- The API function then proceeds to jump to the hooked function (proxy function).
    3- The proxy function executes its own code.
    4- The proxy function jumps back to the original API function to continue normal execution.
    5- Once it finishes, control returns to our code where it left off.

Step 4 is the key difference from trampoline hooking: execution resumes in the original function instead of jumping back to the caller immediately.


<H1> Coding and suffering starts from here </H1>

<img src="/images/hooking/to-do-list.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">



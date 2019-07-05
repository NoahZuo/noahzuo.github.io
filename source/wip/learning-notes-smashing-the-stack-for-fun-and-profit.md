---
title: Learning Notes about Smashing the Stack for Fun and Profit
tags: 
 - Learning notes
 - Coursera
 - Cyber Security
category: Learning notes
description: These are some learning notes about Smashing the Stack while taking the course `Introduction to Cyber Security` on Coursera. 

---


## Basic Introduction about Stack


 - How a process is organized in memory? 

   Processes are divided into three regions: Text, Data and Stack. 
   ![Process Memory Regions]("Process Memory Regions.png")	

	- Text Region
	
	  The text region is fixed by the program and includes code (instructions) and read-only data. This region corresponds to the text section of the executable file. This region is normally marked read-only and any attempt to write to it will result in a segmentation violation. 
	
	- Data Region
	
	  The data region contains initialized and uninitialized data. Static variables are stored in this region. The data region corresponds to the data-bss(Data segment space) of the executable file. 
	
	  The region's size can be changed with the `brk()` system call. If the expansion of the bss data or the user stack exhausts available memory, the process is blocked and is rescheduled to run again with a larger memory space. New Memory is added between the data and stack segments. 
	
	- Stack Region
	
	  A stack is a contiguous block of memory containing data. A register called the stack pointer(`SP`) points to the top of the stack. The bottom of the stack is at a fixed address. Its size is dynamically adjusted by the kernel at runtime. The CPU implements instructions to `PUSH` onto and `POP` off of the stack. 
	
	  
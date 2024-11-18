---
title: Learning Notes about Smashing the Stack for Fun and Profit
tags: 
 - Learning notes
 - Coursera
 - Cyber Security
category: Learning notes
date: 2019-07-8 16:00:22
description: These are some learning notes about Smashing the Stack while taking the course `Introduction to Cyber Security` on Coursera. 

---


## Basic Introduction about Stack


 - How a process is organized in memory? 

   Processes are divided into three regions: Text, Data and Stack. 
   ![Process Memory Regions](Process Memory Regions.png)	

	- Text Region
	
	  The text region is fixed by the program and includes code (instructions) and read-only data. This region corresponds to the text section of the executable file. This region is normally marked read-only and any attempt to write to it will result in a segmentation violation. 
	
	- Data Region
	
	  The data region contains initialized and uninitialized data. Static variables are stored in this region. The data region corresponds to the data-bss(Data segment space) of the executable file. 
	
	  The region's size can be changed with the `brk()` system call. If the expansion of the bss data or the user stack exhausts available memory, the process is blocked and is rescheduled to run again with a larger memory space. New Memory is added between the data and stack segments. 
	
	- Stack Region
	
	  A stack is a contiguous block of memory containing data. A register called the stack pointer(`SP`) points to the top of the stack. The bottom of the stack is at a fixed address. Its size is dynamically adjusted by the kernel at runtime. The CPU implements instructions to `PUSH` onto and `POP` off of the stack. 
	
## Introduction to Some Important Register

### `SP` 

The stack pointer(`SP`) is implementation dependent. It may point to the last address on the stack, or to the next free available address after the stack. 

### `LB` or `FP`

A local base pointer(`LB`), which sometimes is also referred to as a frame pointer(`FP`), points to a fixed location within a frame. In principle, local variables could be referenced by giving their offsets from SP. However, as words are pushed onto the stack and popped from the stack, these offsets change. 

As a result, many compilers use the register, `FP`, for referencing both local variables and parameters because their distances from FP do not change with PUSHes and POPs. 

On Intel CPUs, BP(EBP) is used for this purpose. On the Motorola CPUs, any address register except A7(the stack pointer) will do. Because the way our stack grows, **actual parameters have positive offsets and local variables have negative offsets from FP**. 



## Something Needs to Be Done When Calling a Function

Actually, this is called the **procedure prolog**. 

1. Save the previous `FP`
	We need to restore previous stack at procedure exit. 
	```x86asm
	pushl %ebp
	```
	
2. Copy `SP` to the `FP`
	We need to create new `FP`
	```x86asm
	movel %esp, %ebp
	```
3. Advance `SP`to reserve space for the local variables
	Space is reserved for local variable. For function like this: 
	
	```cpp
	void function(int a, int b, int c)
	{
		char buffer1[5]; 
		char buffer2[10];
	}
	```
	Consider memory align, we need to reserve 20 bytes for buffers. 
	
	```x86asm
	subl  $20, %esp
	```



## Buffer Overflow and How it Works



Consider code like this: 

```c
void function(int a, int b, int c)
{
    char buffer1[5]; 
    char buffer2[10]; 
    int* ret; 
    
    ret = buffer1 + 12; 
    (*ret) += 8; 
}

int main()
{
    int x; 
    x = 0; 
    function(1, 2, 3); 
    x = 1; 
    printf("%d\n", x); 
    
    return 0; 
}

```

This program would print "0" in the command window(for a 32-bit system). But how does this work? 

Let's first take a look at the stack: 

![Function Stack](buffer stack.png)

It is clear that `buffer1` takes 8 bytes of space. And pointer `sfp` is a 4-byte pointer. Thus `ret = buffer1 + 12; `  would get a correct `ret` address. 

Eventually, `(*ret) += 8; `would skip `x = 1; ` statement in the main function. 

And this is really cool! We can influence code execution outside the current stack. 



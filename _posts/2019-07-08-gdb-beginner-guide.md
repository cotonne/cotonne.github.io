---
layout: post
title:  "GDB - A beginner guide"
date:   2019-07-14 08:42:27 +0100
categories: gdb 
---

I have started to learn about reversing of binaries (like ELF). To debug and understand some of them,
GDB is a basic but quite powerful tool This article is for beginners to learn how to use it.
It is a convenient tool if you want to debug (GDB stands for GNU Debugger). If you want to do reversing
and binary exploitation, it is also a great tool to master.

In this article, you will find the most useful command to start doing reversing and binary exploitation.

# Running a program

The first step is to start the program. The most basic approach is just to call GDB with the program name:

    $ gdb ./my.elf

Once you start GDB, you can define breakpoints, view code in assembly, ... from the GDB command line:
To directly start the program, just type **run**:

    > run

Starting this way, GDB might modify the stack by adding new environment variables.
You can also debug a runnning process:

    $ gdb -- pid <PID>
    # or
    $ gdb
    > attach <PID>

    # Remote
    $ gdbserver host:port program
    # Local
    $ gdb
    > target remote localhost:12345

To provide arguments, you can do it at the beginning or directly from GDB:

    $ gdb --args program arg1 ...

You can also pipe content to the program

    $ gdb ./my.elf
    > run < $(python -c 'print("A"*10)')

# Debugging a program

One of the essential aspects is to control the program. 

## Breakpoints

Breakpoints are a way to stop the program in a given position.
Once the program is stopped, you can examine and change the program state.
For example, you can view and modify:
 - Registers
 - Heap
 - Stack
 - BSS
 - Next program instruction 
 - ...

You can defined breakpoints like this:

    # break at a function
    > break main
    # shorter version
    > b main
    # break at a program address
    > b *0x00000

To view the currently defined breakpoint, you can use **info**. **info** is a useful command 
to view a lot of information about your program, its state and about the current environment.

    > info breakpoints

And to delete, it is as simple as that:

    > delete breakpoints 2

In the case of reversing, some programs might use anti-debugging technics to change their
behavior. With breakpoints, they will get the clock time twice and see if the duration
is less than some hundred milliseconds for example.

The most common example is the use of **RDTSC** (ReaD TimeStamp Counter) which is an 
assembly command. It reads the processor time.

## Move in the program

Once the program is stopped, you can control the flow of execution. You can do it with **step**
and **next**.

    > stepi
    > step
    > nexti
    > next

When you call **stepi**, you are following the program. If the program does a **CALL** (call a function),
you will be inside the function. With **nexti**, you are going to the next line of the program.

    0x080485e9 main+69 call   0x80484a0 <strlen@plt> # current position
    0x080485ee main+74 mov    %eax,%edx

    > nexti

    0x080485e9 main+69 call   0x80484a0 <strlen@plt>
    0x080485ee main+74 mov    %eax,%edx              # current position


If you use **stepi** and go into a function you don't want, you can "step out" with **step**.

    0x080485e9 main+69 call   0x80484a0 <strlen@plt>
    > stepi
    0x080484a0 ? jmp    *0x8049ff0
    > step
    0x080485ee main+74 mov    %eax,%edx

**nexti** goes by one machine instruction. **next** goes to next source line.

## Automating (GDB macros)

After a step, you might be interested to have a view of the state of the program.
Sometimes, you will have to do tens of steps. Soon, you will want to automate actions

    (gdb) def n
    >next
    >info registers
    >x/10i $eip
    >end
    (gdb) n
    eax            0x804a020           134520864
    ecx            0x247911ac          611914156
    ...
    => 0x80485bc <main+24>:    mov    WORD PTR [esp+0x426],0x1
       0x80485c6 <main+34>:    mov    DWORD PTR [esp+0x41c],0x0
       0x80485d1 <main+45>:    jmp    0x804861f <main+123>
    ...
    (gdb) 

## Viewing the last call of a loop (skip breakpoint)

You might want to analyze the behavior of a loop. However, loops can run multiple times.
So you might want to skip some part and only review the state of the program after some execution. GDB gives you the possibility to stop the program after hitting a breakpoint a
given number of times.

    > ignore <breakpoint id> 10

## Continue

Once you have seen enough things, you can resume the execution.

    > continue

The execution will continue until the program reaches a breakpoint, an exception or terminate.

# Viewing state of the program

## Registers

Registers are small bytes of memory. For a classical X86 architecture, you will have:
 - EAX, EBX, ECX, EDX : general purpose registers used for computing. They can have extended meanings (like EAX is for Accumulator, ECX for Counter).
 - EBP: Base Pointer of the stack. Inside a function, it doesn't move. So, it can be used as a reference
 - ESP: Current position of the pointer of the stack. Each time you push or pop data, it moves
 - ESI/EDI: Source Index/Destination Index. Useful for copying block of memory (like MOVSB)
 - EIP: Instruction Pointer. What will be the next instruction?
 - EFlags: use to indicate the state of the program. 

To view registers, just type:

   > info registers

## Stack

In the stack, you will find:
 - The return address of the calling function
 - Previous position of the Base Pointer calling function
 - Arguments
 - Local variables
 - Temporary data (PUSH/POP)

So, knowing how to view the stack is essential:

    > x/32wx $esp
    > x/32c  $ebp - 10
    > x/10i  0x0000

Let's decompose it:

 - **x** is for "examine"
 - Then, the number after **/** indicates the number of bytes you want to proceed
 - The last letters describe how the data should be viewed (as an integer, unsigned integer, float, program, ...). More information [here](ftp://ftp.gnu.org/old-gnu/Manuals/gdb/html_chapter/gdb_9.html#TOC56)

You can modify the stack with:
    
    > set {int}0x80000 = 4

# Disassembly

They are two main sets of instructions: **intel** and **att**. Depending on your preferences, you can switch by using this command:

    > set disassembly-flavor intel

You can disassemble almost every byte as a program instruction:

    > disassemble main
    > x/32i $eip

You can also disassemble any address. Why? Because shellcode can be stored in the heap or in the stack. This com

    > x/32i *0x00000 
 
# Conclusion

In this article, we have seen how to start a program, walk through it, viewing and modifying its state.
In the next article, we will have a more concrete example of this usage.

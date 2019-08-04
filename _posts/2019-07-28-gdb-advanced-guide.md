---
layout: post
title:  "GDB - An advanced guide"
date:   2019-07-08 08:42:27 +0100
categories: gdb 
---

In the [previous post](https://cotonne.github.io/gdb/2019/07/08/gdb-beginner-guide.html),I have talked
about basic commands to navigate in a running program with GDB. In this post, we will see how we can
alter the execution of a running program. I will also present advanced techniques.

By altering, I mean changing the normal behavior by rewriting part of the memory, changing value of
a register or directly moving to a specific address. You want the program to take a branch? You can do
 it. You can also dynamically rewrite program text (this is not different than writing in the memory).

# Altering the flow of the program

A full description can be found in the [chap 13 of the GDB manual](ftp://ftp.gnu.org/old-gnu/Manuals/gdb/html_chapter/gdb_13.html).

## ASLR

[ASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization) stands for 
"Address Space Layout Randomization". It is a mechanism designed to make exploitation
of memory vulnerability harder by making address random. So, writing a payload with 
fixed memory become (almost) useless. 

With a base address and the shift of the position, you can directly calculate the 
address you want to overwrite. If not, you will have to search for it. To know the 
base address, you can use:

    > info files
    ...
    Entry point: 0x80484d0

## Update memory

Updating the memory is really simple

   > set *(0x08001234)=0x41414141

Memory are divided in different parts: program, stack, heap, ... To know which parts
you are modifying, you call the following command:

    > info proc mappings
    Mapped address spaces:

	Start Addr   End Addr       Size     Offset objfile
	 0x8048000  0x8049000     0x1000        0x0 /home/buddy/anElf
	 0x8049000  0x804a000     0x1000        0x0 /home/buddy/anElf
	 0x804a000  0x804b000     0x1000     0x1000 /home/buddy/anElf
	0xf7dba000 0xf7f93000   0x1d9000        0x0 /lib/i386-linux-gnu/libc-2.29.so
	0xf7f93000 0xf7f94000     0x1000   0x1d9000 /lib/i386-linux-gnu/libc-2.29.so
	0xf7f94000 0xf7f96000     0x2000   0x1d9000 /lib/i386-linux-gnu/libc-2.29.so
	0xf7f96000 0xf7f98000     0x2000   0x1db000 /lib/i386-linux-gnu/libc-2.29.so
	0xf7f98000 0xf7f9a000     0x2000        0x0 
	0xf7fce000 0xf7fd0000     0x2000        0x0 
	0xf7fd0000 0xf7fd3000     0x3000        0x0 [vvar]
	0xf7fd3000 0xf7fd4000     0x1000        0x0 [vdso]
	0xf7fd4000 0xf7ffb000    0x27000        0x0 /lib/i386-linux-gnu/ld-2.29.so
	0xf7ffc000 0xf7ffd000     0x1000    0x27000 /lib/i386-linux-gnu/ld-2.29.so
	0xf7ffd000 0xf7ffe000     0x1000    0x28000 /lib/i386-linux-gnu/ld-2.29.so
	0xfffdd000 0xffffe000    0x21000        0x0 [stack]

## Update register

You can update the value of the register the same way. For example, to set the value of 
eax to 0, just do:

    > set $eax = 0


If you want to change a flag, which is a one-bit element of the eflags register, you can use
boolean operation. For example, to change ZF, the 6th bit of eflags:

    set $eflags |= (1 << 6)   # set ZF bit in EFLAGS
    set $eflags &= ~(1 << 6)  # clear
    set $eflags ^= (1 << 6)   # toggle
    i r $eflags               # view the state of flags

## Going somewhere else

Sometimes, you might want to go directly to a given position of the program. It is really 
easy to modify EIP:

    jump *0x000A

# Advanced techniques

## Analyzing behavior without GDB

Using a debugger can be really hard. A lot of commands to remember, you need to be extremely 
careful not to break everything. One solution is to use **strace**, an UNIX command. This command
captures external calls of your program. So you can see which file is opened, which syscall is made, ...

    strace ./my_program

This technique can be easily defeated with this section:

```C
    if (ptrace(PTRACE_TRACEME, 0, 1, 0) == -1) {
        // The program has been called with strace ./my_program
    }
```

## Jumping between child process

In case of fork, the program will split in two: the parent and the child. So you might have two
processes to debug. With GDB, you can choose to control the child, the parent or both!

   > set follow-fork-mode parent
   > set follow-fork-mode child
   > set follow-fork-mode off    # hold both processses under the control of gdb

# Conclusion

With GDB, you can do anything you want to the program. Stop it, analyze it, modify it, ...
However, malware authors and commercial companies have tricked to play to prevent you from
reversing their program. This will be the subject of the next article!
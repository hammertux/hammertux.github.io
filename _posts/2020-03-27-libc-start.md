---
title: How glibc calls main()
---

(g)libc is the caller of main in C programs. If you're curious about what is happening, take a few minutes to read on and if you have gdb/objdump, you can follow along :)

_Note: This is valid for Linux X86-64!_


## Starting simple

If we take a simple C program: 

```C

#include <stdio.h>

int main(int argc, char **argv)
{
		printf("Hello World!\n");
		return 0;
}

```

and compile this with ``` gcc -o test -ggdb main.c ``` we get our ELF binary (test).

When we run our program from the terminal (./test), the shell calls execve(). The loader will then call a function named \_start().

## Let's start from the \_start()

If we disassemble our ELF binary with ```objdump -d test```, we see the following block:

```
0000000000000530 <_start>:
 530: 31 ed                	xor    %ebp,%ebp				
 532: 49 89 d1             	mov    %rdx,%r9
 535: 5e                   	pop    %rsi
 536: 48 89 e2             	mov    %rsp,%rdx
 539: 48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
 53d: 50                   	push   %rax
 53e: 54                   	push   %rsp
 53f: 4c 8d 05 8a 01 00 00 	lea    0x18a(%rip),%r8 # 6d0 <__libc_csu_fini>
 546: 48 8d 0d 13 01 00 00 	lea    0x113(%rip),%rcx # 660 <__libc_csu_init>
 54d: 48 8d 3d e6 00 00 00 	lea    0xe6(%rip),%rdi # 63a <main>
 554: ff 15 86 0a 20 00		callq  *0x200a86(%rip) # 200fe0<__libc_start_main@GLIBC_2.2.5>
 55a: f4                   	hlt
 55b: 0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
```

\_start() sets up on the registers the arguments that need to be passed to \_\_libc\_start\_main() such as:


* the constructor of the program (\_\_libc_csu_init() which will get called before calling main()) -- in the rcx register
* the destructor of the program (\_\_libc_csu_fini()) -- in the r8 register
* the main of our program -- in the rdi register.
* and others ...


## Zooming in on \_\_libc\_start\_main()



If we take a look at __libc_start_main() function signature this gets more clear:

```C

int __libc_start_main(int *(main) (int, char * *, char * *),
					  int argc,
					  char * * ubp_av,
					  void (*init) (void),
					  void (*fini) (void),
					  void (*rtld_fini) (void),
					  void (* stack_end));
```

_Note that main seems to take 3 parameters (int, char \*\*, char \*\*). The second array of strings is a pointer to the environmental variables (envp)_

The main operations carried out by __libc_start_main() are:


* Check if the effective uid == real uid (for security reasons)
* Calls the constructors and destructors for our program
* Calls main() with the correct arguments
* Calls exit() with the return value of main.

## Confirming with GDB

By running our program in gdb, we see these functions being called:

```
gdb ./test
```

In gdb's prompt, we run the following commands:

```
set backtrace past-entry
set backtrace past-main
b main
r
```
These allow the backtrace to show callers before main and before the entry point, sets a breakpoint at our main() function in main.c and runs the program. When we hit our breakpoint, if we then run the ```bt``` command (backtrace) we see the following:

```
#0  main (argc=1, argv=0x7fffffffdce8) at main.c:6

#1  0x00007ffff7a05b97 in __libc_start_main (main=0x55555555463a <main>, argc=1,
argv=0x7fffffffdce8, init=<optimized out>, fini=<optimized out>,
rtld_fini=<optimized out>, stack_end=0x7fffffffdcd8) at ../csu/libc-start.c:310

#2  0x000055555555455a in _start ()
```

This shows the stack frames that lead to the execution of main, which are indeed those of \_start() and \_\_libc\_start\_main().


## Conclusion

Even though we have digged deeper, this is still not the whole picture. There is more to how program execution is setup but this is out of the scope of this short post.













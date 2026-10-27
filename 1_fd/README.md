# pwnable.kr: fd

## Analysis

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
        if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
        int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);
        if(!strcmp("LETMEWIN\n", buf)){
                printf("good job :)\n");
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                exit(0);
        }
        printf("learn about Linux file IO\n");
        return 0;

}
```

### Func: main

- Firstly, starts by checking if there's less than two arguments (`./fd argv[1]`):

```c
if(argc<2){
        printf("pass argv[1] a number\n");
        return 0;
}
```

- Secondly, converts `argv [1]` to an `int` with `atoi`. Then subtracts it with `0x1234` (`4660`):
```c
int fd = atoi( argv[1] ) - 0x1234;
```

- Thirdly, it calls the syscall `read`; which writes into `buf` (needing 32 bytes):

```c
len = read(fd, buf, 32);
```

- Finally, gives the flag if `strcmp`'s takes `buf` as `LETMEWIN\n`:

```c
if(!strcmp("LETMEWIN\n", buf)){
        printf("good job :)\n");
        setregid(getegid(), getegid());
        system("/bin/cat flag");
        exit(0);
}
```
#### Understanding

- Needs one argument.
- `fd` needs to equal 0.
- Input the string `LETMEWIN\n` to `buf`.

### Math

```
4660 - 0x1234 (4660) = 0 ⬅︎ standard input value
```

### Steps

Pass the value `4660` into `argv[1]` to give `stdin` to `read`.

Input `LETMEWIN` and press enter, to validate the if statement.

## Final Solution

```bash
./fd 4660
LETMEWIN
good job :)
Mama! Now_I_understand_what_file_descriptors_are!
```

**Flag:** Mama! Now_I_understand_what_file_descriptors_are!
 

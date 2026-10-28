# pwnable.kr: fd

**Source Code**

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

This program takes our `argv[1]`, then translates it from ascii to integer with `atoi`, and subtracts it with `0x1234` (4660). Using that value as the `fd` for `read`.

All you need to do is pass the value `4660` to allow `fd` to be 0 (standard input value).

Then type `LETMEWIN` and press enter.

**Final Solution:**

```bash
./fd 4660
LETMEWIN
good job :)
Mama! Now_I_understand_what_file_descriptors_are!
```

**Flag:** Mama! Now_I_understand_what_file_descriptors_are!
 

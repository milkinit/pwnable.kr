# pwnable.kr: input2

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");
	
	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");	

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	setregid(getegid(), getegid());
	system("/bin/cat flag");	
	return 0;
```

This challenge wants us to complete each stage (1-5) to get the flag. So, we'll walkthrough each one in detail explaining its solution.

**Stage 1:**

This stage wants us to give 100 characters with `argv['A']` (A = 0x41 = 65) to be `0x00` and `argv['B']` (B = 0x42 = 66) to be `0x200a0d`:

```c
// argv
if(argc != 100) return 0;
if(strcmp(argv['A'],"\x00")) return 0;
if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
printf("Stage 1 clear!\n"); 
```

Solution:

```python
port = 12345
args = (
        [b"./input2"] 
        + [b"x"] * 64 
        + [b""] 
        + [b"\x20\x0a\x0d"]
        + [str(port).encode()]
        + [b"x"] * 32
    )
```

**Stage 2:**

This stage goes through `stdio`, it wants us to give the standard input stream `0x000a00ff` and standard error stream `0x000a02ff`:

```c
// stdio
char buf[4];
read(0, buf, 4);
if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
read(2, buf, 4);
if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
printf("Stage 2 clear!\n");
```

Solution:

```python
fd0_input = tempfile.TemporaryFile()
fd0_input.write(b"\x00\x0a\x00\xff")
fd0_input.seek(0)
fd2_input = tempfile.TemporaryFile()
fd2_input.write(b"\x00\x0a\x02\xff")
fd2_input.seek(0)
```

**Stage 3:**

This stage we need to set the environment variable (`0xdeadbeef`) to the value `0xcafebabe`:

```c
// env
if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
printf("Stage 3 clear!\n");
```

Solution:

```python
env = os.environ.copy()
env["\xde\xad\xbe\xef"] = "\xca\xfe\xba\xbe"
```

**Stage 4:**

This stage we need to create the file `0x0a` and write `0x00000000` into it:

```c
// file
FILE* fp = fopen("\x0a", "r");
if(!fp) return 0;
if( fread(buf, 4, 1, fp)!=1 ) return 0;
if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
fclose(fp);
printf("Stage 4 clear!\n"); 
```

Solution:

```python
with open("\x0a", "wb") as f:
    f.write(b"\x00\x00\x00\x00")
```

**Stage 5:**

This stage you need to send the data to the new opened TCP socket thats listening on port `12345`

> NOTE: back in stage 1 there was an added the port `12345` on `argv['C']` (C = 0x43 = 67)

```c
// network
int sd, cd;
struct sockaddr_in saddr, caddr;
sd = socket(AF_INET, SOCK_STREAM, 0);
if(sd == -1){
    printf("socket error, tell admin\n");
    return 0;
}
saddr.sin_family = AF_INET;
saddr.sin_addr.s_addr = INADDR_ANY;
saddr.sin_port = htons( atoi(argv['C']) );
if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
printf("bind error, use another port\n");
    return 1;
}
listen(sd, 1);
int c = sizeof(struct sockaddr_in);
cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
if(cd < 0){
    printf("accept error, tell admin\n");
    return 0;
}
if( recv(cd, buf, 4, 0) != 4 ) return 0;
if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
printf("Stage 5 clear!\n");
```

Solution:

```python
time.sleep(0.1)
conn = remote("127.0.0.1", 12345)
conn.send(b"\xde\xad\xbe\xef")
conn.close()
```

**Running Program:**

```bash
./exploit.py 
[+] Starting local process './input2': pid 717
[+] Opening connection to 127.0.0.1 on port 12345: Done
[*] Closed connection to 127.0.0.1 port 12345
[*] Switching to interactive mode
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
Stage 5 clear!
Mommy_now_I_know_how_to_pa5s_inputs_in_Linux
[*] Process './input2' stopped with exit code 0 (pid 717)
[*] Got EOF while reading in interactive
```

Theres the flag!

**Exploit:**

```python
#!/bin/python3

from pwn import *
import tempfile
import os 
import time

context.log_level = 'debug'

port = 12345

# STAGE 1 --------------------------
args = (
        [b"./input2"] 
        + [b"x"] * 64 
        + [b""] 
        + [b"\x20\x0a\x0d"]
        + [str(port).encode()]
        + [b"x"] * 32
    )
# ----------------------------------

# STAGE 2 --------------------------
fd0_input = tempfile.TemporaryFile()
fd0_input.write(b"\x00\x0a\x00\xff")
fd0_input.seek(0)
fd2_input = tempfile.TemporaryFile()
fd2_input.write(b"\x00\x0a\x02\xff")
fd2_input.seek(0)
# ----------------------------------

# STAGE 3 --------------------------
env = os.environ.copy()
env["\xde\xad\xbe\xef"] = "\xca\xfe\xba\xbe"
# ----------------------------------

# STAGE 4 --------------------------
with open("\x0a", "wb") as f:
    f.write(b"\x00\x00\x00\x00")
# ----------------------------------

p = process(args, stdin=fd0_input, stderr=fd2_input, env=env)

# STAGE 5 --------------------------
time.sleep(0.1)
conn = remote("127.0.0.1", 12345)
conn.send(b"\xde\xad\xbe\xef")
conn.close()
# ----------------------------------

p.interactive()
```

**Flag:** Mommy_now_I_know_how_to_pa5s_inputs_in_Linux

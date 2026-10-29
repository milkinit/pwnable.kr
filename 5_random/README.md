# pwnable.kr: random

**Source Code:**

```c
#include <stdio.h>

int main(){
	unsigned int random;
	random = rand();	// random value!

	unsigned int key=0;
	scanf("%d", &key);

	if( (key ^ random) == 0xcafebabe ){
		printf("Good!\n");
		setregid(getegid(), getegid());
		system("/bin/cat flag");
		return 0;
	}

	printf("Wrong, maybe you should try 2^32 cases.\n");
	return 0;
}
```

From the start we can see that the `random = rand();` has no `srand()` call, so it behaves as though you called `srand(1)`. Producing the same sequence each time the program starts.

In result, making `if( (key ^ random) == 0xcafebabe )` predictable.

Under a debugger, the value of `random` is `0x6b8b4567`:

```bash
0x555555555223 <main+26>    xor    eax, eax                        EAX => 0
0x555555555225 <main+28>    mov    eax, 0                          EAX => 0
0x55555555522a <main+33>    call   rand@plt                    <rand@plt>
0x55555555522f <main+38>    mov    dword ptr [rbp - 0x1c], eax     [0x7fffffffdbb4] <= 0x6b8b4567
```

And to get the value of `key`, we use the formula: `KEY = RANDOM ^ RESULT`:

```
RANDOM = 0x6b8b4567
RESULT = 0xcafebabe

0x6b8b4567 ^ 0xcafebabe = 2708864985

KEY = 2708864985 ( hex value: 0xa175ffd9 )
```

Now that we have `key`'s value, we recreate the `if( (key ^ random) == 0xcafebabe )` condition:

```
0xa175ffd9 ^ 0x6b8b4567 = 0xcafebabe
```

So, when we give `scanf` the value `2708864985`:

```bash
echo '2708864985' | nc pwnable.kr 10005
Good!
m0mmy_I_can_predict_rand0m_v4lue!
```

We get the flag!

**Exploit:**

```python
#!/bin/python3

from pwn import *

HOST = "pwnable.kr"
PORT = 10005

conn = remote(HOST, PORT)

payload = b'2708864985' + b'\n'

conn.send(payload)

conn.interactive()
```

**Flag:** m0mmy_I_can_predict_rand0m_v4lue!

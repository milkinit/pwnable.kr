# pwnable.kr: collision

**Source Code**

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}
```
The main goal for this program is that the returned value of `check_password` needs to equal to `hashcode`.

The function `check_password` type casts `argv[1]` from `char` to `int`:

```c
int* ip = (int*)p;
```

Then 4 bytes from `ip[i]` will be incremented five times each, adding its sum to `res`:

```c
int i;
int res=0;
for(i=0; i<5; i++){
    res += ip[i];
}
return res;
```


Debugging with `argv[1]` being `12345678901234567890`:

**First increment (`ip[1]`):**

1. Stores `argv[1]` into `EAX`.
2. Reads the first 4 bytes (`1234`) from `ip[1]` of array (`12345678901234567890`).
3. Adds value to `res`.
4. Increments `i` from the `for-loop` by 1.

```bash
1) 0x5655621d <check_password+48>    mov    eax, dword ptr [ebp - 0xc]     EAX, [0xffffce4c] => 0xffffd157 ◂— '12345678901234567890'                                        
2) 0x56556222 <check_password+53>    mov    eax, dword ptr [eax]           EAX, [0xffffd157] => 0x34333231 ('1234')                                                         
3) 0x56556224 <check_password+55>    add    dword ptr [ebp - 8], eax       [0xffffce50] <= 0x34333231 (0x0 + 0x34333231)
4) 0x56556227 <check_password+58>    add    dword ptr [ebp - 4], 1         [0xffffce54] <= 1 (0 + 1)
```

**Last increment (`ip[5]`):**

1. The value of `ip[5]` (`7890`).
2. Adds final value to `res`.
3. Last increment.

```bash
1) 0x56556222 <check_password+53>    mov    eax, dword ptr [eax]           EAX, [0xffffd167] => 0x30393837 ('7890')                                                         
2) 0x56556224 <check_password+55>    add    dword ptr [ebp - 8], eax       [0xffffce50] <= 0x50a0509 (0xd4d0ccd2 + 0x30393837)
3) 0x56556227 <check_password+58>    add    dword ptr [ebp - 4], 1         [0xffffce54] <= 5 (4 + 1)
```

**The logic behind the sum of `res`:**

- `hashcode` needs to equal `res`.
- `hashcode` / 5 equals `ip[1]`.

> The 5th value needs to be different due to 0x21DD09EC / 5 = 113626824.8.

**Using python2:**

Getting the 5th value:

```python
>>> int(0x21DD09EC)
568134124

# 5th value
>>> 568134124 - 100000004
468134120
# 1 - 4 value
>>> 468134120 / 4
117033530

# Convert dec back to hex
>>> hex(117033530) 
'0x6f9ca3a'
>>> hex(100000004) 
'0x5f5e104'
```

Putting everything together we have `0x1be728e8` being multiplied by 4 and `0x21dd09ec` being the remainder:

```
0x06f9ca3a * 4 = 0x1be728e8
0x1be728e8 + 0x05f5e104 = 0x21dd09ec
```

**Final Solution:**

```bash
./col $(python2 -c "print '\x3a\xca\xf9\x06' * 4 + '\x04\xe1\xf5\x05'")
Two_hash_collision_Nicely
```

**Flag:** Two_hash_collision_Nicely


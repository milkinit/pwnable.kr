# pwnable.kr: leg

**Source Code:**

```c
#include <stdio.h>
#include <fcntl.h>
int key1(){
        asm("mov r3, pc\n");
}
int key2(){
        asm(
        "push   {r6}\n"
        "add    r6, pc, $1\n"
        "bx     r6\n"
        ".code   16\n"
        "mov    r3, pc\n"
        "add    r3, $0x4\n"
        "push   {r3}\n"
        "pop    {pc}\n"
        ".code  32\n"
        "pop    {r6}\n"
        );
}
int key3(){
        asm("mov r3, lr\n");
}
int main(){
        int key=0;
        printf("Daddy has very strong arm! : ");
        scanf("%d", &key);
        if( (key1()+key2()+key3()) == key ){
                printf("Congratz!\n");
                int fd = open("flag", O_RDONLY);
                char buf[100];
                int r = read(fd, buf, 100);
                write(0, buf, r);
        }
        else{
                printf("I have strong leg :P\n");
        }
        return 0;
}
```

The program takes in input for `int key=0;`, which will be compared to the sum of the three key functions: `key1()`, `key2()`, and `key3()`. 

**Assembler Dump:**

```assembly
Dump of assembler code for function key1:
   0x00008cd4 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
   0x00008cd8 <+4>:     add     r11, sp, #0
   0x00008cdc <+8>:     mov     r3, pc
   0x00008ce0 <+12>:    mov     r0, r3
   0x00008ce4 <+16>:    sub     sp, r11, #0
   0x00008ce8 <+20>:    pop     {r11}           ; (ldr r11, [sp], #4)
   0x00008cec <+24>:    bx      lr
End of assembler dump.

Dump of assembler code for function key2:
   0x00008cf0 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
   0x00008cf4 <+4>:     add     r11, sp, #0
   0x00008cf8 <+8>:     push    {r6}            ; (str r6, [sp, #-4]!)
   0x00008cfc <+12>:    add     r6, pc, #1
   0x00008d00 <+16>:    bx      r6
   0x00008d04 <+20>:    mov     r3, pc
   0x00008d06 <+22>:    adds    r3, #4
   0x00008d08 <+24>:    push    {r3}
   0x00008d0a <+26>:    pop     {pc}
   0x00008d0c <+28>:    pop     {r6}            ; (ldr r6, [sp], #4)
   0x00008d10 <+32>:    mov     r0, r3
   0x00008d14 <+36>:    sub     sp, r11, #0
   0x00008d18 <+40>:    pop     {r11}           ; (ldr r11, [sp], #4)
   0x00008d1c <+44>:    bx      lr
End of assembler dump.

Dump of assembler code for function key3:
   0x00008d20 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
   0x00008d24 <+4>:     add     r11, sp, #0
   0x00008d28 <+8>:     mov     r3, lr
   0x00008d2c <+12>:    mov     r0, r3
   0x00008d30 <+16>:    sub     sp, r11, #0
   0x00008d34 <+20>:    pop     {r11}           ; (ldr r11, [sp], #4)
   0x00008d38 <+24>:    bx      lr
End of assembler dump.
```

**Key1():**

```assembly
; epilogue
0x00008cd4 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
0x00008cd8 <+4>:     add     r11, sp, #0

; code
0x00008cdc <+8>:     mov     r3, pc
0x00008ce0 <+12>:    mov     r0, r3

; prologue
0x00008ce4 <+16>:    sub     sp, r11, #0
0x00008ce8 <+20>:    pop     {r11}           ; (ldr r11, [sp], #4)
0x00008cec <+24>:    bx      lr
```

The code moves `0x00008ce4` (`pc`) into `r3`:

```assembly
0x00008cdc <+8>:     mov     r3, pc
```

Then moves `r3` into `r0`:

```assembly
0x00008ce0 <+12>:    mov     r0, r3
```

**Returned Value:** `0x00008ce4`

**Key2():**


```assembly
; epilogue
0x00008cf0 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
0x00008cf4 <+4>:     add     r11, sp, #0

; code
0x00008cf8 <+8>:     push    {r6}            ; (str r6, [sp, #-4]!)
0x00008cfc <+12>:    add     r6, pc, #1
0x00008d00 <+16>:    bx      r6  
0x00008d04 <+20>:    mov     r3, pc
0x00008d06 <+22>:    adds    r3, #4
0x00008d08 <+24>:    push    {r3}
0x00008d0a <+26>:    pop     {pc}
0x00008d0c <+28>:    pop     {r6}            ; (ldr r6, [sp], #4)
0x00008d10 <+32>:    mov     r0, r3

; prologue
0x00008d14 <+36>:    sub     sp, r11, #0
0x00008d18 <+40>:    pop     {r11}           ; (ldr r11, [sp], #4)
0x00008d1c <+44>:    bx      lr
```

The code pushes `r6` onto the stack, then adds `0x00008d04` (`pc`) and `#1`. Equaling `0x00008d05`, which gets stored into `r6`. 

This activates `thumb state` due to `bx` interpreting the lower bit of `0x00008d05` as `1`.

In result, branching to `0x00008d04` (`0x00008d05-1`):

```assembly
0x00008cf8 <+8>:     push    {r6}            ; (str r6, [sp, #-4]!)
0x00008cfc <+12>:    add     r6, pc, #1
0x00008d00 <+16>:    bx      r6  
```

Here it stores `0x00008d08` (`pc`) into `r3`:

```assembly
0x00008d04 <+20>:    mov     r3, pc
```

Then adds `#4` to `r3`, equaling `0x00008d0c` (`0x00008d08+4`):

```assembly
0x00008d06 <+22>:    adds    r3, #4
```

Finally moving `r3` into `r0`:

```assembly
0x00008d10 <+32>:    mov     r0, r3
```

**Returned Value:** `0x00008d0c`

**Key3():**

```assembly
; epilogue
0x00008d20 <+0>:     push    {r11}           ; (str r11, [sp, #-4]!)
0x00008d24 <+4>:     add     r11, sp, #0

; code
0x00008d28 <+8>:     mov     r3, lr
0x00008d2c <+12>:    mov     r0, r3

; prologue
0x00008d30 <+16>:    sub     sp, r11, #0
0x00008d34 <+20>:    pop     {r11}           ; (ldr r11, [sp], #4)
0x00008d38 <+24>:    bx      lr
```

The code moves `lr` (`0x00008d80`) into `r3`:

```assembly
0x00008d28 <+8>:     mov     r3, lr
```

`0x00008d80`:

```assembly
0x00008d7c <+64>:    bl      0x8d20 <key3>
0x00008d80 <+68>:    mov     r3, r0
```

Then stores `r3` into `r0`:

```assembly
0x00008d2c <+12>:    mov     r0, r3
```

**Returned Value:** `0x00008d80`

**Adding Values:**

```
key1         key2         key3         solution
0x00008ce4 + 0x00008d0c + 0x00008d80 = 0x00001a770 (108400)
```

**Final Solution:**

```
/ $ ./leg
Daddy has very strong arm! : 108400
Congratz!
daddy_has_lot_of_ARM_muscl3
```

Flag!!

**Flag:** daddy_has_lot_of_ARM_muscl3

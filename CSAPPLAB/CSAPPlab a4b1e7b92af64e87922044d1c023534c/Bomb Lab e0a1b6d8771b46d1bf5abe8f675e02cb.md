# Bomb Lab

前置工作

把bomb机器码反汇编到obj.txt

```bash
objdump -d bomb > obj.txt
```

反汇编函数phase_1

```bash
disas/disassemble phase_1
```

下好断点后，带参数ans.txt运行

```bash
r ans.txt
```

显示addr处的两个字符串

```bash
x/2s addr
p (char*) addr
```

以十六进制显示addr处的20个字节的内存

```bash
x/20x addr
```

显示寄存器内容

```bash
info reg
```

# phase_1

先在obj.txt中静态分析

![Snipaste_2021-12-07_10-24-04.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-24-04.png)

可以发现会通过一个比较函数比较地址0x402400处的字符串和输入字符串，不相同就会bomb，相同则函数返回，所以可以动态调试，看一下0x402400处的字符串

对函数phase_1下好断点，带任意一行参数ans.txt运行，然后用gdb的指令查看就行

![Snipaste_2021-12-07_10-27-57.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-27-57.png)

会发现一串字符串，我们输入试试看

![Snipaste_2021-12-07_10-29-48.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-29-48.png)

第一个炸弹成功拆除

# phase_2

先静态分析，发现需要读入六个数字，如果进到**read_six_numbers**函数中，会发现读入数字小于6会bomb

> 可以根据call、jump指令把基本块分出来，看下call指令之前的mov，参数放哪个寄存器，出来再看看参数变成啥样，继续trace一下
> 

![Snipaste_2021-12-07_10-35-47.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-35-47.png)

> 注意x86汇编 mov→ , lea→//?
> 

函数读入数字后会先比较第一个数字是不是1，不是的话就会bomb，所以可以知道第一个数字是1

第一个数字正确之后会跳转到**400f30**处，使rbx指向第二个数字，rbp指向返回地址(rsp+18)，也就是最后一个数字上面的地方，然后jmp到**400f17**处

**400f17**处会把rbx指向的参数的上一个参数的值给到eax，然后把eax翻倍，跟rbx指向的参数进行比较，相当于循环，相等则继续跳转，直到比较完最后一个参数和倒数第二个参数，不相等则bomb

所以从第二个参数开始，每一个参数都应该是前一个参数的两倍

推测答案是**1 2 4 8 16 32**，输入进去试试看，答案正确，炸弹拆除

![Snipaste_2021-12-07_10-44-06.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-44-06.png)

# phase_3

先静态看一下，发现有一串字符串跟scanf有关，gdb看一下

![Snipaste_2021-12-07_10-53-05.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-53-05.png)

![Snipaste_2021-12-07_10-50-38.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-50-38.png)

猜测是输入两个整数

然后第一个整数的值应该小于等于7，不然会bomb，所以第一个整数可以是0 1 2 3 4 5 6 7

像是swith的结构

然后会根据第一个整数的值进行跳转，把跳转的表在gdb里面看一下

![Snipaste_2021-12-07_10-56-39.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_10-56-39.png)

跟汇编代码是对应的

第一个整数是0，跳转到400f7c，eax为207

第一个整数是1，跳转到400fb9，eax为311

第一个整数是2，跳转到400f83，eax为707

第一个整数是3，跳转到400f8a，eax为256

第一个整数是4，跳转到400f91，eax为389

第一个整数是5，跳转到400f98，eax为206

第一个整数是6，跳转到400f9f，eax为682

第一个整数是7，跳转到400fa6，eax为327

因为最后要拿第二个整数跟eax比较，所以可以得出八组答案

```bash
ans:
0 207
1 311
2 707
3 256
4 389
5 206
6 682
7 327
```

随便拿几组试试看，炸弹就拆除了

![Snipaste_2021-12-07_11-03-18.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_11-03-18.png)

# phase_4

```python
	
0x000000000040102e <+34>:    cmp    DWORD PTR [rsp+0x8],0xe		  第一个整数大于0xe就寄
   0x0000000000401033 <+39>:    jbe    0x40103a <phase_4+46>
   0x0000000000401035 <+41>:    call   0x40143a <explode_bomb>
   0x000000000040103a <+46>:    mov    edx,0xe
   0x000000000040103f <+51>:    mov    esi,0x0
   0x0000000000401044 <+56>:    mov    edi,DWORD PTR [rsp+0x8]
   0x0000000000401048 <+60>:    call   0x400fce <func4>				 func4(第一个整数, 0, 0xe);	
   0x000000000040104d <+65>:    test   eax,eax						返回值不为0就寄
   0x000000000040104f <+67>:    jne    0x401058 <phase_4+76>		
   0x0000000000401051 <+69>:    cmp    DWORD PTR [rsp+0xc],0x0		  第二个整数不为0就寄
   0x0000000000401056 <+74>:    je     0x40105d <phase_4+81>
   0x0000000000401058 <+76>:    call   0x40143a <explode_bomb>
```

先静态看一下，发现输入的第一个数应该小于等于0xe，第二个数字应该等于0

![Snipaste_2021-12-07_15-43-56.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_15-43-56.png)

第一个数小于等于0xe后会进入一个func4函数，返回值eax应该等于0，才能不bomb

进func4看一下

![Snipaste_2021-12-07_15-48-12.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_15-48-12.png)

可以看出第一次ecx值为7，然后edi是输入的第一个数字，edi小于等于7而且大于等于7时可以直接跳过递归，返回eax为0，拆除炸弹

所以第一个数字是7第二个数字是0可以拆除炸弹，输入试试看，发现炸弹拆除了

![Snipaste_2021-12-07_15-53-28.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_15-53-28.png)

也可以不跳过递归，进入递归，这样的话就要好好分析一下func4了，可以直接把这个递归函数逆向成伪代码

```c
int func4(int a1, int a2, int x)//a1 = 0xe a2 = 0 x = 第一个数字
{
	int b = (a1 - a2) >> 31;
	int result = ((a1 - a2) + b) >> 1;
	b = result + a2;
	if (b == x)
		return 0;
	else if (b > x)
	{
		result = func4(b - 1, a2, x);
		return result * 2;
	}
	else
	{
		result = func4(a1, b+1, x);
		return result * 2 + 1;
	}
}
```

因为返回值eax要为0，所以可以爆破一下

```c
int main()
{
	int i;
	int a1 = 0xe;
	int a2 = 0;
	for ( i = 0; i <= 0xe; i++)
	{
		if (!func4(a1, a2, i))
		{
			printf("%d\n", i);

		}
	}

	return 0;
}
//0 1 3 7
```

所以答案可以是

```c
0 0
1 0
3 0
7 0
```

# phase_5

先进行静态分析，分析这一部分可以知道程序把输入字符串的地址rdi给了rbx，然后对字符串长度进行判断，不等于6就会bomb

![Snipaste_2021-12-07_18-33-55.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_18-33-55.png)

再来看这一部分，把字符串的第一个字符取出来给ecx，然后让rsp指向它，再把这个字符给rdx，然后异或0xf取低四位，然后再作为0x4024b0地址处字符串的下标，取出值给edx，再让(rsp+10)指向这个取出来的字符，然后循环6次

![Snipaste_2021-12-07_18-37-16.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_18-37-16.png)

先把0x4024b0处地址的字符串打印一下

```bash
pwndbg> x/1s 0x4024b0
0x4024b0 <array.3449>:	"maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
```

因为下标是取的低四位，所以范围是0-0xf，所以能取出来的值在**maduiersnfotvby**之中，后面的字符可以不用管

再看这一部分，给(rsp+16)指向的地方放一个0，作为字符串的终止位置，然后把地址0x40245e中的字符串放入esi，然后调用函数进行比较，比较成功就结束，失败就bomb了

![Snipaste_2021-12-07_18-50-15.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_18-50-15.png)

把0x40245e中的字符串拿出来看一下

```bash
pwndbg> x/1s 0x40245e
0x40245e:	"flyers"
```

也就是说我们输入的字符串经过置换，应该等于**"flyers"**

我们把**"flyers"**在数组中的索引都标出来

```bash
f	l	y	e	r	s
9	15	14	5	6	7
```

也就是说输入的字符串取低八位后分别是**9	15	14	5	6	7**

随便找一组答案输入即可，我输入的是**ionefg**

![Snipaste_2021-12-07_18-58-02.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-07_18-58-02.png)

# phase6

汇编代码太多太乱，分块慢慢看，首先往里读  入六个数字

![Snipaste_2021-12-08_17-01-09.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-08_17-01-09.png)

前面把rsp给了r13，那么地址401117处相当于把rsp中的四位给到eax，也就是输入的第一个数字，然后再把第一个数字减1，跟5比较，小于等于就会跳过炸弹，也就是说第一个数字必须小于等于6

![Snipaste_2021-12-08_17-03-52.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-08_17-03-52.png)

然后跳转到地址401128，r12加1，r12初始值是0，然后跟6比较，等于就跳出，所以这一层大循环的意思应该是验证每一个数字都应该小于等于6

再往下看，地址401132把r12的值给ebx，ebx再给到rax，再把(rsp+rax*4)的值给到eax，相当于rax是rsp的下标，每次取出四个字节，也就是一个我们输入的数字，r12等于1，所以rax等于1，所以相当于取出下一个数字，然后有个 ，把当前数字和下一个数字比较，不相等就跳过炸弹，跳到401145，然后进行下一轮

401145给ebx+1，然后跟0x5进行比较，小于等于就跳转到401135，把ebx给到rax作为下标，跟上面一样，相当于再往后取一个数字，以此类推，这个小循环相当于拿当前数字跟后面的数字作比较，不相等才能跳过炸弹

小循环结束后到40114d这里，这里给r13加0x4，然后进行大循环，大循环就是判断当前数字是否小于等于6，然后进入小循环，判断当前数字跟后面的数字是否相等，以此类推

可以知道这一段代码的作用就是判断每个数字小于等于6且不跟后面的数字相等

结束之后会跳到地址401153，再来看下面这一块汇编

![Snipaste_2021-12-08_17-38-41.png](Bomb%20Lab%20e0a1b6d8771b46d1bf5abe8f675e02cb/Snipaste_2021-12-08_17-38-41.png)

把rsp+0x18给到rsi，rsi实际上指向的是6个数字之后的位置，作为一个标记使用

然后把r14给到rax，r14存的是rsp的位置，相当于把r14给到rax

然后把0x7给到ecx，又把ecx给到edx，然后用edx-0x7存进(rax)中，然后rax+0x4，指向下一个数字

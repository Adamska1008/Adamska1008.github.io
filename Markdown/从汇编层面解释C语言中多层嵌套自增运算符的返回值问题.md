### 问题如下
源代码：
``` c
int x = 1;
int y = ++x + x++;
printf("%d", y);
```
输出：
```
5
```
按照C语言初学者的朴素想法，在第二行中，首先执行++x,返回值为2，此时x值为2，再执行x++，此时x值为3，返回值为2，最后将两个返回值相加得到4，所以运行完毕后x值为3，输出4。但实际输出5，不符合预期。  
如何解释这一现象呢？笔者原本尝试利用C语言本身的性质来解释，但无果。考虑到C语言与汇编代码之间有着较为明确与清晰的关联，如果直接分析程序的汇编代码，想必能得出较好的结果。
### 提取汇编代码
首先，给出一个完整的，可通过编译的代码文件code.c: 
``` c
void test(int x, int y)
{
    x = 1;
    y = ++x + x++;
}
```
执行gcc指令
```
gcc -S code.c
```
得到 code.s
``` x86asm
	.file	"code.c"
	.text
	.globl	test
	.def	test;	.scl	2;	.type	32;	.endef
	.seh_proc	test
test:
	pushq	%rbp
	.seh_pushreg	%rbp
	movq	%rsp, %rbp
	.seh_setframe	%rbp, 0
	.seh_endprologue
	movl	%ecx, 16(%rbp)
	movl	%edx, 24(%rbp)
	movl	$1, 16(%rbp)
	movl	16(%rbp), %eax
	leal	1(%rax), %edx
	movl	%edx, 16(%rbp)
	movl	%eax, 24(%rbp)
	nop
	popq	%rbp
	ret
	.seh_endproc
	.ident	"GCC: (x86_64-win32-seh-rev0, Built by MinGW-W64 project) 8.1.0"
```
提取关键代码：
``` 	x86asm
pushq	%rbp
movq	%rsp, %rbp
movl	%ecx, 16(%rbp)
movl	%edx, 24(%rbp)
movl	$1, 16(%rbp)
movl	16(%rbp), %eax
leal	1(%rax), %edx
movl	%edx, 16(%rbp)
movl	%eax, 24(%rbp)
nop
popq	%rbp
ret
```
对于熟悉汇编的同学来说，得到汇编代码后，这个问题的答案就已经非常显然了。但是考虑到这一块知识点对于大部分同学来说都超纲了，故在下文中进行一步一步的详细解释。
### C语言中=运算符的汇编代码实现
首先考虑如下代码
``` c
void test(int x)
{
    x = 1;
}
```
其汇编实现为
``` x86asm
pushq	%rbp
movq	%rsp, %rbp
movl	%ecx, 16(%rbp)
movl	$1, 16(%rbp)
nop
popq	%rbp
ret
```
分析后不难得出，ecx中起初存放的就是传入的x值，首先将该值传送给地址16(%rbp)，这也就是变量x的临时地址。  
接下来将临时值$1传送给16(%rbp)，就完成了对x的赋值。
### C语言中+运算符的汇编代码实现
考虑如下代码
```c
void test(int x, int y1, int y2, int y3)
{
    x = 1;
    y1 = x + 1;
    y2 = 1 + x;
    y3 = x + x;
}
```
其汇编实现为
``` x86asm
pushq	%rbp
movq	%rsp, %rbp
movl	%ecx, 16(%rbp)
movl	%edx, 24(%rbp)
movl	%r8d, 32(%rbp)
movl	%r9d, 40(%rbp)
movl	$1, 16(%rbp)
movl	16(%rbp), %eax
addl	$1, %eax
movl	%eax, 24(%rbp)
movl	16(%rbp), %eax
addl	$1, %eax
movl	%eax, 32(%rbp)
movl	16(%rbp), %eax
addl	%eax, %eax
movl	%eax, 40(%rbp)
nop
popq	%rbp
ret
```
首先将x和y的临时值分别存放在16(%rbp)和24(%rbp)上，然后对x赋值1，以上内容我们已经了解了。  
然后，汇编代码是如何处理三个加法语句的呢？相关代码如下：
``` x86asm
;y=x+1
movl	16(%rbp), %eax
addl	$1, %eax
movl	%eax, 24(%rbp)
;y=1+x
movl	16(%rbp), %eax
addl	$1, %eax
movl	%eax, 32(%rbp)
;y=x+x
movl	16(%rbp), %eax
addl	%eax, %eax
movl	%eax, 40(%rbp)
```
非常简单，将16(%rbp)，的值传入eax，然后执行addl指令，最后再将eax中的值移回16(%rbp)。  
同时我们发现，y=x+1与y=1+x的语句是相同的，可见在C语言中，+运算符的左右参数顺序并不会影响结果。同时，对于y=x+x,会直接执行addl %eax,%eax指令，而不会使用两个寄存器来分别读取进行加法。
### C语言中++运算符的汇编代码实现
此为**重点**，先放后置自增运算符的代码：
``` c
void test(int x, int y)
{
    x = 1;
    y = x++;
}
```
``` x86asm
pushq	%rbp
movq	%rsp, %rbp
movl	%ecx, 16(%rbp)
movl	%edx, 24(%rbp)
movl	$1, 16(%rbp)
movl	16(%rbp), %eax
leal	1(%rax), %edx
movl	%edx, 16(%rbp)
movl	%eax, 24(%rbp)
nop
popq	%rbp
ret
```
凭借我们的C语言知识可以知道，首先会对y赋值x，等语句执行完之后再使x自增，那么，如何从汇编层面解释呢？关键代码如下
```x86asm
;y=x++;
movl	16(%rbp), %eax
leal	1(%rax), %edx
movl	%edx, 16(%rbp)
movl	%eax, 24(%rbp)
```
首先将16(%rbp)的值传入eax，然后利用了一个leal指令
``` x86asm
leal	1(%rax), %edx
```
这个指令的效果是对edx传值，传的值是eax存储的值加上1，注意，此时eax的值并没有改变。所以执行完这一指令后，eax存储的值依旧为1，而edx存储的值为2。最后从edx传值给16(%rbp)，从eax传值给24(%rbp)，这就完成了对x赋值2，对y赋值1。  
分析这段代码，我们可以得到结论，对需要赋给y的值，始终都存在eax中，而对于x自增后返回值，则运算后用edx存储，由于是利用两个存储器分别传值，所以对x自增并没有影响到对y赋值。    
接下来放出前置自增运算符代码，比较简单：
``` c
void test(int x, int y)
{
    x = 1;
    y = ++x;
}
```
``` x86asm
pushq	%rbp
movq	%rsp, %rbp
movl	%ecx, 16(%rbp)
movl	%edx, 24(%rbp)
movl	$1, 16(%rbp)
addl	$1, 16(%rbp)
movl	16(%rbp), %eax
movl	%eax, 24(%rbp)
nop
popq	%rbp
ret
```
核心代码：
``` x86asm
addl	$1, 16(%rbp)
movl	16(%rbp), %eax
movl	%eax, 24(%rbp)
```
可以看到，首先对16(%rbp)直接adll $1，这一步甚至没有调用寄存器。这样就实现了先自增再赋值。
### 回到开头
还记得这段代码吗？
``` c
void test(int x, int y)
{
    x = 1;
    y = ++x + x++;
}
```
以及对应的汇编代码:
``` x86asm
pushq	%rbp
movq	%rsp, %rbp
movl	%ecx, 16(%rbp)
movl	%edx, 24(%rbp)
movl	$1, 16(%rbp)
addl	$1, 16(%rbp)
movl	16(%rbp), %eax
leal	1(%rax), %edx
movl	%edx, 16(%rbp)
movl	16(%rbp), %edx
addl	%edx, %eax
movl	%eax, 24(%rbp)
nop
popq	%rbp
ret
```
我们看核心的y=++x+x++部分
``` x86asm
;y=++x+x++
addl	$1, 16(%rbp)
movl	16(%rbp), %eax
leal	1(%rax), %edx
movl	%edx, 16(%rbp)
movl	16(%rbp), %edx
addl	%edx, %eax
movl	%eax, 24(%rbp)
```
逐步分析：  
1. 执行++x，将x自增，此时x值为2
2. 然后，将16(%rbp)的值传给eax,此时eax中的值为2
3. 进入x++部分，首先给edx传值1(%eax)，也就是3
4. 然后将edx中的值传回16(%rbp)，此时x值为3
5. 然后又将16(%rbp)存储的值传回edx，edx中的值仍为3
6. 最后执行+，将eax和edx中的值相加，并存储在eax中，此时eax中的值为5
7. 最后从eax传值给24(%rbp)，此时y为5
      
看起来非常清晰啊，y的值就应该是5。  
一开始，C语言初学者理解的y=++x+x++是这样的：
```c
x += 1;
y = x + x;
x += 1;
```
实际上，y=++x+x++相当于这样：
``` c
x += 1;
int a = x;
int b = a + 1;
x = b;
y = a + b; 
```
这说明，自增运算符的运算顺序实际上并不像我们平时学的那样简单。实际编程中，若遇到多层嵌套自增运算符的问题，除了从汇编代码理解外别无选择。所以我们要尽量少使用这一写法。  
当然，从学习角度来说，这可以看作汇编语言的练习，对理解汇编语言有好处。
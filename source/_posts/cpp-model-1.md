title: 函数调用过程: 结构体变量作为函数参数和返回值
category: Asm/Cpp
date: 2016-4-6 
tags: [Asm,C,Cpp]
toc: false
---

##### 开发环境     
- Ubuntu 14.04(32bits)
- GCC      
- 编辑器 Cmd Markdown
- 画图工具 Processon    

##### 1,结构体类型作为函数参数和返回值
[上一节](http://chinchao.xyz/2016/04/05/cpp-model-0/) 简单介绍了基本的函数调用过程，即栈帧。本节介绍结构体类型作为函数参数和返回值的情况。
###### 1.1结构体变量的内存布局
结构体变量的数据成员之间的内存空间是连续的，结构体中的每个成员都是通过相对于结构体变量首地址的偏移量来访问。假设有如下结构体：
```bash
struct test{
	char k;
	int i;
	short j;
};
如果想获得k变量的偏移量可以通过如下方式：
struct test* p=NULL;
(unsigned int)&(p->k);
等价于： (unsigned)(&(((struct test*)0)->k));
或者等价于:
struct test x;
(unsigned int)((unsigned char)&(x.k)-(unsigned char)&(x));
```
struct test x;其中x结构体变量的内存布局为：
其中空白表示该字节为对齐字节，即没有用到；i0表示i变量的第一个字节，以此类推。
![struct](/img/c1-0.png)
<!--more--> 
##### 1.2结构体作为函数参数和返回值
简单的说，当结构体变量作为函数参数时，将实参t的内存复制“一份”，即x，然后压栈。当结构体作为函数返回值时，是将函数返回值赋给的结构体变量tx的首地址压栈,然后执行fun函数时，对tx进行赋值。
C语言代码如下：
```bash
#include<stdio.h>
typedef struct test{
	char k;
	int i;
	short j;
}test;

test fun(test x)
{
	x.k=1;
	x.i=2;
	x.j=3;
	return x;
}

int main()
{
	test t;
	t.k=3;
	t.i=2;

	t.j=1;
	test tx=fun(t);
	printf("%d,%d,%d;%d,%d,%d\n",t.k,t.i,t.j,tx.k,tx.i,tx.j);
}

```
对应的汇编代码如下：
```bash
	.file	"test.c"
	.text
	.globl	fun
	.type	fun, @function
fun:
.LFB0:
	.cfi_startproc
	pushl	%ebp //ebp压栈，即old ebp
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp  //ebp=esp
	.cfi_def_cfa_register 5
	movb	$1, 12(%ebp)  //ebp+12即为形参x的首地址,然后对x.k=1
	movl	$2, 16(%ebp) //ebp+16即为x。i的首地址，即为x.i=2
	movw	$3, 20(%ebp) //ebp+20即为x.j的首地址，即为x.j=3
	movl	8(%ebp), %eax //tx变量的首地址赋给eax
	movl	12(%ebp), %edx //ebp+12为形参x的首地址
	movl	%edx, (%eax) //将x[0,3]的四个字节赋给tx变量的tx[0,3]
	movl	16(%ebp), %edx 
	movl	%edx, 4(%eax) //将x[4,7]的四个字节赋给tx变量的tx[4,7]
	movl	20(%ebp), %edx
	movl	%edx, 8(%eax) //将x[8,11]的四个字节赋给tx变量的tx[8,11]，\
	至此，已经将x的12个bytes全部赋值给tx
	movl	8(%ebp), %eax //fun函数的返回值，即为tx变量的首地址
	popl	%ebp //ebp=old ebp;esp-=4
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret	$4 //pop eip; esp+=4
	.cfi_endproc
.LFE0:
	.size	fun, .-fun
	.section	.rodata
.LC0:
	.string	"%d,%d,%d;%d,%d,%d\n"
	.text
	.globl	main
	.type	main, @function
main:
.LFB1:
	.cfi_startproc
	leal	4(%esp), %ecx
	.cfi_def_cfa 1, 0
	andl	$-16, %esp
	pushl	-4(%ecx)
	pushl	%ebp
	.cfi_escape 0x10,0x5,0x2,0x75,0
	movl	%esp, %ebp
	pushl	%edi
	pushl	%esi
	pushl	%ebx
	pushl	%ecx
	.cfi_escape 0xf,0x3,0x75,0x70,0x6
	.cfi_escape 0x10,0x7,0x2,0x75,0x7c
	.cfi_escape 0x10,0x6,0x2,0x75,0x78
	.cfi_escape 0x10,0x3,0x2,0x75,0x74
	subl	$72, %esp
	movb	$3, -48(%ebp)  // ebp-48为t.k变量的首地址
	movl	$2, -44(%ebp)  //ebp-44为t.i变量的首地址
	movw	$1, -40(%ebp) //ebp-40为t.j变量的首地址
	leal	-36(%ebp), %eax  //eax=ebp-36  为tx变量的首地址
	movl	-48(%ebp), %edx  //edx=[ebp-48] 即取ebp-48为首地址的4个bytes
	movl	%edx, 4(%esp)  //[esp+4]=edx
	movl	-44(%ebp), %edx //edx=[ebp-44] 即取ebp-44为首地址的4个bytes
	movl	%edx, 8(%esp)  //[esp+8]=edx
	movl	-40(%ebp), %edx //edx=[ebp-40] 即取ebp-40为首地址的4个bytes
	movl	%edx, 12(%esp) //[esp+12]=edx;至此，t变量的12个bytes作为函数实参均已压栈
	movl	%eax, (%esp) //将tx变量的首地址压栈，然后直接将返回值存入该地址
	call	fun
	subl	$4, %esp //esp-=4
	movzwl	-28(%ebp), %eax //0扩展的字(即t.j)mov到双字，即4bytes,
	movswl	%ax, %edi //低地址的两个bytes赋值给edi,做有符号扩展
	movl	-32(%ebp), %esi //esi=t.i
	movzbl	-36(%ebp), %eax //t.k 从1个字节0扩展至双字，即四个字节
	movsbl	%al, %ebx //eax寄存器的第一个字节有符号扩展至ebx四个字节，后边tx的类似，略
	movzwl	-40(%ebp), %eax 
	movswl	%ax, %ecx
	movl	-44(%ebp), %edx
	movzbl	-48(%ebp), %eax
	movsbl	%al, %eax
	movl	%edi, 24(%esp)
	movl	%esi, 20(%esp)
	movl	%ebx, 16(%esp)
	movl	%ecx, 12(%esp)
	movl	%edx, 8(%esp)
	movl	%eax, 4(%esp)
	movl	$.LC0, (%esp) // printf中类似于"%d%d"的字符串的首地址压栈
	call	printf //调用printf函数
	leal	-16(%ebp), %esp
	popl	%ecx
	.cfi_restore 1
	.cfi_def_cfa 1, 0
	popl	%ebx
	.cfi_restore 3
	popl	%esi
	.cfi_restore 6
	popl	%edi
	.cfi_restore 7
	popl	%ebp
	.cfi_restore 5
	leal	-4(%ecx), %esp
	.cfi_def_cfa 4, 4
	ret
	.cfi_endproc
.LFE1:
	.size	main, .-main
	.ident	"GCC: (Ubuntu 4.8.2-19ubuntu1) 4.8.2"
	.section	.note.GNU-stack,"",@progbits

```
通过阅读以上汇编代码，我们知道，当结构体作为函数返回值时，其等价形式为(即将返回值赋给的变量的首地址作为函数参数传递给函数):

```bash
#include<stdio.h>
typedef struct test{
	char k;
	int i;
	short j;
}test;

void fun(test x,test* p)
{
	x.k=1;
	x.i=2;
	x.j=3;
	
	p->k=x.k;
	p->i=x.i;
	p->j=x.j;
}

int main()
{
	test t;
	t.k=3;
	t.i=2;

	t.j=1;
	test tx;
	fun(t,&tx);
	printf("%d,%d,%d;%d,%d,%d\n",t.k,t.i,t.j,tx.k,tx.i,tx.j);
}
```
当执行fun函数时，其栈帧结构如下：
![stack](/img/c1-1.png)

title: 函数调用过程(栈帧)
category: Asm/Cpp
date: 2016-4-5 
tags: [Asm,C,Cpp]
toc: false
---

##### 开发环境     
- Ubuntu 14.04(32bits)
- GCC      
- 编辑器 Cmd Markdown
- 画图工具 Processon    

##### 1,函数调用过程
今天先介绍下基本的函数调用过程，即栈帧。
###### 1.1栈帧
每个函数调用都对应一个栈帧。每个栈帧由ESP和EBP寄存器来确定。每个函数执行时，其局部变量都是在自己对应的栈帧内分配内存。假设A函数调用B函数，此时正在执行B函数，需要指出的是，当执行完当前函数B后，返回调用函数A，此时执行函数B时，为B函数的局部变量分配的的内存空间也就不存在了。也就是说，函数返回值不能是函数体内局部变量的地址，也不能是局部变量的引用。即如不能出现如下两种形式之一：
<!--more--> 
```bash
int *test()
{
　　int i=123;
　　return &i;
}
或者
int &test()
{
　　int i=123;
　　return i;
}
```
1.2函数调用过程对应的汇编代码
``` bash
#include<stdio.h>

int main()
{
	int i=10;
	int j=11;
	int k=test(i,j);
	k-=1;
	return 0;
}

int test(int a,int b)
{
	int sum=a+b;
	return sum;
}
```
以上C程序对应的汇编代码如下：
```bash
	.file	"c0.c"
	.text
	.globl	main
	.type	main, @function
main:
.LFB0:
	.cfi_startproc
	pushl	%ebp //将ebp压栈，即old ebp
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp   //ebp=esp
	.cfi_def_cfa_register 5
	andl	$-16, %esp  //esp=esp&0xFFFFFFF0,即保证esp的最低4位为0
	subl	$32, %esp   //栈向低地址生长，esp-=32
	movl	$10, 20(%esp) //esp+20即局部变量i的地址
	movl	$11, 24(%esp)//esp+24即局部变量j的地址
	movl	24(%esp), %eax //将j变量的值赋给eax寄存器
	movl	%eax, 4(%esp) //j为test函数的实参，此处将j的值压栈
	movl	20(%esp), %eax //将变量i的值赋给eax寄存器
	movl	%eax, (%esp) //将变量i的值压栈
	call	test  //调用test函数,其中将下条指令(即movl %eax, 28(%esp))\
	的地址压栈,即ret addr,然后转到test函数执行,即eip=test
	movl	%eax, 28(%esp) //eax即为test函数的返回值，\
	将它赋给k，esp+28级为变量k的地址
	subl	$1, 28(%esp) //k=k-1
	movl	$0, %eax //eax寄存器保存函数的返回值
	leave
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret
	.cfi_endproc
.LFE0:
	.size	main, .-main
	.globl	test
	.type	test, @function
test:
.LFB1:
	.cfi_startproc
	pushl	%ebp //ebp压栈，即old ebp
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp //ebp=esp
	.cfi_def_cfa_register 5
	subl	$16, %esp //esp-=16
	movl	12(%ebp), %eax  //形参b的值赋给eax寄存器，即eax=11
	movl	8(%ebp), %edx //形参a的值赋给edx寄存器，edx=10
	addl	%edx, %eax   //将a和b相加,结果赋给eax寄存器
	movl	%eax, -4(%ebp) //ebp-4为局部变量sum的地址，sum=eax
	movl	-4(%ebp), %eax //eax来保存函数的返回值
	leave //等价于 esp=ebp,pop ebp.其中pop ebp即ebp=old ebp
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret //等价于pop eip，即eip=ret addr,esp-=4
	.cfi_endproc
.LFE1:
	.size	test, .-test
	.ident	"GCC: (Ubuntu 4.8.2-19ubuntu1) 4.8.2"
	.section	.note.GNU-stack,"",@progbits
```
当main函数调用test函数时，对应的栈帧见下图
![stack1](/img/c0-0.png)    
当函数test返回后，main函数的栈帧如下图
![stack2](/img/c0-1.png)




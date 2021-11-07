title: 动态/静态数组内存布局
category: Asm/Cpp
date: 2016-4-7 
tags: [Asm,C,Cpp]
toc: false
---

##### 开发环境     
- Ubuntu 14.04(32bits)
- GCC      
- 编辑器 Cmd Markdown
- 画图工具 Processon    

##### 1,数组内存布局
[上一节](http://chinchao.xyz/2016/04/06/cpp-model-1/) 简单介绍了结构体作为函数参数和返回值的情况。本文准备介绍一下数组的内存布局，即静态数组/动态数组和一维数组/二维数组，顺便介绍一下0长度数组的妙用。
###### 1.1静态一维数组和动态二维数组
静态一维数组，即类似于int a[10];动态数据，即类似于int *p=(int*)malloc(10*sizeof(int));（或者int *p=new int[10]）;
###### 1.1.1静态一维数组
```bash
#include<stdio.h>

int main()
{
        int a[3];
        a[0]=1;
        a[1]=2;
        a[2]=3;
        printf("%d %d %d\n",a[0],a[1],a[2]);
}
```
对应的汇编代码为
```bash
	.file	"c2-0.c"
	.section	.rodata
.LC0:
	.string	"%d %d %d\n"
	.text
	.globl	main
	.type	main, @function
main:
.LFB0:
	.cfi_startproc
	pushl	%ebp
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp
	.cfi_def_cfa_register 5
	andl	$-16, %esp
	subl	$32, %esp
	movl	$1, 20(%esp) //esp+20即为p[0]的地址
	movl	$2, 24(%esp) //esp+24即为p[1]的地址
	movl	$3, 28(%esp) //esp+28即为p[2]的地址
	movl	28(%esp), %ecx
	movl	24(%esp), %edx
	movl	20(%esp), %eax
	movl	%ecx, 12(%esp) //将ecx压栈，即[esp+12]=ecx
	movl	%edx, 8(%esp) //将edx压栈，即[esp+8]=edx
	movl	%eax, 4(%esp) //将eax压栈，即[esp+4]=eax
	movl	$.LC0, (%esp) //类似于“%d %d”字符串的地址
	call	printf
	leave
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret
	.cfi_endproc
.LFE0:
	.size	main, .-main
	.ident	"GCC: (Ubuntu 4.8.4-2ubuntu1~14.04.1) 4.8.4"
	.section	.note.GNU-stack,"",@progbits
```
main函数执行时，栈帧为：
![img0](/img/c2-0.png)
<!--more-->
###### 1.2  动态一维数组
```bash
#include<stdio.h>
#include<stdlib.h>
int main()
{
    int *p=(int*)malloc(sizeof(3*sizeof(int)));
    p[0]=1;
    p[1]=2;
    p[2]=3;
    free(p);
}
```
对应的汇编代码为
```bash
	.file	"c2-1.c"
	.text
	.globl	main
	.type	main, @function
main:
.LFB2:
	.cfi_startproc
	pushl	%ebp
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp
	.cfi_def_cfa_register 5
	andl	$-16, %esp
	subl	$32, %esp
	movl	$12, (%esp) //将12压栈
	call	malloc //调用malloc函数
	movl	%eax, 28(%esp) //eax为malloc在堆上分配空间的首地址\
	，esp+28即为p的地址
	movl	28(%esp), %eax //获取p,即p指向的内存的首地址 
	movl	$1, (%eax)  //即p[0]=1
	movl	28(%esp), %eax //p
	addl	$4, %eax //p+=4
	movl	$2, (%eax) //即p[1]=2
	movl	28(%esp), %eax
	addl	$8, %eax
	movl	$3, (%eax) //p[2]=3
	movl	28(%esp), %eax
	movl	%eax, (%esp) //将p压栈，即p指向的堆上内存的首地址
	call	free
	leave
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret
	.cfi_endproc
.LFE2:
	.size	main, .-main
	.ident	"GCC: (Ubuntu 4.8.4-2ubuntu1~14.04.1) 4.8.4"
	.section	.note.GNU-stack,"",@progbits


```
通过对比静态一维数组，和动态一维数组，可以知道，静态数组名为数组的首地址，但是并不占用内存（据此，可以实现0长度数组的妙用）。动态二维数组,在堆上分配的首地址保存在指针内，需要分配内存。
##### 1.1.3 0长度数组的妙用
```bash
typedef struct test{
	int i;
	int j;
	unsigned char ch[0];
}test;
其中的ch[0]即为0长度数组，sizeof(test)=8,
即ch为一个任意大小内存的指针，但unsigned ch[0]
并不占用test结构体变量的内存。
其用法如下：
int main()
{
        test *p=(test*)malloc(12);
        p->i=1;
        p->j=2;
        *((int*)(p->ch))=3;
        printf("%u\n",sizeof(test));
        printf("%d %d %d\n",p->i,p->j,*((int*)(p->ch)));
}
```
以上结构体指针p指向的堆上内存布局为：
![imgx](/img/c2-x.png)
##### 1.2静态二维数组和动态二维数组
###### 1.2.1静态二维数组

```bash
静态二维数组的内存布局即为一维数组，
假设int p[3][4];int *px; 另px=p；
则访问p[2][1]的元素，可以转换为px[2*16+1*4]元素。
int main()
{
    int p[3][4];
    p[1][0]=123;
    /*
     1, p+1是二维数组p中序号为1的行的首地址，
     而*(p+1)并不是p+1单元的内容，*(p+1)可以
     理解为由行地址的计算转向了列地址的计算。
     如：*(p+1)表示第1行，第0列的地址。
     2, 不要把&p[i]简单的理解为p[i]单元的地址，
     因为并没有给p[i]分配内存。&p[i]可以理解为
     由当前的列地址计算转向了行地址计算(&(*(p+i))=p+i)。
     */
    printf("%x\n%x\n%x\n%x\n",p[1],p+1,&p[1][0],&p[1]);
    return 0;
}
```
以上代码对应的内存布局为：
![img2](/img/c2-2.png)

###### 1.2.2动态二维数组
```bash
#include<stdio.h>
#include<stdlib.h>
int main()
{

	int **p=NULL;
	p=(int**)malloc(sizeof(int*)*3);
	p[0]=(int*)malloc(sizeof(int)*4);
	p[1]=(int*)malloc(sizeof(int)*4);
	p[2]=(int*)malloc(sizeof(int)*4);

	printf("%x\n%x\n%x\n%x\n",p[1],p+1,&p[1][0],&p[1]);
	free(p[0]);
	free(p[1]);
	free(p[2]);
	free(p);
   /* int **p=NULL;
    p=new int*[3];
    p[0]=new int[4];
    p[1]=new int[4];
    p[2]=new int[4];
   */ /*
        1, p+1是二维数组p中序号为1的行的首地址，而*(p+1)是p+1单元的内容。
        2, &p[i]可以理解为p[i]单元的地址。
    */
/*
    printf("%x\n%x\n%x\n%x\n",p[1],p+1,&p[1][0],&p[1]);
    
    delete[] p[0];
    delete[] p[1];
    delete[] p[2];
    delete[] p;
*/
    return 0;
}
```

以上代码对应的内存布局为：
![img3](/img/c2-3.png)


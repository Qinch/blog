title: 奇怪的死循环
category: Asm/C
date: 2013-10-19
tags: [Asm,C]
toc: false
comments: true
---

```bash                   
#include<stdio.h>
int main()
{
    int i;
    int a[10];
    for(i=0;i<=10;++i)
    {
        a[i]=0;
        printf("%d\n",a[i]);
    }
    return 0;
}
```
该程序对应的汇编代码见如下代码:
<!--more-->
```bash
.file "cs18.c"
        .section .rodata
.LC0:
        .string "%d\n"
        .text
.globl main
        .type main, @function71
main:
        pushl %ebp ;old ebp压栈
        movl %esp, %ebp ;ebp=esp
        andl $-16, %esp 
        subl $64, %esp  //esp=esp-64
        movl $0, 60(%esp)  //i=0
        jmp .L2
.L3:
        movl 60(%esp), %eax  //eax=i
        movl $0, 20(%esp,%eax,4) //即a[i]=0
        movl 60(%esp), %eax
        movl 20(%esp,%eax,4), %edx  //edx=a[i]
        movl $.LC0, %eax
        movl %edx, 4(%esp) //参数压栈
        movl %eax, (%esp) //参数压栈
        call printf  //调用printf函数
  addl $1, 60(%esp) //i+=1
.L2:
        cmpl $10, 60(%esp) //比较10和i
        jle .L3  //if i<=10 jmp .L3
        leave
        ret
        .size main, .-main
        .ident "GCC: (GNU) 4.4.7 20120313 (Red Hat 4.4.7-3)"
        .section .note.GNU-stack,"",@progbits
```


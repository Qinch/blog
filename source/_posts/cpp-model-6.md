title: C++对象模型-Non-Staitc成员函数
category: Asm/Cpp
date: 2016-4-9
tags: [Asm,C,Cpp]
toc: false
comments: false
---

##### 开发环境

gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.10)  
                   
##### 1, non-static成员函数调用过程

class X定义了一个virtual function foo:函数调用过程(栈帧)class X定义了一个virtual function foo:

```bash
#include<iostream>
using namespace std;

//不分析virtual dctor
class X{
        public:
                virtual void foo();

        private:
                int _x;
};

void X::foo()
{
        cout<<_x<<endl;
}

X foobar()
{
        X xx;
        X *px = new X;

        xx.foo();
        px->foo();

        delete px;
        return xx;
}

int main()
{
        foobar();
        return 0;
}
```


该程序对应的汇编代码如下（g++ -S -m32 p13.cc）：

```bash
_ZN1X3fooEv:  //X::foo()
.LFB1021:
        .cfi_startproc
        pushl   %ebp  //ebp压栈([esp]=ebp, esp=esp-4)
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp  //ebp=esp
        .cfi_def_cfa_register 5
        subl    $8, %esp  //esp=esp-8
        movl    8(%ebp), %eax  //eax=[ebp+8],即eax=this
        movl    4(%eax), %eax  //eax=[eax+4], 即eax=this->_x
        subl    $8, %esp  //esp=esp-8
        pushl   %eax  //[esp]=eax, esp=esp-4, 即this->_x压栈
        pushl   $_ZSt4cout //std::cout压栈
        call    _ZNSolsEi  //operator<< 对应两步操作：1，retAddr压栈 2，IP=operator<< func
        addl    $16, %esp
        subl    $8, %esp
        pushl   $_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
        pushl   %eax
        call    _ZNSolsEPFRSoS_E
        addl    $16, %esp
        nop
        leave  //ebp=esp, pop ebp
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret //pop IP
        .cfi_endproc
.LFE1021:
        .size   _ZN1X3fooEv, .-_ZN1X3fooEv
        .section        .text._ZN1XC2Ev,"axG",@progbits,_ZN1XC5Ev,comdat
        .align 2
        .weak   _ZN1XC2Ev
        .type   _ZN1XC2Ev, @function
```

<!--more-->
```bash

_ZN1XC2Ev: //X::X() 默认构造函数
.LFB1024:
        .cfi_startproc
        pushl   %ebp //ebp压栈
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp //ebp = esp
        .cfi_def_cfa_register 5
        movl    $_ZTV1X+8, %edx  //edx=vptr
        movl    8(%ebp), %eax  //eax=[ebp+8],即eax=this
        movl    %edx, (%eax) //[eax]=edx, 即[this]=vptr
        nop
        popl    %ebp  //ebp=old ebp
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
.LFE1024:
        .size   _ZN1XC2Ev, .-_ZN1XC2Ev
        .weak   _ZN1XC1Ev
        .set    _ZN1XC1Ev,_ZN1XC2Ev
        .text
        .globl  _Z6foobarv
        .type   _Z6foobarv, @function

```

```bash
_Z6foobarv: //foobar
.LFB1022:
        .cfi_startproc
        pushl   %ebp  //ebp压栈
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp //ebp=esp
        .cfi_def_cfa_register 5
        pushl   %ebx  //ebx压栈
        subl    $36, %esp //esp=esp-36
        .cfi_offset 3, -12
        movl    8(%ebp), %eax //eax=tmp.this (返回值的首地址，即this指针)
        movl    %eax, -28(%ebp) //[ebp-28]=tmp.this
        movl    %gs:20, %eax
        movl    %eax, -12(%ebp)
        xorl    %eax, %eax //eax=0
        subl    $12, %esp  //esp=esp-12
        pushl   -28(%ebp) //push xx.this, 即xx对象首地址压栈
        call    _ZN1XC1Ev //X::X(), 即默认构造函数
        addl    $16, %esp
        subl    $12, %esp
        pushl   $8  //分配内存operator new size=8,
        call    _Znwj //operator new
        addl    $16, %esp
        movl    %eax, %ebx  //eax为new函数返回值(px.this指针)， ebx=eax,
        subl    $12, %esp
        pushl   %ebx  //push px.this
        call    _ZN1XC1Ev   //X::X(),构造px指针指向的对象
        addl    $16, %esp
        movl    %ebx, -16(%ebp)  //[ebp-16]=px.this
        subl    $12, %esp
        pushl   -28(%ebp) //xx对象this指针压栈
        call    _ZN1X3fooEv  //xx.foo()
        addl    $16, %esp
        movl    -16(%ebp), %eax //eax=px.this
        movl    (%eax), %eax //eax=vptr
        movl    (%eax), %eax  //eax=[vptr],即eax=virtual foo()
        subl    $12, %esp
        pushl   -16(%ebp) //px.this压栈
        call    *%eax  //call virtual foo
        addl    $16, %esp
        subl    $12, %esp
        pushl   -16(%ebp)
        call    _ZdlPv  //delete px
        addl    $16, %esp
        nop
        movl    -28(%ebp), %eax
        movl    -12(%ebp), %edx
        xorl    %gs:20, %edx
        je      .L5
        call    __stack_chk_fail
.L5:
        movl    -4(%ebp), %ebx
        leave
        .cfi_restore 5
        .cfi_restore 3
        .cfi_def_cfa 4, 4
        ret     $4
        .cfi_endproc
```

```bash
main:
.LFB1029:
        .cfi_startproc
        leal    4(%esp), %ecx
        .cfi_def_cfa 1, 0
        andl    $-16, %esp
        pushl   -4(%ecx)
        pushl   %ebp
        .cfi_escape 0x10,0x5,0x2,0x75,0
        movl    %esp, %ebp
        pushl   %ecx
        .cfi_escape 0xf,0x3,0x75,0x7c,0x6
        subl    $36, %esp
        movl    %gs:20, %eax
        movl    %eax, -12(%ebp)
        xorl    %eax, %eax
        leal    -40(%ebp), %eax
        subl    $12, %esp
        pushl   %eax  //tmp.this压栈
        call    _Z6foobarv //foobar
        addl    $12, %esp
        movl    $0, %eax
        movl    -12(%ebp), %edx
        xorl    %gs:20, %edx
        je      .L8
        call    __stack_chk_fail
.L8:
        movl    -4(%ebp), %ecx
        .cfi_def_cfa 1, 0
        leave
        .cfi_restore 5
        leal    -4(%ecx), %esp
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```


#### 2,通过分析cpp代码对应的汇编，发现foobar函数在内部被转化为：

```bash
void foobar(X *result)
{
    //构造result指向的对象
    X::X(result);
    
    
    px = malloc(sizeof(X));
    X::X(px);

   
    //xx.foo()不采用virtual机制
    X::foo(result);
    
    //px->foo()采用virtual机制
    (px->vptr[0])(px);
  
    //略
    
}
```

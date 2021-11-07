title: C++对象模型-引用
category: Asm/Cpp
date: 2016-4-9
tags: [Asm,C,Cpp]
toc: false
comments: false
---

#### 0.测试环境

gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.10)  
                   

#### 1.引用

```bash
class Widget{
public:
        virtual void show()=0;

        int _w;
};

class Bell:public Widget{
public:
        void show()
        {
                cout<<"Bell Bell..."<<endl;
        }
};


void test()
{       
        Bell b;
        Widget &w =b;
        w.show();

}
```

test函数对应的汇编代码如下：

```bash
_Z4testv:
.LFB1022:
        .cfi_startproc
        pushl   %ebp
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp
        .cfi_def_cfa_register 5
        subl    $24, %esp
        movl    %gs:20, %eax
        movl    %eax, -12(%ebp)
        xorl    %eax, %eax
        subl    $12, %esp
        leal    -20(%ebp), %eax  //eax=ebp-20,即eax=b.this
        pushl   %eax  //b首地址压栈
        call    _ZN4BellC1Ev //Bell::Bell（）
        addl    $16, %esp
        leal    -20(%ebp), %eax //eax=b.this
        movl    %eax, -24(%ebp)  //Widget &w=b
        movl    -24(%ebp), %eax  //eax=&w
        movl    (%eax), %eax  //eax=vptr4Bell
        movl    (%eax), %eax //eax = vptr4Bell[0]
        subl    $12, %esp
        pushl   -24(%ebp)  //w.this压栈
        call    *%eax   //virtual机制，w.show()
        addl    $16, %esp
        nop
        movl    -12(%ebp), %eax
        xorl    %gs:20, %eax
        je      .L5
        call    __stack_chk_fail
.L5:
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```

<!--more-->
test函数对应的伪码如下:

```bash
test()
{
   Bell::Bell(&b);
   Widget *w=&b;
   (w->vptr[0])(w);
}
```

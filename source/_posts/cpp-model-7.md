title: C++对象模型-ObjectSliced
category: Asm/Cpp
date: 2016-4-9
tags: [Asm,C,Cpp]
toc: false
comments: false
---

##### 开发环境

gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.10)  
                   
##### 1, ObjectSliced

当一个base class object 被直接初始化（copy ctor）/赋值(operator =)为一个derived class object 时，derived object的base 部分会被切割(sliced)以塞入base type内存中，derived type将没有留下任何蛛丝马迹(即：base class object 的vptr不会被derived class object的vptr替换)

```bash
#include<iostream>
using namespace std;

class ZooAnimal{
        public:
                ZooAnimal(int l):loc(l)
                {
                }

                virtual void print()
                {
                        cout<<"ZooAnimal";
                }

        private:
                int loc;
};


class Bear : public ZooAnimal{
        public:
                Bear(int c, int l):ZooAnimal(l), cell(c)
                {
                }

        private:
                void print()
                {
                        cout<<"Bear";
                }

        private:
                int cell;
};

int main()
{
        Bear b(1, 2);
        ZooAnimal z = b;
        z.print();
}
```

以上代码对应的汇编如下(g++ -S -m32 p27.cc):

```bash
_ZN9ZooAnimalC2ERKS_: //ZooAnimal默认copy ctor
.LFB1031:
        .cfi_startproc
        pushl   %ebp
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp
        .cfi_def_cfa_register 5
        movl    $_ZTV9ZooAnimal+8, %edx //edx=ZooAnimal.vptr
        movl    8(%ebp), %eax  //eax=[ebp+8],即eax=z.this
        movl    %edx, (%eax) //设置z的vptr
        movl    12(%ebp), %eax //eax=b.this
        movl    4(%eax), %edx //edx = b.loc
        movl    8(%ebp), %eax //eax=z.this
        movl    %edx, 4(%eax) //z.loc = b.loc
        nop
        popl    %ebp
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```

<!--more-->

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
        subl    $4, %esp
        pushl   $2
        pushl   $1
        leal    -24(%ebp), %eax  //eax=ebp-24, b对象首地址
        pushl   %eax  //b的首地址压栈，即b.this压栈
        call    _ZN4BearC1Eii //Bear::Bear(b.this, 1, 2)
        addl    $16, %esp
        subl    $8, %esp
        leal    -24(%ebp), %eax //eax = ebps-24, 即eax=b.this
        pushl   %eax //b.this压栈
        leal    -32(%ebp), %eax //eax=ebp-32,即eax=z.this
        pushl   %eax //z.this压栈
        call    _ZN9ZooAnimalC1ERKS_ //ZooAnimal::ZooAnimal(ZooAnimal const&)
        addl    $16, %esp
        subl    $12, %esp
        leal    -32(%ebp), %eax //eax=z.this
        pushl   %eax //z.this压栈
        call    _ZN9ZooAnimal5printEv  //ZooAnimal::print()
        addl    $16, %esp
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
#### 2.图示

![ObjectSliced](/img/cppos.png)

#### 3.参考
《深度探索C++对象模型》

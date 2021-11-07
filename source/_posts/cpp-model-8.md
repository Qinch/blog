title: C++对象模型-Default Constructor
category: Asm/Cpp
date: 2016-4-9
tags: [Asm,C,Cpp]
toc: false
comments: false
---

#### 0.测试环境

gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.10)  
                   
#### 1, 默认构造函数在编译器需要的时产生出来

在如下片段的代码中, 通过分析汇编代码，发现并不会合成出来一个Default Constructor函数，因为如下代码是代码逻辑需要一个默认构造函数来初始化val和pnext数据成员。而不是编译器需要合成一个Default Constructor.
```bash
#include<iostream>

using namespace std;


class Foo{
        public:
                int val;
        Foo *pnext;
};

void foobar()
{
        Foo bar;
        if(bar.val || bar.pnext) //代码逻辑需要一个Default Constructor来初始化val和pnext数据成员
        {
                cout<<bar.val<<endl;
        }
}

```

以上代码对应的汇编代码如下(g++ -S -m32 p39.cc):

```bash
_Z6foobarv: //foobar(), c++filt _Z6foobarv
.LFB1021:
        .cfi_startproc
        pushl   %ebp
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp
        .cfi_def_cfa_register 5
        subl    $24, %esp
        movl    -16(%ebp), %eax  //eax=[ebp-16],即eax = bar.val  ,NOTICE:NO Default Constructor is Called
        testl   %eax, %eax //eax&eax
        jne     .L2 // 如果testl的结果非0，jmp .L2 ,jne means jmp if not equal
        movl    -12(%ebp), %eax //eax=[ebp-12], 即eax=bra.pnext
        testl   %eax, %eax
        je      .L4  //如果testl的结果为0, jmp .L4
.L2:
        movl    -16(%ebp), %eax  //eax=[ebp-16], 即bar.val
        subl    $8, %esp
        pushl   %eax
        pushl   $_ZSt4cout
        call    _ZNSolsEi
        addl    $16, %esp
        subl    $8, %esp
        pushl   $_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
        pushl   %eax
        call    _ZNSolsEPFRSoS_E
        addl    $16, %esp
.L4:
        nop
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```

<!--more-->

#### 2. nontrivial Default Constructor

nontrivial Default Constrctor 即编译器所需要的默认构造函数，必要时候会由编译器合成出一个。

##### 2.1 “带有Default Constructor"的Member Class Object 

```bash
                                                                                                                                                                                        1,1           All
#include<iostream>
using namespace std;


class Foo{
        public:
                Foo():_f(0)
                {
                }
        private:
                int _f;
};

class Bar{
        public:
                char *str;
                Foo foo;
		/*
		  NOTICE:即使写了默认构造函数，Bar的数据成员的初始化顺序也是:按数据成员的声明顺序初始化，即
			先初始化str，然后调用foo的默认构造函数
			Bar():foo(), str(NULL)
			{
			}
		*/
};

void test()
{
        Bar bar;
        if(bar.str)
        {
                cout<<bar.str<<endl;
        }
}

```

test函数对应的汇编代码如下:

```bash
_Z4testv:
.LFB1024:
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
        leal    -20(%ebp), %eax  //eax=bar的首地址
        pushl   %eax   //bar对象首地址压栈
        call    _ZN3BarC1Ev  //Bar::Bar()
        addl    $16, %esp
        movl    -20(%ebp), %eax
        testl   %eax, %eax
        je      .L6
        movl    -20(%ebp), %eax
        subl    $8, %esp
        pushl   %eax
        pushl   $_ZSt4cout
        call    _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
        addl    $16, %esp
        subl    $8, %esp
        pushl   $_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
        pushl   %eax
        call    _ZNSolsEPFRSoS_E
        addl    $16, %esp
.L6:
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

Bar构造函数对应的汇编代码如下:

```bash
_ZN3BarC2Ev:  //Bar::Bar()
.LFB1026:
        .cfi_startproc
        pushl   %ebp
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp
        .cfi_def_cfa_register 5
        subl    $8, %esp
        movl    8(%ebp), %eax  //eax=this
        addl    $4, %eax  //eax=eax+4,即foo的首地址
        subl    $12, %esp
        pushl   %eax  //foo对象的this指针压栈
        call    _ZN3FooC1Ev //Foo:Foo(), NOTICE：初始化Bar::foo时编译器的责任，初始化Bar::str时代码逻辑的责任
        addl    $16, %esp
        nop
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc

```

Bar::Bar()函数伪码如下:

```bash
Bar的default construtor合成如下:
	Bar::Bar()
	{
		Foo::Foo(&foo);
	}
```

通过分析以上代码，发现: 如果一个class没有任何Constructor，但他的数据成员含有member object(该object有默认构造函数)，那么
该class的implicit default  constructor 就是nontrivial的，编译器会为该class合成一个default contrutor.


#### 3.”带有Default Construtor“的Base Class


```bash
nclude<iostream>
using namespace std;


class Foo{
        public:
                Foo():_f(0)
                {
                }
        private:
                int _f;
};

class Bar :public Foo{
        public:
                char *str;
                Foo foo;
};

void test()
{
        Bar bar;
        if(bar.str)
        {
                cout<<bar.str<<endl;
        }
}
```

对应的汇编代码如下(g++ -S -m32 p44.cc):

```bash
_ZN3BarC2Ev:  //Bar::Bar()
.LFB1026:
        .cfi_startproc
        pushl   %ebp
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp
        .cfi_def_cfa_register 5
        subl    $8, %esp
        movl    8(%ebp), %eax  //eax=[ebp+8], eax=this
        subl    $12, %esp
        pushl   %eax //this指针压栈
        call    _ZN3FooC2Ev  //Foo:Foo()
        addl    $16, %esp
        nop
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
.LFE1026:
        .size   _ZN3BarC2Ev, .-_ZN3BarC2Ev
        .weak   _ZN3BarC1Ev
        .set    _ZN3BarC1Ev,_ZN3BarC2Ev
        .text
        .globl  _Z4testv
        .type   _Z4testv, @function
_Z4testv:
.LFB1024:
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
        leal    -20(%ebp), %eax  //eax=bar的首地址
        pushl   %eax  //bar.this压栈
        call    _ZN3BarC1Ev //call Bar::Bar()
        addl    $16, %esp
        movl    -16(%ebp), %eax
        testl   %eax, %eax
        je      .L6
        movl    -16(%ebp), %eax
        subl    $8, %esp
        pushl   %eax
        pushl   $_ZSt4cout
	//略
```

Bar::Bar()的伪码如下：

```bash
	Bar::Bar(&bar)
	{
		Foo:Foo(&bar):
	}
```

通过分析代码，发现:如果一个没有任何construtor的class继承了一个带有default construtor的base class, 那么该derived class的cdefualt constrtor会被编译器合成出来，该defaulut constructor为nontrivial

#### 4. "带有virtual function"的class

```bash
#include<iostream>
using namespace std;

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

对应的汇编代码如下:

```bash
_ZN6WidgetC2Ev:  //Widget::Widget()
.LFB1025:
        .cfi_startproc
        pushl   %ebp
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp
        .cfi_def_cfa_register 5
        movl    $_ZTV6Widget+8, %edx  //edx = vptr
        movl    8(%ebp), %eax  //eax=this
        movl    %edx, (%eax)  //[this]=vptr
        nop
        popl    %ebp
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc


_ZN4BellC2Ev:  //Bell:Bell()
.LFB1027:
        .cfi_startproc
        pushl   %ebp
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp
        .cfi_def_cfa_register 5
        subl    $8, %esp
        movl    8(%ebp), %eax  //eax=this
        subl    $12, %esp //esp=esp-12
        pushl   %eax //this压栈
        call    _ZN6WidgetC2Ev  //调用base class的默认构造函数, Widget:Widget()
        addl    $16, %esp
        movl    $_ZTV4Bell+8, %edx //edx=vptr4Bell
        movl    8(%ebp), %eax //eax=this
        movl    %edx, (%eax) //[this]=vptr4Bell
        nop
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc


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

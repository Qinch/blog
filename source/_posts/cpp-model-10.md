title: C++对象模型-virtual继承
category: Asm/Cpp
date: 2016-4-9
tags: [Asm,C,Cpp]
toc: false
comments: false
---

#### 0.测试环境

gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.10)  
                   

#### 1.虚继承

```bash

        	X
      	       / \
      virtual /   \virtual
             /     \
	     A	    B	
             \     /
              \   /
               \ /
                C

````

```bash
class X{
        public:
                int i;
        virtual void show()
        {
                cout<<"X"<<endl;
        }
        virtual ~X()
        {
                cout<<"~X";
        }
};

class A:public virtual X{
        public:
                int j;

        virtual void show()
        {
                cout<<"A"<<endl;
        }
        virtual ~A()
        {
                cout<<"~A";
        }
};

class B:public virtual X{
        public:
                int d;
        virtual void show()
        {
                cout<<"B"<<endl;
        }
        virtual ~B()
        {
                cout<<"~B";
        }
};

class C:public A, public B{
        public:
                int k;
        virtual void show()
        {
                cout<<"C"<<endl;
        }
        virtual ~C()
        {
                cout<<"~C";
        }
};

void test()
{
        B *b= new C;
        b->show();
        delete b;

}

```

#### 1.1 test()函数对应的汇编代码如下

```bash

_Z4testv:  //test
.LFB1041:
        .cfi_startproc
        pushl   %ebp  //ebp压栈
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp //ebp=esp
        .cfi_def_cfa_register 5
        pushl   %ebx //ebx压栈
        subl    $20, %esp  //esp=esp-20
        .cfi_offset 3, -12
        subl    $12, %esp
        pushl   $28  //malloc的内存大小, Class C  Object的大小为28
        call    _Znwj //malloc内存
        addl    $16, %esp
        movl    %eax, %ebx  //ebx=eax, eax为malloc的内存的首地址，即C.this
        subl    $12, %esp
        pushl   %ebx  //C的首地址压栈
        call    _ZN1CC1Ev  //C::C()
        addl    $16, %esp
        testl   %ebx, %ebx  //ebx&ebx
        je      .L36  //if ebx&ebx=0, jmp
        leal    8(%ebx), %eax //eax=C.this+8
	/*
		|-------|<-----C.this
		|  A    |
		|-------|<-----b
		|  B    |
		|-------|
		|  C    |
		|-------|
		|  X    |
		|-------|
	*/
        jmp     .L37
.L36:
        movl    $0, %eax
.L37:
        movl    %eax, -12(%ebp)  //b=C.this+8, 即b=BastTypeB.this, 对应的代码为b=new C;
        movl    -12(%ebp), %eax //eax=b
        movl    (%eax), %eax //eax=vptr
        movl    (%eax), %eax //eax=vptr[0]
        subl    $12, %esp
        pushl   -12(%ebp)  //b的地址压栈
        call    *%eax  //b->show()
        addl    $16, %esp
        cmpl    $0, -12(%ebp)
        je      .L39
        movl    -12(%ebp), %eax  //eax=b
        movl    (%eax), %eax //eax=vptr
        addl    $8, %eax //eax=vptr+8
        movl    (%eax), %eax //eax=vptr[2], virtual机制
        subl    $12, %esp
        pushl   -12(%ebp) //b压栈
        call    *%eax //call vptr[2], delete b
        addl    $16, %esp
.L39:
	//略
```

<!--more-->

#### 1.2 _ZN1CC1Ev对应的汇编代码如下：

```bash
_ZN1CC1Ev:  //C:C()
.LFB1053:
        .cfi_startproc
        pushl   %ebp
        .cfi_def_cfa_offset 8
        .cfi_offset 5, -8
        movl    %esp, %ebp
        .cfi_def_cfa_register 5
        subl    $8, %esp
        movl    8(%ebp), %eax //eax=this
        addl    $20, %eax  //base type X的首地址

	/*
		|-------|<-----C.this
		|  A    |
		|-------|<-----b
		|  B    |
		|-------|
		|  C    |
   Offset 20 -->|-------|<-----eax
		|vptr   |
		|-------|
	*/
        jmp     .L37
        subl    $12, %esp
        pushl   %eax  //压栈
        call    _ZN1XC2Ev  //X:X()
	/* _ZN1XC2Ev对应的代码如下:

		_ZN1XC2Ev: //X::X()
		.LFB1044:
        	.cfi_startproc
        	pushl   %ebp
        	.cfi_def_cfa_offset 8
        	.cfi_offset 5, -8
        	movl    %esp, %ebp
        	.cfi_def_cfa_register 5
        	movl    $_ZTV1X+8, %edx  //dex=vptr4X
       		 movl    8(%ebp), %eax  //eax=this
        	movl    %edx, (%eax)  //[this]=vptr4X
        	nop
        	popl    %ebp
        	.cfi_restore 5
        	.cfi_def_cfa 4, 4
        	ret
        	.cfi_endproc
	*/
        addl    $16, %esp
        movl    $_ZTT1C+4, %edx  //edx=VTT+4
        movl    8(%ebp), %eax //eax=this
        subl    $8, %esp
        pushl   %edx
        pushl   %eax
        call    _ZN1AC2Ev //A::A()
	/*
		_ZN1AC2Ev:
		.LFB1047:
        	.cfi_startproc
        	pushl   %ebp
        	.cfi_def_cfa_offset 8
        	.cfi_offset 5, -8
        	movl    %esp, %ebp
        	.cfi_def_cfa_register 5
        	movl    12(%ebp), %eax  //eax=VTT+4
       		movl    (%eax), %edx //edx=VTT[1], edx=vptr
        	movl    8(%ebp), %eax  //eax=this
        	movl    %edx, (%eax) //[this]=vptr
        	movl    8(%ebp), %eax //eax=this
        	movl    (%eax), %eax
        	subl    $12, %eax  //eax=vptr-12,即vtable+0获取offset to X
        	movl    (%eax), %eax //eax=vtable[0], eax=20
        	movl    %eax, %edx //edx=eax
        	movl    8(%ebp), %eax //eax=this
        	addl    %eax, %edx //edx=this+offset2X, 即edx=this+20
        	movl    12(%ebp), %eax //eax=VTT+4
        	movl    4(%eax), %eax //eax=eax+4,即eax=VTT+8
        	movl    %eax, (%edx) //[edx] =VTT+8,即设置X的vptr(construction vtable for A-in-C)
        	nop
        	popl    %ebp
        	.cfi_restore 5
        	.cfi_def_cfa 4, 4
        	ret
        	.cfi_endproc
	*/
         addl    $16, %esp
        movl    $_ZTT1C+12, %edx //edx=VTT(VTT for C)+12
        movl    8(%ebp), %eax //eax=this
        addl    $8, %eax //eax=eax+8,即base type B的地址
        subl    $8, %esp
        pushl   %edx //PUSH VTT+12, the address of vptr(_ZTC1C8_1B+12)
        pushl   %eax //PUSH B.this
        call    _ZN1BC2Ev //B::B()
	/* 分析略
		_ZN1BC2Ev:
		.LFB1050:
        	.cfi_startproc
        	pushl   %ebp
        	.cfi_def_cfa_offset 8
        	.cfi_offset 5, -8
       		movl    %esp, %ebp
        	.cfi_def_cfa_register 5
        	movl    12(%ebp), %eax
        	movl    (%eax), %edx
        	movl    8(%ebp), %eax
        	movl    %edx, (%eax)
        	movl    8(%ebp), %eax
        	movl    (%eax), %eax
        	subl    $12, %eax
        	movl    (%eax), %eax
        	movl    %eax, %edx
        	movl    8(%ebp), %eax
        	addl    %eax, %edx
        	movl    12(%ebp), %eax
        	movl    4(%eax), %eax
       		movl    %eax, (%edx)
        	nop
        	popl    %ebp
       		.cfi_restore 5
        	.cfi_def_cfa 4, 4
        	ret
        	.cfi_endproc
	*/
        addl    $16, %esp
        movl    $_ZTV1C+12, %edx  //edx=VTT+12
        movl    8(%ebp), %eax //eax=this
        movl    %edx, (%eax) //[this]=VTT+12
        movl    $20, %edx //edx=20
        movl    8(%ebp), %eax //eax=this
        addl    %edx, %eax //eax=this+20
        movl    $_ZTV1C+64, %edx //edx=vptr4C
        movl    %edx, (%eax) //[this]=vptr4C
        movl    $_ZTV1C+36, %edx
        movl    8(%ebp), %eax //eax=this
        movl    %edx, 8(%eax) //[this+8]=vptr4C
        nop
        leave
        .cfi_restore 5
        .cfi_def_cfa 4, 4
        ret
        .cfi_endproc
```
#### 1.3 delete b对应的汇编代码如下：

大概意思就是：delete b，就是将base type class b的地址压栈，但是编译器生成的代码会做一层转换，自动将压栈的地址-8，即将C.this压栈，然后析构，然后free


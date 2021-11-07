title: C++对象模型-关于对象
category: Asm/Cpp
date: 2016-4-9
update: 2017-1-1
tags: [Asm,C,Cpp]
toc: true
comments: false
---

##### 开发环境     
- Ubuntu 14.04(32bits)
- GCC      
- 编辑器 Cmd Markdown
- 画图工具 Processon    
                   
##### 1,关于对象        
从这篇博客开始真正介绍C++对象模型，前边BB了那么多没用的，终于开始了C++对模型的分析。关于C++对象模型的介绍，我将根据《深度探索C++对象模型》这本书，其书中的每一章，对应一篇博客，博客内容为自己对这本书的理解和补充吧。
###### 1.1C语言中的struct
我们知道，C语言是面向过程的，即数据和处理数据的函数时分开的，也就是说，struct中不能包含函数（当然也不能包含static变量）。但是我们可以在struct中声明指向函数的指针来模拟数据和处理数据的函数指针。其中，需要指出的是，c语言中struct变量的数据成员的访问方式都是基于该结构体变量的首地址的偏移量来访问的（同样适用于C++中的class/struct）。
即，见如下代码:
```bash
#include<stdio.h>

struct point3d;
typedef void (*initfun)(struct point3d*);

struct point3d{
        float x;
        float y;
        float z;
        initfun init;//指向初始化数据函数的指针
};
void initp(struct point3d* p3d) //struct point3d变量初始化函数
{
        p3d->x=1;
        p3d->y=2;
        p3d->z=3;
        printf("%f %f %f\n",p3d->x,p3d->y,p3d->z);
        return;
}

int main()
{
        struct point3d pd;
        pd.init=initp;//将initp函数地址赋给结构体中指向函数的指针pd.init。
        pd.init(&pd);
}

```
<!--more-->
###### 1.2 class
需要指出的是，C++类的非static的成员函数都有一个隐式的参数，即this(class object *const this)指针（对象的首地址）。在gcc，通过压栈来传递this指针。在vc6.0,通过ecx寄存器来传递this指针。
```bash

#include<iostream>
using namespace std;

class point3d{
        private:
                float x;
                float y;
                float z;
                friend ostream& operator<<(ostream&,point3d&);
        public:
                point3d():x(1),y(2),z(3){}
};

//该函数不能作为class的成员函数，因为它的第一个参数不是point3d的对象！
//设置为友元函数，可以访问class point3d的私有数据成员。
inline ostream& operator <<(ostream& os, point3d& pt)
{
        os<<pt.x<<' '<<pt.y<<' '<<pt.z<<endl;
        return os;
}

int main()
{
        point3d pd;
        cout<<pd;
}

```
以上程序对应的汇编代码为（C++ name mangling）：
```bash
	.file	"c5-1.cpp"
	.local	_ZStL8__ioinit
	.comm	_ZStL8__ioinit,1,1
	.section	.text._ZN7point3dC2Ev,"axG",@progbits,_ZN7point3dC5Ev,comdat
	.align 2
	.weak	_ZN7point3dC2Ev
	.type	_ZN7point3dC2Ev, @function
_ZN7point3dC2Ev:  //即函数point3d::point3d()
.LFB972:
	.cfi_startproc
	pushl	%ebp  //ebp压栈
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp  //ebp=esp
 	.cfi_def_cfa_register 5
	movl	8(%ebp), %edx //将this(即point3d对象的首地址)之后怎赋值给edx,即edx=this
	movl	.LC0, %eax //即eax=1
	movl	%eax, (%edx)//edx即为class point3d中x的首地址，即this->x=1
	movl	8(%ebp), %edx //edx=this
	movl	.LC1, %eax //eax=2
	movl	%eax, 4(%edx) //edx+4即为y的首地址，即this->y=2
	movl	8(%ebp), %edx //edx=this
	movl	.LC2, %eax //eax=3
	movl	%eax, 8(%edx) //this->z=3,其中edx+8即this->z 
	popl	%ebp //ebp=old ebp,esp+=4
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret //eip=ret addr, esp+=4
	.cfi_endproc
.LFE972:
	.size	_ZN7point3dC2Ev, .-_ZN7point3dC2Ev
	.weak	_ZN7point3dC1Ev
	.set	_ZN7point3dC1Ev,_ZN7point3dC2Ev
	.section	.text._ZlsRSoR7point3d,"axG",@progbits,_ZlsRSoR7point3d,comdat
	.weak	_ZlsRSoR7point3d
	.type	_ZlsRSoR7point3d, @function

main:
.LFB975:
	.cfi_startproc
	pushl	%ebp  //ebp压栈
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp  //ebp=esp
	.cfi_def_cfa_register 5
	andl	$-16, %esp //esp为16的整数倍
	subl	$32, %esp //esp-=32
	leal	20(%esp), %eax //eax=esp+20,\
	即pd对象的首地址（也就是我们说的this指针）
	movl	%eax, (%esp) //将eax压栈，即将pd的\
	首地址压栈(需要重点指出的，this指针作为类\
	非static成员函数的隐式参数，gcc采用压栈方\
	式传递this指针，而VC6.0通过ecx寄存器来传递this指针)
	call	_ZN7point3dC1Ev //调用构造函数pointd3d()
	leal	20(%esp), %eax //esp+20即pd对象的首地址
	movl	%eax, 4(%esp) //eax压栈
	movl	$_ZSt4cout, (%esp) //cout压栈
	call	_ZlsRSoR7point3d  //调用operator<<函数
	movl	$0, %eax
	leave //esp=ebp,pop ebp;
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret
	.cfi_endproc
.LFE975:
	.size	main, .-main
	.type	_Z41__static_initialization_and_destruction_0ii, @function
_Z41__static_initialization_and_destruction_0ii:
.LFB983:
	.cfi_startproc
	pushl	%ebp
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp
	.cfi_def_cfa_register 5
	subl	$24, %esp
	cmpl	$1, 8(%ebp)
	jne	.L6
	cmpl	$65535, 12(%ebp)
	jne	.L6
	movl	$_ZStL8__ioinit, (%esp)
	call	_ZNSt8ios_base4InitC1Ev
	movl	$__dso_handle, 8(%esp)
	movl	$_ZStL8__ioinit, 4(%esp)
	movl	$_ZNSt8ios_base4InitD1Ev, (%esp)
	call	__cxa_atexit
.L6:
	leave
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret
	.cfi_endproc
.LFE983:
	.size	_Z41__static_initialization_and_destruction_0ii, .-_Z41__static_initialization_and_destruction_0ii
	.type	_GLOBAL__sub_I_main, @function
_GLOBAL__sub_I_main:
.LFB984:
	.cfi_startproc
	pushl	%ebp
	.cfi_def_cfa_offset 8
	.cfi_offset 5, -8
	movl	%esp, %ebp
	.cfi_def_cfa_register 5
	subl	$24, %esp
	movl	$65535, 4(%esp)
	movl	$1, (%esp)
	call	_Z41__static_initialization_and_destruction_0ii
	leave
	.cfi_restore 5
	.cfi_def_cfa 4, 4
	ret
	.cfi_endproc
.LFE984:
	.size	_GLOBAL__sub_I_main, .-_GLOBAL__sub_I_main
	.section	.init_array,"aw"
	.align 4
	.long	_GLOBAL__sub_I_main
	.section	.rodata
	.align 4
.LC0:
	.long	1065353216
	.align 4
.LC1:
	.long	1073741824
	.align 4
.LC2:
	.long	1077936128
	.hidden	__dso_handle
	.ident	"GCC: (Ubuntu 4.8.4-2ubuntu1~14.04.1) 4.8.4"
	.section	.note.GNU-stack,"",@progbits

```
其中调用构造函数point3d::point3d()时的栈帧结构为：
![statck0](/img/c5-x.png)

###### 1.2.1 class对象内存布局

C++在内存布局以及存取时间上主要的额外负担是虚函数（即链接时的多态）和虚继承（即多次出现在继承体系中的父类，在子类对象中有一个单一共享的实例，其最典型的是菱形继承）
另外，需要指出的是，C++中class和struct关键字都可以包含函数，其不同点在于，class默认为private，struct默认为public.在继承时，class默认为私有继承，strutc默认为公有继承。
```bash
#include<iostream>
using namespace std;

class point{
        float x;
public:
        point():x(1){}
	virtual void draw()
	{
		cout<<"draw point"<<endl;
	}
        virtual ~point()
        {//需要将析构函数声明为虚函数(用于动态绑定)
                cout<<"point"<<endl;
        }
};

class point2d:public point{
        float y;
public:
        point2d():point(),y(2){}
        ~point2d()
        {
                cout<<"point2"<<endl;
        }

};

int main()
{
        point *p=new point2d;
        delete p;
}   
```
class point的对象对应的内存布局
![layout1](/img/c5-0.png)
class point2d的对象对应的内存布局
![layout2](/img/c5-1.png)

通过对比point和point2d的对象内存布局，可知，如果父类中定义了虚函数，并且在子类中进行了重写，则在子类的对象模型中，用子类重写的函数的地址将父类的虚函数地址替换掉，否则不进行替换。
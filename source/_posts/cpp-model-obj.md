title: C++对象模型_Class Obj作为函数参数
category: Asm/Cpp
date: 2014-5-10 
tags: [Asm,C,Cpp]
toc: false
---

##### 开发环境     
- VC6.0     
- 编辑器 Cmd Markdown

1. 关于C/C++中基本类型（如：int,int*等）作为函数参数时，是通过将该变量的值压栈来进行参数传递；本文通过C++反汇编代码分析了当对象作为函数参数时（该形参非引用或指针），参数如何传递以及此时栈帧的结构。

2.  对象作为函数参数时，参数传递过程(如：函数的声明为：void show(class Object obj);该函数的调用的为show(arg);其中实参arg的类型为class Object)：1,在栈顶上为obj对象分配内存空间，然后将对象arg的首地址压栈;2,调用拷贝构造函数（此为C++中三种调用拷贝构造函数情况之一），将arg的数据成员拷贝至obj;3,执行show()函数体（此时，ebp+8即为obj的首地址）。

3. 具体分析过程，见代码注释。

```bash

//C++源码。
//VC6.0

#include<iostream>
#includestdio>
using namespace std;

class CBase{
 int i;
public:
 CBase(int i=0)
 {
  this->i=i;
 }
 CBase(CBase& rhs) //拷贝构造函数。
 {
  i=rhs.i;
  printf("拷贝构造函数=%d\n",i);
 }
 void show(CBase B1, CBase B2) //对象作为形参。
 {
  printf("show=%d,%d\n",B1.i,B2.i);
 }
};
int main()
{
 CBase Base;
 CBase Basex(1);
 CBase Basexx(2);

 Base.show(Basex,Basexx);
// printf("this is an end!\n");
}
---------------------------------------------------------------------------------
//main函数对应的汇编代码。
 int main()
24: {
0040D4A0 push ebp
0040D4A1 mov ebp,esp
0040D4A3 sub esp,54h
0040D4A6 push ebx
0040D4A7 push esi
0040D4A8 push edi
0040D4A9 lea edi,[ebp-54h]
0040D4AC mov ecx,15h
0040D4B1 mov eax,0CCCCCCCCh
0040D4B6 rep stos dword ptr [edi]
25: CBase Base;
0040D4B8 push 0
0040D4BA lea ecx,[ebp-4] //ecx保存的是对象Base的this指针。
0040D4BD call @ILT+0(CBase::CBase) (00401005)//调用构造函数
26: CBase Basex(1);
0040D4C2 push 1
0040D4C4 lea ecx,[ebp-8] //ecx保存的是对象Basex的this指针。
0040D4C7 call @ILT+0(CBase::CBase) (00401005)//调用构造函数
27: CBase Basexx(2);
0040D4CC push 2
0040D4CE lea ecx,[ebp-0Ch] //ecx保存的是对象Basexx的this指针。
0040D4D1 call @ILT+0(CBase::CBase) (00401005)//调用构造函数

28: Base.show(Basex,Basexx);
0040D4D6 push ecx //等价于esp=esp-4;
0040D4D7 mov ecx,esp //ecx保存的是show函数的形参B2的this指针。
0040D4D9 lea eax,[ebp-0Ch]
0040D4DC push eax //对Basexx的this（即Basexx对象的首地址）指针压栈。
0040D4DD call @ILT+10(CBase::CBase) (0040100f)//调用拷贝构造函数
0040D4E2 push ecx //等价于esp=esp-4;
0040D4E3 mov ecx,esp //ecx保存的是show函数的形参B1的this指针。
0040D4E5 lea edx,[ebp-8]
0040D4E8 push edx //对Basex的this（即Basex对象的首地址）指针压栈。
0040D4E9 call @ILT+10(CBase::CBase) (0040100f)//调用拷贝构造函数
0040D4EE lea ecx,[ebp-4] //ecx保存的是Base对象的this指针。
0040D4F1 call @ILT+20(CBase::show) (00401019)//调用show函数。
29: printf("this is an end!\n");
0040D4F6 push offset string "this is an end!\n" (00422fa4)
0040D4FB call printf (0040d7c0)
0040D500 add esp,4
30: }
0040D503 pop edi
0040D504 pop esi
0040D505 pop ebx
0040D506 add esp,54h
0040D509 cmp ebp,esp
0040D50B call __chkesp (004010b0)
0040D510 mov esp,ebp
0040D512 pop ebp
0040D513 ret

//show函数对应的汇编代码。
16: void show(CBase B1,CBase B2)
17: {
0040D550 push ebp
0040D551 mov ebp,esp
0040D553 sub esp,44h
0040D556 push ebx
0040D557 push esi
0040D558 push edi
0040D559 push ecx
0040D55A lea edi,[ebp-44h]
0040D55D mov ecx,11h
0040D562 mov eax,0CCCCCCCCh
0040D567 rep stos dword ptr [edi]
0040D569 pop ecx
0040D56A mov dword ptr [ebp-4],ecx
18: printf("show=%d,%d\n",B1.i,B2.i);
0040D56D mov eax,dword ptr [ebp+0Ch] //ebp+0Ch对应的是B2的首地址。
0040D570 push eax
0040D571 mov ecx,dword ptr [ebp+8] //ebp+8h对应的是B1的首地址。
0040D574 push ecx
0040D575 push offset string "show=%d,%d\n" (00422fcc)
0040D57A call printf (0040d7c0)
0040D57F add esp,0Ch
19: }
0040D582 pop edi
0040D583 pop esi
0040D584 pop ebx
0040D585 add esp,44h
0040D588 cmp ebp,esp
0040D58A call __chkesp (004010b0)
0040D58F mov esp,ebp
0040D591 pop ebp
0040D592 ret 8

//拷贝构造函数对应的汇编代码（注释略）。
11: CBase(CBase& rhs)
0040D840 push ebp
0040D841 mov ebp,esp
0040D843 sub esp,44h
0040D846 push ebx
0040D847 push esi
0040D848 push edi
0040D849 push ecx
0040D84A lea edi,[ebp-44h]
0040D84D mov ecx,11h
0040D852 mov eax,0CCCCCCCCh
0040D857 rep stos dword ptr [edi]
0040D859 pop ecx
0040D85A mov dword ptr [ebp-4],ecx
12: {
13: i=rhs.i;
0040D85D mov eax,dword ptr [ebp-4]
0040D860 mov ecx,dword ptr [ebp+8]
0040D863 mov edx,dword ptr [ecx]
0040D865 mov dword ptr [eax],edx
14: printf("拷贝构造函数=%d\n",i);
0040D867 mov eax,dword ptr [ebp-4]
0040D86A mov ecx,dword ptr [eax]
0040D86C push ecx
0040D86D push offset string "\xbf\xbd\xb1\xb4\xb9\xb9\xd4\xec\xba\xaf\xca\xfd=%d\n" (00422fb8)
0040D872 call printf (0040d7c0)
0040D877 add esp,8
15: }
0040D87A mov eax,dword ptr [ebp-4]
0040D87D pop edi
0040D87E pop esi
0040D87F pop ebx
0040D880 add esp,44h
0040D883 cmp ebp,esp
0040D885 call __chkesp (004010b0)
0040D88A mov esp,ebp
0040D88C pop ebp
0040D88D ret 4
```

4. 当执行show函数作用域内代码时，栈结构图示如下：

![stack](/img/obj.png)

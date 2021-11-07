title: C++对象模型_operator delete异常分析
category: Asm/Cpp
date: 2014-5-10 
tags: [Asm,C,Cpp]
toc: false
---

##### 开发环境     
- VC6.0     
- 编辑器 Cmd Markdown


1. C++中delete表达式执行的操作是：1，调用析构函数；2，释放对象内存（operator delete(...)）。

2. 如果父类的析构函数没有声明为virtual函数，且子类中至少存在一个virtual函数，此时将子类的对象地址赋值给父类指针。当对父类的指针执行delete操作时,会调用父类析构函数，然后在释放内存时（即delete表达式执行的操作的2，释放对象内存）出现崩溃。然而如果子类中不存在一个virtual函数时，执行上面同样的操作就不会出现崩溃。

- 原因分析如下：

```bash
//已知本示例 父类的析构函数应声明为virtual函数。但是由于本程序不需要析构函数执行特殊的操作，所以delete父类指针pB同样可以释放内存，然而引起出乎意料的内//存释放异常。因此对本程序进行如下分析
#include<iostream>
#include stdio>
using namespace std;

class Base
{
public:
 ~Base() {printf("\nBase::destructor.");}
};

class Derived: public Base
{
    virtual void show()
    {
        cout<<"show"<
    }
public:
    ~Derived(){printf("\nDerived::destructor.");}
};

int main()
{
    Base* pB=NULL;
    Derived *pD= new Derived;
    pB=pD;//此时pB得到的是(unsigned char*)pD+4的地址，所以执行operator delete时会发生崩溃（因为此时把vptr的内存当成了待释放的内存块的大小）。
    /*unsigned char *pC=(unsigned char*)pB;
      pC=pC-4;
      delete pC;//这样就可以释放new Derived分配的内存,而不会发生崩溃。
    */
    delete pB;
}
```

- 其main函数对应的汇编代码如下(VC6.0)：
```bash

int main()
23: {
00401060 push ebp
00401061 mov ebp,esp
00401063 push 0FFh
00401065 push offset __ehhandler$_main (00416a0b)
0040106A mov eax,fs:[00000000]
00401070 push eax
00401071 mov dword ptr fs:[0],esp
00401078 sub esp,64h
0040107B push ebx
0040107C push esi
0040107D push edi
0040107E lea edi,[ebp-70h]
00401081 mov ecx,19h
00401086 mov eax,0CCCCCCCCh
0040108B rep stos dword ptr [edi]

24：Base* pB=NULL;
0040108D mov dword ptr [ebp-10h],0
25: Derived *pD= new Derived;
00401094 push 4
00401096 call operator new (00403780)
0040109B add esp,4
0040109E mov dword ptr [ebp-1Ch],eax  //eax保存的是operator new（...）函数分配的内存的首地址。
004010A1 mov dword ptr [ebp-4],0
004010A8 cmp dword ptr [ebp-1Ch],0    //判断operator new(...)函数分配内存是否成功。
004010AC je main+5Bh (004010bb)
004010AE mov ecx,dword ptr [ebp-1Ch]   //调用Derived::Derived（）函数，ecx保存的是内存指针。
004010B1 call @ILT+35(Derived::Derived) (00401028)
004010B6 mov dword ptr [ebp-28h],eax
004010B9 jmp main+62h (004010c2)
004010BB mov dword ptr [ebp-28h],0
004010C2 mov eax,dword ptr [ebp-28h]
004010C5 mov dword ptr [ebp-18h],eax
004010C8 mov dword ptr [ebp-4],0FFFFFFFFh
004010CF mov ecx,dword ptr [ebp-18h]
004010D2 mov dword ptr [ebp-14h],ecx
26: pB=pD;
004010D5 cmp dword ptr [ebp-14h],0
004010D9 je main+86h (004010e6)
004010DB mov edx,dword ptr [ebp-14h]
004010DE add edx,4
004010E1 mov dword ptr [ebp-2Ch],edx
004010E4 jmp main+8Dh (004010ed)
004010E6 mov dword ptr [ebp-2Ch],0
004010ED mov eax,dword ptr [ebp-2Ch]
004010F0 mov dword ptr [ebp-10h],eax   //pB=(unsinged char*)(pD)+4
27:
28: delete pB;
004010F3 mov ecx,dword ptr [ebp-10h]
004010F6 mov dword ptr [ebp-24h],ecx
004010F9 mov edx,dword ptr [ebp-24h]
004010FC mov dword ptr [ebp-20h],edx
004010FF cmp dword ptr [ebp-20h],0
00401103 je main+0B4h (00401114)
00401105 push 1
00401107 mov ecx,dword ptr [ebp-20h]
0040110A call @ILT+20(Base::`scalar deleting destructor
```

- 图示如下

![cxxmodel](/img/del.png)

title: MFC thunk技术模拟 
category: Asm/Cpp
date: 2013-4-3
tags: [Asm, MFC, thunk]
toc: false
comments: false
---

//参考http://www.cnblogs.com/satng/archive/2010/12/30/2138833.html

```bash
#include<iostream>
using namespace std;

//thunk技术模拟

typedef void (*fun)(void *,int i);

class CFun;//类声明。

#pragma pack(push)
#pragma pack(1)
typedef struct Thunk{
    unsigned char call;
    int offset;
    fun pf;//函数指针。
    unsigned char code[5];
    CFun *ths;//this指针。
    unsigned char jmp;
    unsigned char ecx;
}Thunk;
#pragma pack(pop)

#define OFF(s,m) ((unsigned int)&((s*)0)->m)//求结构体的偏移量，s为结构体的类型，m为结构体的数据成员。

class CFun{
public:
    CFun()
    {
        createThunk();
    }
    ~CFun()
    {
        delete thunk;
    }
public:
    void createThunk()
    {
        Thunk* tk=new Thunk;
        //call des
        tk->call=0xE8;//call
        tk->offset=OFF(Thunk,code[0])-OFF(Thunk,pf);//des

        tk->pf=CFun::funx;//函数地址。
        //pop ecx
        //等价于：
        //mov ecx,[esp]
        //sub esp,4
        tk->code[0]=0x59;//pop ecx
        //mov [esp+4],this
        tk->code[1]=0xc7;//mov
        tk->code[2]=0x44;//dword ptr
        //4[esp]
        tk->code[3]=0x24;//[esp]
        tk->code[4]=0x04;//+4
        tk->ths=this;//修改栈，设置this指针。

        //jmp [ecx]
        tk->jmp=0xFF;//jmp
        tk->ecx=0x21;//[ecx]

        thunk=(fun)tk;

        return ;
    }
    static void funx(void *pFun,int i)
    {
        CFun *pf=(CFun*)pFun;
        pf->print(i);
    }
    void print(int i )
    {
        cout<<"Recevie="<<i<<endl;
    }

    fun GetThunk()
    {
        return thunk;
    }
private:
    fun thunk;
};

int main()
{
    CFun cf;
    fun pf=cf.GetThunk();
    pf("Hello",123);
    return 0;
}
```

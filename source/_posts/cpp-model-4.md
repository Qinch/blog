title: 基础类型的内存表示
category: Asm/Cpp
date: 2016-4-8
tags: [Asm,C,Cpp]
toc: false
---

##### 开发环境     
- Ubuntu 14.04(32bits)
- GCC      
- 编辑器 Cmd Markdown
- 画图工具 Processon    

##### 1,基础类型内存表示
[上一节](http://chinchao.xyz/2016/04/07/cpp-model-2/) 介绍了swich/if-else的实现机制。本文准备介绍一下基础类型的内存表示。
###### 1.1大端法和小端法
小端法，即数字的低位在低地址。大端法，即数字高位在低地址。大小端法的最小单位为byte,而不是bit.
举例：
```bash
int i=-16;
对应的小端法表示为（从低地址到高地址（从左到右为地址增长方向）对应的每个byte）：
0xf0 ff ff ff 
大端法表示为（从低地址到高地址（从左到右为地址增长方向）对应的每个byte）：
0x FF FF FF F0
```
在实际开发中，可以需要做大小端法互换，其代码如下：
```bash
void swap1(int *rhs)
{
    unsigned char *p=rhs;
    unsigned char tmp;
    tmp=p[0];
    p[0]=[3];
    p[3]=tmp;

    tmp=p[1];
    p[1]=[2];
    p[2]=tmp;
    return ;
}
```
在小端机上，如下程序的输出结果为-16.
```bash
int main()
{
        int i;
        unsigned char *p=(unsigned char*)&i;
        p[0]=0xF0;
        p[1]=0XFF;
        p[2]=0XFF;
        p[3]=0Xff;
        printf("%d\n",i);
}
```
<!--more-->
###### 1.2 int/unsigned int内存表示
在有符号整数，和无符号整数中，需要指出的是0扩展和符号拓展。
在开发中，不可避免的需要强制类型转换。此时需要明确到底是符号扩展还是0拓展。假设 n bytes的无符号，无论是扩展为有符号数（m bytes），还是扩展为无符号数(m bytes)，并且m>=n,则n通过0拓展（即最高位补0）为m bytes.假设 n bytes的有符号数，无论是扩展为有符号数（m bytes），还是扩展为无符号数(m bytes)，并且m>=n,则n通过符号拓展（n bytes最高位如果为0，则补0，如果为1则补1）为m bytes.

###### 1.3 float内存表示
float 32bits,其中1bit为符号为，8bits为阶码，23bits为尾码。
如下代码的输出结果为3.25
```bash
int main()
{
        float  i;
        unsigned char *p=(unsigned char*)&i;
        p[0]=0x00;
        p[1]=0x00;
        p[2]=0x50;
        p[3]=0x40;
        printf("%f\n",i);

}

```
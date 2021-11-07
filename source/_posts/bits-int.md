title: 24 bits解析为有符号整数
category: C/C++
date: 2013-1-10
tags: [bits,int]
toc: false
---
最近遇到了24bits来解析为有符号整数的问题，提供如下两个解决思路：

#### 方法1:    
24bits即从高位到低位为:buf[0],buf[1],buf[2] 三个bytes;通过将最高字节buf[0]符号扩展为两个字节（假设从高位到低位为ps[1],ps[0]），然后将ps[1],ps[0],buf[1],buf[2]拼接为int变量的四个字节;见下图;    
![pic1](/img/bits2int.png)
<!--more-->

#### 方法2    
类似于：十进制123=1乘以10^2+2乘以10^1+3乘以10^0；将buf[0],buf[1],buf[2]分别乘以相应的指数，即可得到相应的整型值;    

#### 代码如下：
``` bash
#include<stdio.h>

//方法1
int getdig1(char* buf,int n)
 {
        int ret=0;
        char *pret=(unsigned char*)&ret;
		short s=(short)buf[0];//buf[0]符号扩展为2个字节;
        char *ps=(unsigned char*)&s;
        pret[0]=buf[2];
        pret[1]=buf[1];
        
        pret[2]=ps[0];
        pret[3]=ps[1];
        return ret;
 }

//方法2
int getdig2(char *buf,int n)
{
	int ret=buf[0]<<16;
	ret|=((unsigned char)buf[1])<<8;
	ret|=(unsigned char)buf[2];
	return ret;

}
 void resolve(unsigned char* buf,int n,int  (*ret)[50],int fn)
 {
	 int i=0;
	 int k=0;
        for( i=0;i<1;i++)
        {
            for( k=7;k>=0;k--)
            {
                ret[k][i]=getdig1(buf+i*24+(7-k)*3,3);
                printf("%d\n",ret[k][i]);
            }
        }
 }
int main()
{
	int test[8][50];
	char recv[24]={0x0d,0xd7,0x00,0x0c,0xa1,0xff,0xf5,0x27,0xff,0xf7,0x60, \
	0xff,0xfb,0xfe,0x00,0x00,0xd3,0xff,0xf8,0x4a,0xff,0xfb,0x89,0x00 };

	resolve(recv,24,test,8);
}
```

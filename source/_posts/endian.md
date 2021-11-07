title: 大小端法互换
category: Asm/C
date: 2013-10-9
tags: [Asm,C,Endian]
toc: false
comments: false
---

#### 1.以int32类型为例:

- 方法1:

```bash      
void swapInt(int *rhs)
{
    unsigned char *p=rhs;
    unsigned char temp;
    temp=p[0];
    p[0]=[1];
    p[1]=temp;

    temp=p[1];
    p[1]=[2];
    p[2]=temp;
    return ;
}
```

- 方法2：

```bash
void swapInt(int *rhs)
{
    *rhs=(((*rhs)&0xff000000)>>24) | (((*rhs)&0x00ff0000)>>8) \
	 | (((*rhs)&0x000000ff)<<24) | (((*rhs)&0x0000ff00)<<8); 
}
```


#### 2.检测机器字节序：大端法or小端法

- 方法1：

```bash
int checkEndian()//检查主机字节顺序是否是大端法，如果是，返回1，否则返回0.
{
        int i=0x12345678;
        unsigned char *p=&i;
        return (0x12==p[0]);
}
```

- 方法2：

```bash
int checkEndian()
{
        union{
            unsigned int i;
            unsigned char s[4];
        }c;
        c.i=0x12345678;
        return (0x12==c.s[0]);
}
```

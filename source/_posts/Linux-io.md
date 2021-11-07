title: Linux内存流
date: 2015-12-23 15:25:29
category: Linux
tags: [apue, linux,内存流]

---
#### 1, 内存流
As we’ve seen, the standard I/O library buffers data in memory, so operations such as character-at-a-time I/O and line-at-a-time I/O are more efficient. We’ve also seen that we can provide our own buffer for the library to use by calling setbuf or setvbuf. In Version 4, the Single UNIX Specification added support for memory streams. These are standard I/O streams for which there are no underlying files, although they are still accessed with FILE pointers. All I/O is done by transferring bytes to and from buffers in main memory. 
``` bash
#include <stdio.h>     
FILE * fopen(const char * path,const char * mode);
FILE *fmemopen(void *restrict mem, size_t size, const char *restrict type);
Returns: stream pointer if OK, NULL on error
```
其中mem相当于fopen函数中的文件(即参数path)；如果mem参数为空，则fmemopen函数分配size字节数的内存（我们称作“anonmem”，当流关闭时anonmem会被释放)。
其中type参数取值的含义：   
1. whenever a memory stream is opened for append, the current file position is set to the first null byte in the buffer. If the buffer contains no null bytes, then the current position is set to one byte past the end of the buffer. When a stream is not opened for append, the current position is set to the beginning of the buffer. Because the append mode determines the end of the data by the first null byte, memory streams aren’t well suited for storing binary data (which might contain null bytes before the end of the data).      
2. a null byte is written at the current position in the stream whenever we increase the amount of data in the stream’s buffer and call fclose, fflush, fseek,fseeko, or fsetpos.(只有mem中的数据量增加（条件1）并且调用fclose,fflush,fseek,fseeko,fsetpos函数时（条件2），当前位置才会被写入一个null字节.如果条件1和条件2只有一个成立，则不会插入null)。
<!--more-->    

#### 2, 测试代码   

``` bash
/********************************************************
 > File Name: p138.c
 > Author: qinchao
 > Mail: 1187620726@qq.com
 > Created Date:2015-12-23 Time:01:11:47.
 *********************************************************/
#include<stdio.h>
#include<string.h>

#define BSZ 100
char buf[1000];
int main()
{
    FILE *fp;
	char mem[BSZ];
	memset(mem,'a',BSZ-2);
	buf[BSZ-2]='\0';
	buf[BSZ-1]='x';
	printf("memset mem=%s\n",mem);
	
	fp=fmemopen(mem,BSZ,"w+");//mem[0]='\0';
	printf("initial mem=%d %s\n",mem[0],mem);
	setbuf(fp,buf);//设置缓冲区;
	
	fprintf(fp,"hello world");
	printf("buff=%s\n",buf);
	printf("before fflush mem=:%s\n",mem);
	fflush(fp);
	printf("after fflush mem=%s\n",mem);

	fprintf(fp," qinchao");
	fseek(fp,0,SEEK_SET);
	//mem数据量增加和fseek函数调用，满足条件1和条件2，当前位置写入null;
	printf("after seek mem=%s\n",mem);
	
	fprintf(fp,"whut");
	fseek(fp,0,SEEK_SET);
	//fseek引起buf冲洗,只满足条件2，所以当前位置没有写入null;
	printf("after seek mem=%s\n",mem);
	fclose(fp);
}
```
#### 3, 测试输出

``` bash 
qin@qin-Lenovo-G450:~/apue/ch5$ gcc p138.c 
qin@qin-Lenovo-G450:~/apue/ch5$ ./a.out 
memset mem=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa \
	aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@
initial mem=0 
buff=hello world
before fflush mem=:
after fflush mem=hello world
after seek mem=hello world qinchao
after seek mem=whuto world qinchao
qin@qin-Lenovo-G450:~/apue/ch5$ 


```
#### 4, fmemopen源码分析

---待续

#### 5, 参考文献
 1,Advanced Programming in the UNIX Environment, Third Edition


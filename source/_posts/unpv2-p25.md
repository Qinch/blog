title: IPC-标识符重用
category: unpv2
date: 2017-04-07
tags: [unpv2,icp]
toc: false
---
##### 1,槽位使用情况序列号
ipc_perm结构中含有一个名为seq的变量，是一个槽位使用情况序列号。该变量是一个由内核为系统中每个潜在的IPC对象维持的计数器。每当删除一个IPC对象时，内核就递增相应槽位的使用情况序列号，若溢出则循环回0.

简单理解就是: 存在ipc_perm[max]，当创建ipc对象时，在ipc_perm数组中查找第一个ipc_perm元素为空，记为ipc_perm[i].当删除ipc_perm[i]对应的ipc时，ipc_perm[i].seq++,下次再次创建ipc对象时，假设ipc_perm[i]元素为空，则返回ipc id为i+ipc_perm[i].seq*max.
注：槽位使用情况序列号是一个跨进程保持的内核变量。
#### 2，示例代码    

##### 2.1 创建ipc，然后立即删除该ipc

对应的源码：

``` bash
/*************************************************************************
 > File Name: p26_1.c
 > Author: qinchao
 > Mail: 1187620726@qq.com
 > Created Date:2017-04-07 Time:11:02:25.
 ************************************************************************/
#include<stdio.h>
#include<sys/msg.h>
#include<sys/types.h>

int main()
{
        int i,msgid;
        for(i=0;i<10;i++)
        {
                msgid=msgget(IPC_PRIVATE,0666|IPC_CREAT);
                printf("msgid=%d\n",msgid);
                msgctl(msgid,IPC_RMID,NULL);
        }

        return 0;
}
```
对应的结果：

``` bash
bogon:unp qinchao$ gcc p26_1.c  -o p26_1
bogon:unp qinchao$ ./p26_1 
msgid=196608
msgid=262144
msgid=327680
msgid=393216
msgid=458752
msgid=524288
msgid=589824
msgid=655360
msgid=720896
msgid=786432
bogon:unp qinchao$ 
```
即：创建ipc1之后，ipc_perm[0]维护该ipc对象的相关信息，然后删除ipc1，ipc_perm[0].seq++;当创建ipc2时，仍然是ipc_perm[0]维护ipc2相关信息。
<!--more--> 
##### 2.2创建ipc之后，最后统一删除ipc
对应源码：即创建10个ipc，最后统一删除，每个ipc对应ipc_perm[0],ipc_perm[1],...,ipc_perm[9].

``` bash
/*************************************************************************
 > File Name: p26.c
 > Author: qinchao
 > Mail: 1187620726@qq.com
 > Created Date:2017-04-07 Time:11:02:25.
 ************************************************************************/
#include<stdio.h>
#include<sys/msg.h>
#include<sys/types.h>

int main()
{
        int i,msgid[10];
        for(i=0;i<10;i++)
        {
                msgid[i]=msgget(IPC_PRIVATE,0666|IPC_CREAT);
                printf("msgid=%d\n",msgid[i]);
        }
        for(i=0;i<10;i++)
                msgctl(msgid[i],IPC_RMID,NULL);//ipc_perm[i].seq++;

        return 0;
}
```

对应结果（为了便于观察，在2.1执行完之后，sudo rebot系统）:
``` bash
bogon:unp qinchao$ ./p26
msgid=65536
msgid=65537
msgid=65538
msgid=65539
msgid=65540
msgid=65541
msgid=65542
msgid=65543
msgid=65544
msgid=65545
bogon:unp qinchao$ ./p26
msgid=131072
msgid=131073
msgid=131074
msgid=131075
msgid=131076
msgid=131077
msgid=131078
msgid=131079
msgid=131080
msgid=131081
bogon:unp qinchao$ 
```
##### 2.3 创建ipc1,立即删除，然后创建ipc2,和ipc3
对应源码（sudo reboot）：

``` bash
/*************************************************************************
 > File Name: p26_2.c
 > Author: qinchao
 > Mail: 1187620726@qq.com
 > Created Date:2017-04-07 Time:11:02:25.
 ************************************************************************/
#include<stdio.h>
#include<sys/msg.h>
#include<sys/types.h>

int main()
{
        int i,msgid[2];
        msgid[0]=msgget(IPC_PRIVATE,0666|IPC_CREAT);
        printf("msgid=%d\n",msgid[0]);
        msgctl(msgid[0],IPC_RMID,NULL);
        ／／ipc_perm[0].seq++;
        //再次创建ipc1,ipc2
        msgid[0]=msgget(IPC_PRIVATE,0666|IPC_CREAT);
        msgid[1]=msgget(IPC_PRIVATE,0666|IPC_CREAT);
        printf("msgid=%d\n",msgid[0]);
        printf("msgid=%d\n",msgid[1]);
        msgctl(msgid[0],IPC_RMID,NULL);
        msgctl(msgid[1],IPC_RMID,NULL);

        return 0;
}
```
对应结果：

``` bash
bogon:unp qinchao$ ./p26_2 
msgid=65536
msgid=131072
msgid=65537
bogon:unp qinchao$  
```
##### ,参考文献
《UNIX网络编程 卷2:进程间通信》

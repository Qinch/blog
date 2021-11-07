title: xv6 study
category: xv6
date: 2019-03-17
tags: [xv6, linux]
toc: false
---

### 1. xv6 book

#### 1 chapter1: os interfaces

0, The system call `exec` replaces the calling process's memory but perserves its file table.

1, It should be clear why it is a good idea that fork and exec are separate calls. Beacuse if they are separate, the shell can fork a child, use open, close, dup in the child to change the standard input and output file descriptors and then exec.
  Two file descriptors share an offset if they were derived from the same original file descriptor by a sequence of fork and dup calls. Otherwise file descriptors do not share offsets, even if they resulted from open calls for the same file. （其实结合APUE看效果更好^-^）

2, The `unlink` system call removes a name from the file system. The file's inode and the disk space holding its content are only freed when the file's link count is zero and no file descrptors refer to it.

#### 2 os organization

0, Similarly, Unix transparently switches hardware CPUs among processes, saving and restoring register state as necessary, so that applications don't have to be aware of time sharing. This transparency allows the operating system to share CPUs even if some applications are in infinite loops. 

1, If an application in user mode attempts to execute a priviliged instruction, then the CPU doesn't execute the instruction, but switches to supervisor mode so that supervisort-mode code can terminate the application.

2, Each process has two stacks: a user stack and a kernel stack. While a process in the kernel, its user stack still contains saved data, but isn't actively used.

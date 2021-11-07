title: Makefile
category: makefile
date: 2017-4-19
tags: [makefile,make]
toc: false
comments: true
---
#### 1,makefile的规则

```bash
target ...: prerequisites ...
	cmd
	...
	...

其中，target可以是一个obj file,也可以是一个执行文件，还可以是一个label.
prerequisites：为生成该target所依赖的文件或target
cmd(必须以一个Tab键作为开头)：为target要执行的命令
注：prerequisites中如果有一个以上的文件比target文件要新的话，cmd所定义的命令就会被执行。
```

<!--more-->



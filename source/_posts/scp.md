title: scp.exp脚本:scp命令自动输入密码
category: shell
date: 2015-06-21
tags: [scp,tips]
toc: false
---

#### 1.scp.exp用法(以ubuntu为例):

- sudo apt-get install expect
- expect scp.exp 127.0.0.1 root passwd srcfile destfile 300

#### 2.scp.exp脚本内容

```bash
#!/usr/bin/expect
#目的机器的ip
set host [lindex $argv 0]
#目的机器的用户名
set username [lindex $argv 1]
#目的机器的用户密码
set password [lindex $argv 2]
#源文件或源目录
set src_file [lindex $argv 3]
#目的文件或目的目录
set dest_file [lindex $argv 4]
#expect超时时间
set time_out [lindex $argv 5]

spawn scp -r $src_file $username@$host:$dest_file
set timeout $time_out
expect {
 "(yes/no)?"
  {
    send "yes\n"
    exp_continue
  }
 "*assword:"
  {
    send "$password\n"
    exp_continue
  }
 "ermission denied"
  {
    send_user "Copy $src_file to $dest_file failed.\n"
    exit 1
  }
  eof
  {
    send_user "Copy $src_file to $dest_file succ.\n"
    exit 0
  }
}
#expect超时
exit 2
```


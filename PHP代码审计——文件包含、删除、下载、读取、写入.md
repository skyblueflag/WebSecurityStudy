# PHP代码审计——文件包含、删除、下载、读取、写入

# 梦想CMS（lmxcms）文件删除

1. 漏洞详情——CNVD-2020-59469

   ![1.png](img/PHPCode/File/1.png)

2. 漏洞描述称后台`Ba***.cl***.php`文件存在任意文件删除，查看cms源码，只有BackdbAction.class.php和BasicAction.class.php这两个类文件符合名字要求

   ![2.png](img/PHPCode/File/2.png)

3. 查看源代码，代码中的注释显示只有BackdbAction.class.php有文件删除功能。php中的文件删除函数是unlink()，搜索一下该函数，发现BackdbAction.class.php文件存在该函数，且有变量传入，那么应该就是这个文件存在文件删除漏洞了

   ![3.png](img/PHPCode/File/3.png)

   ![4.png](img/PHPCode/File/4.png)

4. 类文件最下方的delOne()方法中传入了filename的变量，并且与`file/back`进行拼接组成文件路径，也就是说只能删除`file/back`这个目录下的文件，但只要能绕过路径，就能实现任意文件删除

   ![5.png](img/PHPCode/File/5.png)

5. 可以看到文件中的delbackdb()和delmorebackdb()都调用了delOne()方法，delbackdb()中支队文件名进行了首尾两端的空格去除，而且只校验了文件名。

   ![5.png](img/PHPCode/File/6.png)

6. 使用`../../`即可绕过目录限制，在根目录下新建一个test.txt文件，尝试删除test.txt文件，如下所示，文件被成功删除

   ``` html
   http://localhost:8081/admin.php?m=backdb&a=delbackdb&filename=../../test.txt
   ```

   ![8.png](img/PHPCode/File/8.png)

7. 若网站的install目录在完成安装后没有删除，则利用该文件删除漏洞删除install目录下的install_ok.txt文件，则可实现对网站的重新覆盖安装

# 梦想CMS（lmxcms）任意文件读取与写入

## 文件读取

1. php文件读取的函数

   ``` php
   fread()
   fgets()
   fgetss()
   file()
   readfile()
   file_get_contents()
   fpassthru()
   ```

2. 漏洞详情————CNVD-2020-51412

   ![9.png](img/PHPCode/File/9.png)

3. 漏洞详情中没有给出任何提示，但是只给了低危，说明漏洞在后台，在项目中搜索`file_get_contents`

   ![10.png](img/PHPCode/File/10.png)

4. 只有file.class.php文件中的file_get_contents存在变量传入，该函数被getcon函数调用，查看getcon函数被哪些方法调用了，双击进入该文件，选择getcon右键查找使用，如下，可以看到getcon被TemplateAction.class.php文件的TemplateAction类的editfile方法调用了，并且传入了dir变量

   ![11.png](img/PHPCode/File/11.png)

5. dir变量传入editfile方法后，与`$this->config['template']`进行拼接，echo输出一下`$this->config['template'].$dir`，访问一下看拼接的是什么

   ``` html
   admin.php?m=template&a=editfile&dir=1
   ```

   ![12.png](img/PHPCode/File/12.png)

6. 如下所示，拼接的是`E:/phpstudy/phpstudy_pro/WWW/localhost/lmxcms1.4/template/`，也就是模板文件所在的目录

   ![13.png](img/PHPCode/File/13.png)

7. 尝试一下读取网站配置文件，如下所示，成功读取到配置文件

   ``` html
   http://localhost:8081/admin.php?m=Template&a=editfile&dir=../inc/db.inc.php
   ```

   ![14.png](img/PHPCode/File/14.png)

## 文件写入

1. 如上所示，成功读取到文件后，可以直接编辑读取到的文件并提交，我们可以直接在读取到的文件中写入webshell，这样就能直接上线了

2. 既然可以修改文件，说明一定还有修改文件的方法，查看源码，发现还是在editfile方法中，存在一个`file::put()`，先判断POST数据里面是否设置了settemcontent，然后再进行写入操作

   ![15.png](img/PHPCode/File/15.png)

3. 搜索fileput，方法介绍如下，有两个参数，file参数规定要写入数据的文件，文件不存在则创建新文件，data参数规定要写入的数据

   ![16.png](img/PHPCode/File/16.png)

4. 尝试写入一个webshell，可以看到成功在template目录下写入了shell.php，也可以加上`../`写入到根目录下

   ![17.png](img/PHPCode/File/17.png)

5. 使用蚁剑连接成功

   ![18.png](img/PHPCode/File/18.png)
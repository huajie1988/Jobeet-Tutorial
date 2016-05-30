# Symfony2 Jobeet Day 1: 开始项目

今天我们主要是建立开发环境，安装symfony2并使其在浏览器中展示一个demo页面。
首先你必须确保你的电脑已经具备了一个web开发环境。因此你至少需要有一个web 服务器（例如apache或者Nginx又或是其他），一个数据库（MySQL）和PHP解释器（5.3.2及以上版本）

## 下载并安装symfony2

在你的web 服务器上建立一个新的站点目录，用来安装这个新项目。我们将其命名为Jobeet。
打开 http://symfony.com/download ，选择”Symfony Standard”并下载（截至作者书写时最新版本为2.0.11）。下载完成之后将其解压到之前你所创建的目录。结构如下图： 
 
![目录结构](http://www.ens.ro/wp-content/uploads/2012/03/jobeet_screen_001.png)

> 译者注：上面提供的方法已经失效，自symfony2.3之后下载通过Symfony Installer进行安装。具体方法可参考官网。或由Composer进行安装（~~其中1.0.0-alpha10有较大bug，会导致无法安装包，请谨慎下载~~ 截至至现在该问题已经恢复正常），由于国内网络众所周知的原因，基本上两者直接下载可能性都较小，前者可选定网络流量较少的时段下载。


## 配置web 服务器

一个好的网站应该使浏览器只能访问web根目录文件夹，其中包括样式表，js和图片。基于此我们将修改一下apache的配置
打开http.conf并填写如下配置：
```apache
<VirtualHost *:80>
    ServerName jobeet.local
    DocumentRoot /home/dragos/work/jobeet/web
    DirectoryIndex app.php
    ErrorLog /var/log/apache2/jobeet-error.log
    CustomLog /var/log/apache2/jobeet-access.log combined
    <Directory "/home/dragos/work/jobeet/web">
        AllowOverride All
        Allow from All
    </Directory>
</VirtualHost>
```

更改完上述配置之后请重启服务器。
这是apache定义虚拟主机的标准方式，根据你的服务器配置及apache的版本不同，配置上可能会略有差异。比如在Ubuntu，你必须在/etc/apache2/sites-enabled/文件夹下创建一个与上述内容相同的文件并取名为jobeet。

最后声明一个本地域名 **jobeet.local**。
如果你是Linux用户，请修改/etc/hosts文件；
如果你是windows用户，打开C:\WINDOWS\system32\drivers\etc\ directory\hosts，添加如下行

> 127.0.0.1 jobeet.local

## 测试Symfony2安装

重启apache，然后在浏览器中输入 *http://jobeet.local/app_dev.php* 。你将会看到如下页面：

![运行效果](http://www.ens.ro/wp-content/uploads/2012/03/jobeet_screen_002.png) 


为了避免后期开发中遇到的各种环境上的问题，请打开 *http://jobeet.local/config.php* 检查您的配置是否达到要求。并确保没有什么大问题列出，同时遵循列出的建议进行修改（如果有的话）。


## Symfony2 命令行
就像Symfony 1.x一样，Symfony2也自带了命令行工具帮助你创建不同的项目。想要查看命令列表可以使用如下指令：

> **php app/console list**

创建一个Bundle
正如你所知道的那样，一个Symfony2的项目是由一个个的Bundle组成的，而且Symfony框架本身就是一个Bundle。为我们的应用程序创建一个新的Bundle可以使用如下命令：

> **php app/console generate:bundle --namespace=Ens/JobeetBundle --format=yml**

生成器会在过程中问你几个问题，这里有问题及相应的答案供你参考（除了一个使用了非默认回答）

> Bundle namespace [Ens/JobeetBundle]: Ens/JobeetBundle
Bundle name [EnsJobeetBundle]: EnsJobeetBundle
Target directory [/home/dragos/work/jobeet/src]:
/home/dragos/work/jobeet/src
Configuration format (yml, xml, php, or annotation) [yml]: yml
Do you want to generate the whole directory structure [no]? yes
Do you confirm generation [yes]? Yes
Confirm automatic update of your Kernel [yes]? Yes
Confirm automatic update of the Routing [yes]? Yes

接下来清理缓存


> **php app/console cache:clear --env=prod
php app/console cache:clear --env=dev**


我们能在项目目录下看到`src/Ens/JobeetBundle`这个新的Bundle。
这个Bundle生成器帮助我们生成了一个DefaultController控制器和一个index action。现在你能通过浏览器访问 *http://jobeet.local/hello/jobeet* 或 *http://jobeet.local/app_dev.php/hello/jobeet* 这两个网址来查看效果。

## 环境
与Symfony 1.x相比，Symfony2也有它自己不同的环境。如果你打开这个项目的web目录，将看到`app.php` 和`app_dev.php`这两个php文件。他们称之为前端控制器。应用程序的所有请求都将通过他们。`app.php`文件是用于生产环境，而`app_dev.php`则被用来在web开发人员进行工作时使用。开发环境被证明是非常方便的，这是因为它会告诉你所有的错误和警告与`Web调试工具栏`---开发人员最好的朋友。
这就是今天所有的一切了。明天我们将讨论关于网站所需要实现的功能，希望到时不见不散。

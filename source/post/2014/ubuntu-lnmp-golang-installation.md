```toml

title = "Ubuntu 下 nginx , php , mysql 和 golang 的简单安装"
slug = "ubuntu-lnmp-golang-installation"
date = "2014-02-19 21:56:39"
author = "fuxiaohei"
tags = ["LNMP","Go"]

```

我是搞php出身，自然安装lnmp是常规技能。以前的手段还是lnmp安装包，比如军哥的[lnmp1.0](http://lnmp.org/)。随着php和mysql的更新，大多数一键安装都开始版本老化，更新困难的问题。因此，重新研究了一下Ubuntu下lnmp的安装，发现现在简单的多，记录一下。

另外最近在学习golang，Ubuntu下安装自然也是必须的过程。不过golang的安装也有一些奥妙。当然，不是源码安装的啦。<!--more-->

#### Nginx Stable/Development

Ubuntu下的包管理器是`apt-get`或者说`dpkg`。常规的安装命令`apt-get install`(注意权限`sudo apt-get install`)。Nginx是这几个软件里最友好的，直接可以添加stable源：

	add-apt-repository ppa:nginx/stable
    
或者development源：

	add-apt-repository ppa:nginx/development
    
如果没有安装命令`add-apt-repository`,安装：

	apt-get install python-software-properties 
    
之后常规的操作：

	apt-get update
    apt-get install nginx
    service nginx start
    
#### PHP 5.4+

PHP的ppa源有个老兄专门在做，[Ondrej Sury](https://launchpad.net/~ondrej/)。有[php5.4](https://launchpad.net/~ondrej/+archive/php5-oldstable)，[php5.5](https://launchpad.net/~ondrej/+archive/php5)和[php5.6](https://launchpad.net/~ondrej/+archive/php5-5.6)的源，具体的可以看官方页面。
为什么没有5.3？你落伍啦！5.4+性能提高很多，5.5还有内置的`ZendOpCache`。安装php5.5:

	add-apt-repository ppa:ondrej/php5
    apt-get update
    apt-get install php5 php5-fpm
    service php5-fpm start

还有些必要的包，安装一下，记得重启php5-fpm：

	apt-get install php5-gd php5-curl php5-sqlite php5-mysqlnd php5-mcrypt
    service php5-fpm restart
    
至于nginx怎么配置php-fpm，一搜一大把，不多说。

#### MySQL 5.5+ & MariaDB

还是这个老兄，维护着[mysql5.5](https://launchpad.net/~ondrej/+archive/mysql-5.5), [mysql5.6](https://launchpad.net/~ondrej/+archive/mysql-5.6) 和 [MariaDB5.5](https://launchpad.net/~ondrej/+archive/mariadb-5.5)。所以，很简单，比如安装MariaDB(不喜欢mysql，被oracle摧残了)：

	add-apt-repository ppa:ondrej/mariadb-5.5
    apt-get update
    apt-get install mariadb-server-5.5
    service mysql start
    
这里注意，安装会提示`InnoDB Plugin Disabled`。不要紧，MariaDB把InnoDB内置进去了，其实是已经启动的。具体的可以：

	mysql SHOW ENGINE INNODB STATUS;
    
#### Golang

重头戏是golang啦。我搜寻了半天ppa源，只找到一个可以安装golang1.1.1的源，很不爽。其实可以golang官方下载已经编译好的[linux.tar.gz](https://code.google.com/p/go/downloads/list)。但是需要自己手动设置`GOROOT`，有点麻烦啊。

终于还是发现了个好工具[Godeb](http://blog.labix.org/2013/06/15/in-flight-deb-packages-of-go)。实际上这就是一个deb包构建器。先把官方编译好的tar.gz下载，打包成deb然后执行安装。

以64位安装为例：

```
wget https://godeb.s3.amazonaws.com/godeb-amd64.tar.gz
tar -zxvf godeb-amd64.tar.gz
./godeb install
 ```
    
就开始安装最新版本。还可查看支持的版本，并安装特定版本：

	./godeb list
    1.2
    1.2rc5
    1.2rc4
    1.2rc3
    1.2rc2
    1.2rc1
    1.1.2
    1.1.1
    1.1
    (...)
    
    ./godeb install 1.1
    
安装好后，可以用`go env`查看，是否安装完成。

剩下的设置`GOPATH`,`GOBIN`就不赘述了。我是修改在`/etc/profile`里面的。

#### 写在最后

Ubuntu下很多东西都有源，容易安装，也是好事啊。
    

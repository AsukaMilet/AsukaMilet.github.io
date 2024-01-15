+++
title = "在WSL2中安装MySQL"
author = "AsukaMilet"
date = 2023-04-09
tags = ["WSL", "Database"]

description = "一次关于WSL2安装MySQL跳坑与爬出坑的记录"

+++

写这篇文章的目的是自己已经在WSL中安装了*SQLite*、*PostgreSQL*，都没有遇到什么问题，然后准备安装MySQL时也以为会轻轻松松，开开心心的启动`mysql-server`，接着就遇到了令人抓狂的error，希望通过这篇文章记录下自己是如何不知不觉跳进坑中，又如何奋力从坑中爬出来的😋

## Environment

- WSL2
- Ubuntu 22.04(jammy)

## 入坑

### 安装MySQL APT repository

根据MySQL的[官方文档](https://dev.mysql.com/doc/mysql-apt-repo-quick-guide/en/)，我们要做的就是下载MySQL APT repository，然后安装MySQL APT repository:

```shell
$ sudo dpkg -i mysql-apt-config_w.x.y-z_all.deb
```

然后再在安装过程期间选择所需要的MySQL版本和组件，在选择完一切之后，我们只要`ok`就可以了。

然后我们再从MySQL APT repository更新package information：

```shell
$ sudo apt-get update
```

### 安装MySQL Server

在更新完package information之后，按照惯例，我们只需要运行如下命令：

```shell
$ sudo apt-get install mysql-server
```

等待系统完成这一切，我们就可以开开心心地使用MySQL了，一切都如往常一样，今天又是赞美WSL的一天。

安装完成之后，根据Ubuntu的[官方文档](https://ubuntu.com/server/docs/databases-mysql)，`mysql-server`应该会自动启动，我们可以使用下面的命令来查看状态：

```shell
$ sudo service mysql status
```

如果不出差错，这个时候就会出现差错了，一般这个差错长这样：

```shell
$ sudo service mysql start
mysql: unrecognized service

$ sudo service mysqld start
mysqld: unrecognized service

$ sudo service mysql-server start
mysql-server: unrecognized service
```

头疼啊，这个问题折磨了我整整一天，也不知道是问题太冷门还是怎么的，使用Google/Stackoverflow大部分的解决方案也没起到作用，在最后，我找到了一个CSDN的文章(可惜这篇文章找不到了)，作者也遇到了和我一样的问题。当然，这篇文章也没有解决我的问题，但是他给出了Github的一个issue，最终通过这个issue我终于从坑里跳了出来。

## 出坑

在这里我首先要给出对我帮助最大的两个网页：

- [Why is the mysql-server package from mysql.com missing server files?](https://askubuntu.com/questions/1160642/why-is-the-mysql-server-package-from-mysql-com-missing-server-files)
- [Linux Native AIO interface is not supported on this platform.](https://github.com/Microsoft/WSL/issues/3631)

这两个网页均提到在WSL2直接安装MySQL8.x无法直接运行，需要先退版本至MySQL 5.x再升级至MySQL 8.x(我也不知道为什么)

### 安装MySQL 5.x

这一部分不再赘述，直接上面给出的两个网页已经给出了粗略的步骤，在这个过程中我也遇到了一些小问题，最后通过万能的Google找到了这两篇文章，里面有降级的详细过程：

- [How to Install MySQL 5.7 on Ubuntu 22.04 LTS](https://www.fosstechnix.com/how-to-install-mysql-5-7-on-ubuntu-22-04-lts/)

- [How to install MySQL 5.7 or 8 on Ubuntu 18.04, 20.04, 22.04](https://www.devart.com/dbforge/mysql/how-to-install-mysql-on-ubuntu/)

### 重新升级至MySQL 8.x

在重新升级的过程中，出坑中的两篇文章都是在不卸载原MySQL的情况下直接进行升级，但是我在使用`sudo apt-get install`的过程中总是会冲突错误(原谅我没有截图，我不想再复现这整个bug了)

经过又一阵摸索之后，我发现解决办法如下：

1. 重新下载[*MySQL apt-config*包](https://dev.mysql.com/downloads/repo/apt/)，然后彻底[卸载MySQL 5.x](https://www.baeldung.com/linux/mysql-complete-clean-reinstall)

2. 安装下载好的*MySQL apt-config*包，最好再运行一次:

   ```shell
   $ sudo dpkg-reconfigure mysql-apt-config
   ```

3. 最后安装MySQL 8.x，并修改文件`/etc/init.d/mysql`，找到：

   ```shell
   "./usr/share/mysql/mysql-helpers"
   # 将其修改为
   "./usr/share/mysql-8.0 /mysql-helpers"
   ```

如果正确执行了上述过程，此时`MySQL server`应该就能正确启动了，结果如下：

![屏幕截图 2023-04-09 190136](https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-04-09%20190136.png)

## Reference

- [A Quick Guide to Using the MySQL APT Repository](https://dev.mysql.com/doc/mysql-apt-repo-quick-guide/en/)

- [Linux Native AIO interface is not supported on this platform.](https://github.com/Microsoft/WSL/issues/3631)

- [Why is the mysql-server package from mysql.com missing server files?](https://askubuntu.com/questions/1160642/why-is-the-mysql-server-package-from-mysql-com-missing-server-files)

- [How to Install MySQL 5.7 on Ubuntu 22.04 LTS](https://www.fosstechnix.com/how-to-install-mysql-5-7-on-ubuntu-22-04-lts/)

- [How to install MySQL 5.7 or 8 on Ubuntu 18.04, 20.04, 22.04](https://www.devart.com/dbforge/mysql/how-to-install-mysql-on-ubuntu/)

- [Uninstall and Reinstall MySQL on Ubuntu : A Step-by-Step Guide](https://linuxscriptshub.com/uninstall-reinstall-mysql-ubuntu/)

- [How to Do a Complete Clean Reinstall of MySQL on Linux](https://www.baeldung.com/linux/mysql-complete-clean-reinstall)

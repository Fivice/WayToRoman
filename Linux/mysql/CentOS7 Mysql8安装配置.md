# CentOS7 Mysql8安装配置

## 1. 环境检查及准备

```bash
#检查是否安装MySQL 或 MariaDB：
rpm -qa | grep -E 'mysql|mariadb'
#如果存在，则将其卸载掉：
rpm -e --nodeps 查询到的rpm
#创建mysql安装目录与数据存放目录:
mkdir /usr/local/mysql
mkdir /usr/local/mysql/data
#给安装和存放目录赋权：
sudo chmod -R 777 /usr/local/mysql
sudo chmod -R 777 /usr/local/mysql/data
#创建mysql组：
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
#说明：创建MySQL用户但该用户不能登陆(-s /bin/false参数指定mysql用户仅拥有所有权，而没有登录权限)
```

## 2.获取安装包

通过国内镜像ftp服务器下载指定版本的mysql,地址如下：
[http://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-8.0/mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz](http://mirrors.ustc.edu.cn/mysql-ftp/Downloads/MySQL-8.0/mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz)

## 3.安装准备

通过xftp工具软件将下载后文件上传到指定服务器，后通过指令解压

```bash
tar -xvJf mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
#或者如下操作
xz -d mysql-8.0.20-linux-glibc2.12-x86_64.tar.xz
tar -xvf mysql-8.0.20-linux-glibc2.12-x86_64.tar
```

解压后

``` bash
#进入解压后目录
cd mysql-8.0.20-linux-glibc2.12-x86_64
#将文件家中所有文件拷贝到指定文件夹
cp -rf * /usr/local/mysql/
#创建mysql配置文件
sudo vi /etc/my.cnf
#修改配置文件的拥有用户
chown mysql:mysql my.cnf
#这里之所以修改，是因为服务器权限的原因，'用户名'是我使用的用户，为防止sudo权限被回收，先修改配置文件的拥有者。
```

添加如下内容，然后保存并退出：

```ini
[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8
[mysqld]
#用于忽略数据库名称大小写比较
lower_case_table_names=1
#设置3306端口
port=3306
# 设置mysql的安装目录
basedir=/usr/local/mysql
# 设置mysql数据库的数据的存放目录
datadir=/usr/local/mysql/data

# 允许最大连接数
max_connections=200
# 服务端使用的字符集默认为8比特编码的latin1字符集
character-set-server=utf8

# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB

max_allowed_packet=16M

[client]
#设置mysql客户端连接服务器默认使用端口
port=3306
# 设置mysql客户端默认字符集
efault-character-set=utf8
```

## 4.正式安装

```bash
cd /usr/local/mysql/bin
./mysqld --initialize --console --user=mysql
#指令结束后会打印出一个比较乱的密码要记住，后面登录mysql修改密码用
```

## 5.启动服务

```bash
cd /usr/local/mysql/support-files
sudo ./mysql.server start
```

## 6.系统配置

```bash
#mysql加入系统进程中：
sudo cp mysql.server /etc/init.d/mysqld
#重启MySQL服务：
sudo service mysqld restart
#建立软连接，在系统中增加mysql指令：
sudo ln -s /usr/local/mysql/bin/mysql /usr/bin

# 查看firewall启动情况
systemctl status firewalld
# 开启3306端口
firewall-cmd --zone=public --add-port=3306/tcp --permanent
# firewall-cmd --reload 重启
firewalld
#查看3306端口是否开启
firewall-cmd --query-port=3306/tcp
```

## 7.mysql数据库初始设置

```bash
#修改随机登陆密码：
mysql -uroot -p
#输入上面记录的随机密码。
alter user 'root'@'localhost' identified by '你的新密码';
#必须先重置密码，否则会一直报错，提示：
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.

#设置允许远程登陆：在已登录mysql数据库的前提下:
use mysql;
update user set user.Host='%' where user.User='root';
flush privileges;
quit
```

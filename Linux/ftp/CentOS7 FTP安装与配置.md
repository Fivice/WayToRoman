# CentOS7 FTP安装与配置

## 1、FTP的安装

``` bash
#安装
yum install -y vsftpd

#设置开机启动
systemctl enable vsftpd.service

#启动
systemctl start vsftpd.service

#停止
systemctl stop vsftpd.service

#查看状态
systemctl status vsftpd.service
```

## 2、配置FTP

```bash
#打开配置文件
vim /etc/vsftpd/vsftpd.conf

#显示行号
:set number

#修改配置 12 行
anonymous_enable=NO

#修改配置 33 行
anon_mkdir_write_enable=YES

#修改配置48行
chown_uploads=YES

#修改配置72行
async_abor_enable=YES

#修改配置82行
ascii_upload_enable=YES

#修改配置83行
ascii_download_enable=YES

#修改配置86行
ftpd_banner=Welcome to blah FTP service.#修改配置100行chroot_local_user=YES

#添加下列内容到vsftpd.conf末尾
use_localtime=YES
listen_port=21
idle_session_timeout=300
guest_enable=YES
guest_username=vsftpd
user_config_dir=/etc/vsftpd/vconf
data_connection_timeout=1
virtual_use_local_privs=YES
pasv_min_port=40000
pasv_max_port=40010
accept_timeout=5
connect_timeout=1allow_writeable_chroot=YES
```

## 3、建立用户文件

```bash
#创建编辑用户文件
vim /etc/vsftpd/virtusers
#第一行为用户名，第二行为密码。不能使用root作为用户名
bhl
12345
```

## 4、生成用户数据文件

```bash
db_load -T -t hash -f /etc/vsftpd/virtusers /etc/vsftpd/virtusers.db

#设定PAM验证文件，并指定对虚拟用户数据库文件进行读取

chmod 600 /etc/vsftpd/virtusers.db
```

## 5、修改 /etc/pam.d/vsftpd 文件

```bash
# 修改前先备份

cp /etc/pam.d/vsftpd /etc/pam.d/vsftpd.bak

vi /etc/pam.d/vsftpd
#先将配置文件中原有的 auth 及 account 的所有配置行均注释掉
auth sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/virtusers
account sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/virtusers

# 如果系统为32位，上面改为lib
```

### 6、新建系统用户vsftpd，用户目录为/home/vsftpd

```bash
#用户登录终端设为/bin/false(即：使之不能登录系统)
useradd vsftpd -d /home/vsftpd -s /bin/false
chown -R vsftpd:vsftpd /home/vsftpd
```

## 7、建立虚拟用户个人配置文件

```bash
mkdir /etc/vsftpd/vconf
cd /etc/vsftpd/vconf

#这里建立虚拟用户bhl配置文件
touch bhl
#编辑leo用户配置文件，内容如下，其他用户类似
vi bhl

local_root=/home/vsftpd/bhl/
write_enable=YES
anon_world_readable_only=NO
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
#建立leo用户根目录
mkdir -p /home/vsftpd/bhl/
```

## 8、防火墙设置

```bash
#IPtables 的设置方式：
vi /etc/sysconfig/iptables
#编辑iptables文件，添加如下内容，开启21端口
-A INPUT -m state --state NEW -m tcp -p tcp --dport 21 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 40000:40010 -j ACCEPT
#firewall 的设置方式：
firewall-cmd --zone=public --add-service=ftp --permanentfirewall-cmd --zone=public --add-port=21/tcp --permanent
firewall-cmd --zone=public --add-port=40000-40010/tcp --permanent
```

## 9、重启vsftpd服务器

```bash
systemctl restart vsftpd.service
```

## 10、使用ftp工具连接测试

这个时候，使用ftp的工具连接时，我们发现是可以连接的。传输文件的时候，会发现文件上传和下载都会出现
500、503 、200等问题。这个时候，可以进行以下操作：

### 方式一、关闭SELINUX

```bash
#打开SELINUX配置文件
vim /etc/selinux/config
#修改配置参数
#注释  
SELINUX=enforcing
#增加  
SELINUX=disabled
#修改完成后，需要重启！
```

### 方式二、修改SELINUX

```bash
setenforce 0 #暂时让SELinux进入Permissive模式

#列出与ftp相关的设置
getsebool -a|grep ftp

#以下是显示出来的权限，off是关闭权限，on是打开权限。不同的机器显示的可能不一样。我看了我的显示的，和网上其他教程就不太一样
ftp_home_dir --> off
ftpd_anon_write --> off
ftpd_connect_all_unreserved --> off
ftpd_connect_db --> off
ftpd_full_access --> off
ftpd_use_cifs --> off
ftpd_use_fusefs --> off
ftpd_use_nfs --> off
ftpd_use_passive_mode --> off
httpd_can_connect_ftp --> off
httpd_enable_ftp_server --> off
sftpd_anon_write --> off
sftpd_enable_homedirs --> off
sftpd_full_access --> off
sftpd_write_ssh_home --> off
tftp_anon_write --> off
tftp_home_dir --> off

#将包含有 ftp_home_dir 和 ftpd_full_access 相关的都设置为 1

setsebool -P ftp_home_dir 1setsebool -P allow_ftpd_anon_write 1
setsebool -P ftp_home_dir 1

setenforce 1 #进入Enforcing模式
```

### 方式三、 SELINUX不对vsftp不做任何限制

```bash
setsebool -P ftpd_connect_all_unreserved 1
```

### 如果连接后无法访问文件夹，给用户文件夹授权

```bash
chmod -R 777 /home/vsftpd/bhl
```

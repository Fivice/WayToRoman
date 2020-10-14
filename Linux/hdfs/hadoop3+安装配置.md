# Hadoop安装配置  

## 一、服务器准备  

| 主机名 | IP | NameNode | DataNode | Yarn | Zookeeper | JournalNode|
| -- | -- | -- | -- | -- | -- | -- |
| hadoop01 | 172.30.64.72 | √ |  |  |  √ |  √ |
| hadoop02 | 172.30.64.78 | √ | √  |  |  √ |  √ |
| hadoop03 | 172.30.64.67 |  | √ |  √ |  √ |  √ |
| hadoop04 | 172.30.64.66 |  | √ |  |  √ |   |

## 二、Hadoop集群环境的准备

### 1.添加hadoop用户身份

以root身份登录每台虚拟机服务器，在每台服务器上执行如下操作。

```bash
#添加用户组
groupadd hadoop
#添加用户
useradd -r -g hadoop hadoop
#设置用户密码
passwd hadoop
Changing password for user hadoop.
New password: 新密码
Retype new password: 确认新密码
#出现如下提示设置成功
passwd: all authentication tokens updated successfully.
#修改文件夹权限
chown -R hadoop.hadoop /usr/local/
chown -R hadoop.hadoop /tmp/
chown -R hadoop.hadoop /home/
#给新用户所有权限
vi /etc/sudoers
#找到
root    ALL=(ALL)       ALL
#下面添加
hadoop    ALL=(ALL)       ALL
```

### 2.关闭防火墙

因为服务器不对外网开放端口，所以这里可以直接将服务器防火墙关了。
以下以CentOS7服务器为例

```bash
#查看防火墙状态
systemctl status firewalld
#如果防火墙是开启的，可以通过以下指令查看已开放端口
firewall-cmd --list-ports
#关闭防火墙
systemctl stop firewalld
#关闭防火墙开机启动
systemctl disable firewalld
#查看防火墙状态
systemctl status firewalld
```

### 3.设置静态IP

如果服务器不是固定IP需要设置静态IP

```bash
#这里的ifcfg-eth0后面的eth0为需要设置静态IP的网卡名
vi /etc/sysconfig/network-scripts/ifcfg-eth0
#添加如下描述
BOOTPROTO=static
#后面IP修改为自己想修改成的IP
IPADDR=192.168.175.201
#网络掩码（子网掩码）
NETMASK=255.255.255.0
#广播地址，一般为IP前三段+255
BROADCAST=192.168.175.255
#网关地址
GATEWAY=192.168.175.2
#DNS服务器地址
DNS1=114.114.114.114
DNS2=8.8.8.8

#设置完静态IP后需要重启网络
service network restart
```

其他服务器也参考上面设置，IP改为不同IP

### 4.设置主机名

设置每台服务器的主机名

```bash
vi /etc/sysconfig/network

NETWORKING=yes
#这里后面的名字为主机名，可修改为自己想修改成的名字
HOSTNAME=hadoop01

vi /etc/hostname
#在文件中写入本机主机名
hadoop01
```

其他主机以此类推

### 5.设置主机名与IP地址的映射关系

在每台服务器上修改“/etc/hosts”文件，添加如下配置：

```bash
#服务器01
172.30.64.72     hadoop01
#服务器01
xxx.xxx.xxx.xxx     hadoop02
#以此类推添加
···
```

### 6.集群环境下配置SSH免密码登录

注意：配置SSH免密码登录，使用hadoop身份登录虚拟机服务器，进行相关的操作。

```bash
su hadoop
#此时进入新用户出现'bash-4.2$:'则可以通过以下操作进入正常用户身份
#原因是在用useradd添加普通用户时，有时会丢失家目录下的环境变量文件，丢失文件如下：
#.bash_profile
#.bashrc
cp /etc/skel/.bashrc  /home/hadoop/
cp /etc/skel/.bash_profile  /home/hadoop/
#其他服务器如果也出现上面问题也可以通过此操作
```

（1）生成SSH免密码登录公钥和私钥

在每台虚拟机服务器上执行如下命令，在每台服务器上分别生成SSH免密码登录的公钥和私钥。

```bash
ssh-keygen -t rsa
cat /home/hadoop/.ssh/id_rsa.pub >> /home/hadoop/.ssh/authorized_keys
```

（2）设置目录和文件权限

在每台虚拟机服务器上执行如下命令，设置相应目录和文件的权限。

```bash
chmod 700 /home/hadoop/
chmod 700 /home/hadoop/.ssh
chmod 644 /home/hadoop/.ssh/authorized_keys
chmod 600 /home/hadoop/.ssh/id_rsa
```

（3）将公钥拷贝到每台服务器

在每台虚拟机服务器上执行如下命令，将生成的公钥拷贝到每台服务器上

```bash
#例如这里将hadoop01主机的公钥拷贝到hadoop02服务器上
#中途会要求输入目标服务器用户的登录密码
[hadoop@bhl home]$ ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub -p 10022 hadoop02
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/hadoop/.ssh/id_rsa.pub"
The authenticity of host '[hadoop02]:10022 ([172.30.64.78]:10022)' can't be established.
ECDSA key fingerprint is SHA256:5lMJkaIxnclLOSzoPzvH2dZhqe8vZhhh6SPiXlbvzWc.
ECDSA key fingerprint is MD5:95:6c:8d:9a:c6:48:bd:92:c3:81:a7:4d:de:a7:00:19.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
hadoop@hadoop02's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh -p '10022' 'hadoop02'"
and check to make sure that only the key(s) you wanted were added.
```

## 三、JDK安装配置

参考：[jdk安装配置](../jdk/jdk安装配置.md)

## 四、搭建并配置Zookeeper集群

### 1.下载Zookeeper

在hadoop01上执行

```bash
wget https://downloads.apache.org/zookeeper/zookeeper-3.6.1/apache-zookeeper-3.6.1-bin.tar.gz
```

### 2.安装并配置Zookeeper系统环境变量

我们将Zookeeper安装在 `/usr/local/zookeeper/` 目录下,即ZOOKEEPER_HOME安装目录为 `/usr/local/zookeeper/apache-zookeeper-3.6.1-bin` 。

结合配置JDK后，文件“/etc/profile”文件中添加的内容如下：

```profile
export JAVA_HOME=/usr/local/java/jdk1.8.0_261
export JRE_HOME=/usr/local/java/jdk1.8.0_261/jre/
export ZOOKEEPER_HOME=/usr/local/zookeeper/apache-zookeeper-3.6.1-bin

export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin:$ZOOKEEPER_HOME/bin
```

### 3.配置Zookeeper

首先，需要将“$ZOOKEEPER_HOME/conf”（“$ZOOKEEPER_HOME”为Zookeeper的安装目录）目录下的zoo_sample.cfg文件修改为zoo.cfg文件。具体命令如下

```bash
cd /usr/local/zookeeper/apache-zookeeper-3.6.1-bin/conf/
mv zoo_sample.cfg zoo.cfg
```

接下来修改zoo.cfg文件，修改后的具体内容如下：(注意本服务棋对应的hostname用 `0.0.0.0` 代替)

```ini
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/usr/local/zookeeper/apache-zookeeper-3.6.1-bin/data
dataLogDir=/usr/local/zookeeper/apache-zookeeper-3.6.1-bin/dataLog
clientPort=2181
server.1=0.0.0.0:2888:3888
server.2=hadoop02:2888:3888
server.3=hadoop03:2888:3888
server.3=hadoop04:2888:3888
```

在Zookeeper的安装目录下创建“data”和“dataLog”两个文件夹。

```bash
mkdir -p /usr/local/zookeeper/apache-zookeeper-3.6.1-bin/data
mkdir -p /usr/local/zookeeper/apache-zookeeper-3.6.1-bin/dataLog
#切换到新建的data目录下，创建myid文件，具体内容为数字“1”，如下所示：
echo "1" >> /usr/local/zookeeper-3.5.5/data/myid
```

### 4.复制Zookeeper和系统环境变量到其他服务器

将 `hadoop01` 上安装的Zookeeper和系统环境变量文件拷贝到 `hadoop02` 、 `hadoop03` 和 `hadoop04` 服务器，具体操作如下

```bash
scp -r /usr/local/zookeeper/apache-zookeeper-3.6.1-bin/ hadoop02:/usr/local/
scp -r /usr/local/zookeeper/apache-zookeeper-3.6.1-bin/ hadoop03:/usr/local/
scp -r /usr/local/zookeeper/apache-zookeeper-3.6.1-bin/ hadoop04:/usr/local/
sudo scp /etc/profile hadoop02:/etc
sudo scp /etc/profile hadoop03:/etc
sudo scp /etc/profile hadoop04:/etc
```

### 5.修改myid文件内容

将 `hadoop02` 服务器上Zookeeper的myid文件内容修改为数字2。
将 `hadoop03` 服务器上Zookeeper的myid文件内容修改为数字3。
将 `hadoop04` 服务器上Zookeeper的myid文件内容修改为数字4。

### 6.使环境变量生效

在所有服务器上执行以下命令

```bash
source /etc/profile
```

## 五、搭建并配置Hadoop集群

注意：1-5步是在 `hadoop01` 服务器上执行的操作。

### 1.下载Hadoop

在 `hadoop01` 上执行如下命令下载Hadoop。

```bash
wget mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.2.0/hadoop-3.2.0.tar.gz
```

### 2.解压并配置系统环境变量

（1）解压Hadoop

输入如下命令对Hadoop进行解压。

```bash
tar -zxvf hadoop-3.2.0.tar.gz
```

（2）配置Hadoop系统环境变量

同样，Hadoop的系统环境变量也需要在 `/etc/profile` 文件中进行相应的配置，通过如下命令打开 `/etc/profile` 文件并进行相关设置。

```bash
sudo vim /etc/profile
```

上述命令可能要求输入密码，根据提示输入密码即可。

在 `/etc/profile` 文件中添加如下配置：

```profile
HADOOP_HOME=/usr/local/hadoop-3.2.0
PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
export HADOOP_HOME PATH
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
```

（3）使系统环境变量生效

```bash
source /etc/profile
```

（4）验证Hadoop系统环境变量是否配置成功

具体验证方式如下所示：

```bash
hadoop version
Hadoop 3.2.0
Source code repository https://github.com/apache/hadoop.git -r e97acb3bd8f3befd27418996fa5d4b50bf2e17bf
Compiled by sunilg on 2019-01-08T06:08Z
Compiled with protoc 2.5.0
From source with checksum d3f0795ed0d9dc378e2c785d3668f39
This command was run using /usr/local/hadoop-3.2.0/share/hadoop/common/hadoop-common-3.2.0.jar
```

### 3.修改Hadoop配置文件

Hadoop集群环境的搭建流程基本和Zookeeper集群的搭建流程相同，除了要解压安装包和配置系统环境变量外，还需要对自身框架进行相关的配置。

（1）配置 `hadoop-env.sh`

在hadoop-env.sh文件中，需要指定JAVA_HOME的安装目录，具体如下：

```bash

cd /usr/local/hadoop-3.2.0/etc/hadoop/
vim hadoop-env.sh
export JAVA_HOME=/usr/local/java/jdk1.8.0_261
```

（2）配置 `core-site.xml`

具体配置信息如下：

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://ns/</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/hadoop-3.2.0/tmp</value>
    </property>
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>hadoop01:2181,hadoop02:2181,hadoop03:2181,hadoop04:2181</value>
    </property>
</configuration>
```

（3）配置hdfs-site.xml

具体配置信息如下：

```xml
<configuration>
    <property>
        <name>dfs.nameservices</name>
    <value>ns</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.ns</name>
        <value>nn1,nn2</value>
    </property>
    <!--主节点-->
    <property>
        <name>dfs.namenode.rpc-address.ns.nn1</name>
        <value>hadoop01:9000</value>
    </property>
    <property>
        <name>dfs.namenode.http-address.ns.nn1</name>
        <value>hadoop01:9870</value>
    </property>
    <!--从节点-->
    <property>
        <name>dfs.namenode.rpc-address.ns.nn2</name>
        <value>hadoop02:9000</value>
    </property>
    <property>
        <name>dfs.namenode.http-address.ns.nn2</name>
        <value>hadoop02:9870</value>
    </property>
    <!--journalnode-->
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://hadoop01:8485;hadoop02:8485;hadoop03:8485/ns</value>
    </property>
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/usr/local/hadoop-3.2.0/journaldata</value>
    </property>
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>dfs.client.failover.proxy.provider.ns</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <property>
        <name>dfs.ha.fencing.methods</name>
        <value>
        sshfence
            shell(/bin/true)
        </value>
    </property>
    <property>
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/home/hadoop/.ssh/id_rsa</value>
    </property>
    <property>
        <name>dfs.ha.fencing.ssh.connect-timeout</name>
        <value>30000</value>
    </property>
</configuration>
```

（4）配置mapred-site.xml

具体配置信息如下：

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
</configuration>
```

（5）配置yarn-site.xml

具体配置信息如下：

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <!--将yarn部署在hadoop03上面-->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop03</value>
    </property>
</configuration>
```

（6）修改workers文件

这个文件主要是用来存放DataNode节点用的。在Hadoop3.0之前的版本中，这个文件叫作“slaves”。

具体配置信息如下：(这里将4台服务器中的三台作为DataNode节点服务器)

```txt
hadoop02
hadoop03
hadoop04
```

### 4.将配置好的Hadoop拷贝到其他节点

将在 `hadoop01` 上安装并配置好的Hadoop复制到其他服务器上，具体操作如下：

```bash
scp -r /usr/local/hadoop-3.2.0/ hadoop02:/usr/local/
scp -r /usr/local/hadoop-3.2.0/ hadoop03:/usr/local/
scp -r /usr/local/hadoop-3.2.0/ hadoop04:/usr/local/
```

### 5.将配置好的Hadoop系统环境变量拷贝到其他节点

```bash
sudo scp /etc/profile hadoop02:/etc/
sudo scp /etc/profile hadoop03:/etc/
sudo scp /etc/profile hadoop04:/etc/
```

### 6.使系统环境变量生效

在所有服务器上执行如下命令，使系统环境变量生效，并验证Hadoop系统环境变量是否配置成功。

```bash
source /etc/profile
hadoop version
```

可以看到，输入“hadoop version”命令之后，命令行输出了如下信息：

```bash
Hadoop 3.2.0
Source code repository https://github.com/apache/hadoop.git -r e97acb3bd8f3befd27418996fa5d4b50bf2e17bf
Compiled by sunilg on 2019-01-08T06:08Z
Compiled with protoc 2.5.0
From source with checksum d3f0795ed0d9dc378e2c785d3668f39
This command was run using /usr/local/hadoop-3.2.0/share/hadoop/common/hadoop-common-3.2.0.jar
```

说明，Hadoop系统环境变量配置成功。

## 六、启动Zookeeper集群

在四台服务器上分别执行如下命令启动Zookeeper进程。

```bash
zkServer.sh start
```

在每台服务器上查看是否存在Zookeeper进程

```bash
#查看正在运行的Java进程
jps
#出现如下进程则表示启动成功
1476 QuorumPeerMain
1514 Jps
```

所有服务器启动完成后，可以看到每天服务器上都启动了Zookeeper进程。
查看每台服务器上Zookeeper的运行模式，具体如下所示。

```bash
zkServer.sh status
#输出结果
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper-3.5.5/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
Mode: follower
#上面的Mode还有一种情况是 Mode: leader
```

## 七、启动Hadoop集群

### 1.启动并验证journalnode进程

（1）启动journalnode进程

在 `hadoop01` 服务器上执行如下命令启动journalnode进程。

```bash
hdfs --workers --daemon start journalnode
```

注意：在Hadoop 3.0以前是输入如下命令启动journalnode进程。

```bash
hadoop-daemons.sh start journalnode
```

（2）验证journalnode进程是否启动成功

在四台服务器上分别执行 `jps` 命令查看是否存在journalnode进程，以此确认journalnode进程是否启动成功。

```bash
jps
1476 QuorumPeerMain
1669 Jps
1640 JournalNode
```

### 2.格式化HDFS

在 `hadoop01` 服务器上执行如下命令格式化HDFS。

```bash
hdfs namenode -format
```

格式化成功之后，会输出“common.Storage: Storage directory /usr/local/hadoop-3.2.0/tmp/dfs/name has been successfully formatted.”信息，并在HADOOP_HOME（/usr/local/hadoop-3.2.0/）目录下自动创建tmp目录。

### 3.格式化ZKFC

在 `hadoop01` 服务器上执行如下命令格式化ZKFC。

```bash
hdfs zkfc -formatZK
```

格式化成功之后，会输出“ha.ActiveStandbyElector: Successfully created /hadoop-ha/ns in ZK.”信息。

### 4.启动NameNode并验证

（1）启动NameNode

在 `hadoop01` 服务器上执行如下命令启动NameNode。

```bash
hdfs --daemon start namenode
```

注意：在Hadoop3.0以前的版本启动NameNode是输入如下的命令：

```bash
hadoop-daemon.sh start namenode
```

（2）验证NameNode是否启动成功

在 `hadoop01` 服务器上输入“jps”命令查看是否存在NameNode进程，以此确认NameNode是否启动成功，具体如下：

```bash
jps
1892 Jps
1476 QuorumPeerMain
1640 JournalNode
1852 NameNode
```

从输出结果可以看出，存在 `NameNode` 进程，说明NameNode启动成功。

### 5.同步元数据信息

在 `hadoop02` 服务器上执行如下命令进行元数据信息的同步操作。

```bash
hdfs namenode -bootstrapStandby
```

同步元数据信息的时候输出了“common.Storage: Storage directory /usr/local/hadoop-3.2.0/tmp/dfs/name has been successfully formatted.”信息，说明同步元数据信息成功。

### 6.启动并验证备用NameNode

（1）启动备用NameNode

在 `hadoop02` 服务器上执行如下命令启动备用NameNode。

```bash
hdfs --daemon start namenode
#注意：在Hadoop3.0以前的版本启动NameNode是输入如下的命令：
hadoop-daemon.sh start namenode
```

（2）验证备用NameNode是否启动成功

在 `hadoop02` 服务器上输入“jps”命令查看是否存在NameNode进程，以此确认备用NameNode是否启动成功，具体如下：

```bash
jps
1750 NameNode
1462 QuorumPeerMain
1816 Jps
1594 JournalNode
```

从输出结果可以看出，存在 `NameNode` 进程，说明备用NameNode启动成功。

### 7.启动并验证DataNode

（1）启动DataNode

在 `hadoop01` 服务器上执行如下命令启动DataNode。

```bash
hdfs --workers --daemon start datanode
#注意：在Hadoop3.0以前的版本启动DataNode是输入如下的命令：
hadoop-daemons.sh start datanode
```

（2）验证DataNode是否启动成功

在四台服务器分别输入 `jps` 命令，查看是否存在 `DataNode` 进程，以此确认DataNode是否启动成功。

```bash
jps
2145 DataNode
1476 QuorumPeerMain
2231 Jps
1640 JournalNode
1852 NameNode
```

四台服务器中均启动了 `DataNode` 进程，说明DataNode启动成功。

### 8.启动并验证YARN

（1）启动YARN

在 `hadoop03` 服务器上执行如下命令启动YARN。

```bash
start-yarn.sh
```

（2）验证YARN是否启动成功

在四台服务器上执行 `jps` 命令来验证YARN是否启动成功。

```bash
jps
2464 Jps
2145 DataNode
1476 QuorumPeerMain
1640 JournalNode
2329 NodeManager
1852 NameNode
```

均出现  `ResourceManager` 和 `NodeManager` 进程则表示YARN启动成功。

### 9.启动并验证ZKFC

（1）启动ZKFC

在 `hadoop01` 服务器上执行如下命令启动ZKFC。

```bash
hdfs --workers --daemon start zkfc
#注意：在Hadoop3.0以前的版本中，启动ZKFC需要使用如下命令：
hadoop-daemons.sh start zkfc
```

（2）验证ZKFC是否启动成功

在 `hadoop01` 和 `hadoop02` 服务器上分别执行 `jps` 命令，查看是否存在 `DFSZKFailoverController` 进程。

```bash
jps
2145 DataNode
1476 QuorumPeerMain
1640 JournalNode
2329 NodeManager
1852 NameNode
2734 Jps
2670 DFSZKFailoverController
```

两台服务器均启动了“DFSZKFailoverController”进程，说明ZKFC启动成功。

## 九、 测试Hadoop HA的高可用性

此时可以访问 `hadoop01` 和 `hadoop02` ip对应的地址+9870端口。其中主节点active从节点standby。

[`安装说明`](https://docs.cloudera.com/documentation/enterprise/5-14-x/topics/installation_installation.html#concept_qpf_2d2_2p)

[版本依赖](https://docs.cloudera.com/documentation/enterprise/release-notes/topics/rn_consolidated_pcm.html#)

[节点间的角色分配](https://docs.cloudera.com/documentation/enterprise/5-14-x/topics/cm_ig_host_allocations.html#concept_f43_j4y_dw)

[Package与Parcel的区别](https://docs.cloudera.com/documentation/enterprise/5-14-x/topics/cm_ig_managing_software.html)

[Parcels]( [Parcels | 5.14.x | Cloudera Documentation](https://docs.cloudera.com/documentation/enterprise/5-14-x/topics/cm_ig_parcels.html) )

[更新Cloudera版本]( [Cloudera Enterprise Upgrade Guide | (5.x and 6.x) | Cloudera Documentation](https://docs.cloudera.com/documentation/enterprise/upgrade/topics/ug_overview.html) )

## 一、概述

Cloudera版本（Cloudera’s Distribution Including Apache Hadoop，简称“CDH”），基于Web的用户界面,支持大多数Hadoop组件，包括HDFS、MapReduce、Hive、Pig、 Hbase、Zookeeper、Sqoop,简化了大数据平台的安装、使用难度。

[官方文档](https://docs.cloudera.com/documentation/enterprise/5-14-x.html)

[概览](https://docs.cloudera.com/documentation/enterprise/5-14-x/topics/installation_installation.html#install-phases)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_135023.p4hfse1yn3.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_135042.6z2f5odsndw0.webp)

## 二、安装部署

### 2.0 安装centos7

使用root用户

00.配置网卡【 /etc/sysconfig/network-scripts/ifcfg-ens18】

```shell
## 将原有配置中【BOOTPROTO=dhcp】改为【BOOTPROTO=static】,【ONBOOT=no】改为【ONBOOT=yes】；此外，再添加以下配置
# server
IPADDR=172.20.3.81
NETMASK=255.255.255.0
GATEWAY=172.20.3.254
DNS1=8.8.8.8
#METRIC=104
# hadoop-1
IPADDR=172.20.3.82
NETMASK=255.255.255.0
GATEWAY=172.20.3.254
DNS1=8.8.8.8
#METRIC=104
# hadoop-2
IPADDR=172.20.3.83
NETMASK=255.255.255.0
GATEWAY=172.20.3.254
DNS1=8.8.8.8
#METRIC=104
```

检测网络

```shell
service network restart
service network status
ping -c 3 baidu.com
```

01.安装openssh-server, openssh-client,以便远程连接【所有主机】

```shell
yum install -y openssh-server openssh-client
```

02.使用Finalshell连接

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_101147.2nfeii882so.webp)

### 2.1 基础环境配置

a.修改主机名配置hosts【所有主机】

```shell
#更改主机名，其他主机为 hadoop-1, hadoop-2,...
hostnamectl set-hostname  cm-server 
#临时关闭SELinux
setenforce 0
#查看状态
getenforce   
#永久关闭（当前不生效，下次开机生效）
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

#添加各个节点hosts解析
cat >>/etc/hosts<<EOF    
172.20.3.81    cm-server
172.20.3.82    hadoop-1
172.20.3.83    hadoop-2
EOF
#在生成密钥之前重启
reboot
#开机后检查主机名是否修改，并关闭防火墙【每次开机后都要记得关闭】
systemctl stop firewalld
```

> SELinux(Security-Enhanced Linux) 是美国国家安全局（NSA）对于强制访问控制的实现，是 Linux历史上最杰出的新安全子系统。
>
> ```shell
> sed -i 's/要被取代的字串/新的字串/g' file
> ```

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_101657.b0fkamrbe0k.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_102124.2n37rp3mk7y0.webp)

b.配置cm-server免密钥登录其他节点 【公钥加密，私钥解密】

```shell
ssh-keygen -t rsa     #在cm-server生成密钥对
for num in `seq 1 2`;do ssh-copy-id -i /root/.ssh/id_rsa.pub root@hadoop-$num;done
```

-i 指定公钥文件

c.配置各节点服务器需求 【所有主机】

```shell
#临时调整
sysctl -w vm.swappiness=10
#永久调整（当前不生效，下次开机生效）
echo "vm.swappiness=10" >>/etc/sysctl.conf
#临时禁用透明大页
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

> - 内核参数【vm.swappiness】控制换出运行时内存的【相对权重】，参数值大小对如何使用swap分区有很大联系。值越大，表示越积极使用swap分区，越小表示越积极使用物理内存。默认值swappiness=60，表示内存使用率超过100-60=40%时开始使用交换分区。swappiness=0的时候表示最大限度使用物理内存，然后才是 swap空间；swappiness＝100的时候表示积极使用swap分区，并把内存上的数据及时搬运到swap空间。
>
> - Centos7禁用THP（Transparent HugePages）：
>
>   THP的目的是通过将页表项映射更大的内存，来减少 Page Fault，从而提升 TLB （快表）的命中率
>
>   当程序的访存【局部性】较好时，THP 将带来性能提升，反之 THP 的优势不仅丧失，还有可能化身为恶魔，引起系统的不稳定。遗憾的是【数据库】的负载访问特征通常是【离散的】。
>
>   ```
>   #查看THP的启用状态
>   cat /sys/kernel/mm/transparent_hugepage/defrag
>   输出： 
>   [always] madvise never  表示THP启用了
>   always [madvise] never  表示THP禁用了
>   always madvise [never]  表示只在MADV_HUGEPAGE标志的VMA中使用THP（用户进程拥有用户空间的地址）
>   ```

### 2.2 Java环境配置

使用oracle的jdk，此处使用`jdk-8u271-linux-x64.tar.gz` 【所有主机】

```shell
cat >/etc/profile.d/java.sh<<EOF
export JAVA_HOME=/usr/java/jdk1.8.0_271
export CLASSPATH=.:\$JAVA_HOME/jre/lib/rt.jar:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar 
export PATH=\$PATH:\$JAVA_HOME/bin
EOF
source /etc/profile.d/java.sh
```

> - 关于path和classpath的含义： path变量的含义就是系统在任何路径下都可以识别java,javac命令
>
>   classpath 变量的含义是告诉jvm要使用或执行的class放在什么路径上，便于JVM加载class文件
>
> - rt.jar是JAVA基础类库，dt.jar是关于运行环境的类库，tools.jar是工具类库

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_121207.21zdacihyns0.webp)

### 2.3 安装配置数据库

在cm-server上安装mariadb，用于后期数据存储,默认没有密码

```shell
yum install mariadb*
service mariadb start
# 默认不使用密码
mysql -uroot
# 设置密码
mysqladmin -uroot password mysqladmin
```

每次开机都要执行

```shell
systemctl start mariadb
systemctl stop firewalld
```

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_120132.44s5uu3paa60.webp)

### 2.4 安装CM

a.下载解压Cloudera Manager软件包 【官网不可用，见压缩包文件】

```shell
mkdir /software && cd /software
wget -c https://archive.cloudera.com/cm5/cm/5/cloudera-manager-centos7-cm5.14.1_x86_64.tar.gz 
wget -c http://archive.cloudera.com/cdh5/parcels/5.14.2/CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel
wget -c http://archive.cloudera.com/cdh5/parcels/5.14.2/CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha1 -O CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel.sha
wget -c http://archive.cloudera.com/cdh5/parcels/5.14.2/manifest.json

wget -c https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.zip
tar -zxvf cloudera-manager-centos7-cm5.14.1_x86_64.tar.gz -C /opt/   #解压cm包
unzip mysql-connector-java-5.1.46.zip  #解压java-mysql连接jar包
chmod 777  mysql-connector-java*
cp mysql-connector-java-5.1.46/mysql-connector-java-5.1.46-bin.jar /opt/cm-5.14.1/share/cmf/lib/    #将jar包复制到指定目录下
```
> 将 mysql-connector-java-5.1.46.jar 改为 mysql-connector-java.jar 同时复制到 /opt/cm-5.14.1/share/cmf/lib/ 下， 和 /opt/cloudera/parcels/CDH-5.14.4-1.cdh5.14.4.p0.3/lib/hive/lib/

安装 hive 时需要创建数据库 hive。

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_122933.6a1knqmbbow0.webp)

### 2.5 配置CM数据库

a.创建用户及初始化数据库

【在所有节点均创建这个用户】

```shell
useradd --system --home=/opt/cm-5.14.1/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm 
```

------

指向cm-server

```shell
vim /opt/cm-5.14.1/etc/cloudera-scm-agent/config.ini 将其中的 server_host=cm-server
```

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_123305.2beyyzs7ua68.webp)

```shell
/opt/cm-5.14.1/share/cmf/schema/scm_prepare_database.sh mysql cmdb -h"cm-server" -uroot -pmysqladmin --scm-host cm-server scm scm scm 
```

> ```
> /opt/cm-5.14.1/share/cmf/schema/scm_prepare_database.sh <databaseType> [options]  <databaseName> <databaseUser> <password>
> ```
>
> --scm-host 指定Cloudera Manager Server安装在的 hostname 可以不指定，在数据库和cms安装在同一个主机的情况下
>
> SCM_HOST=, SCM_DATABASE=, SCM_USER=, SCM_PASSWORD=,

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_123457.5a1amnjlqz40.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_123738.6p0kdbgilnw0.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_123906.7e3drfykyjs0.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_123952.1b1sl30lejk0.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_124046.1v7rotf5auzk.webp)

查看新增目录

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_124305.3phneyouiww0.webp)

b.将cm-server修改完成的文件分发到其他各节点

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_122933.6a1knqmbbow0.webp)

```SHELL
for i in `seq 1 2`;do scp -r /opt/cm-5.14.1 hadoop-$i:/opt/;done
```

c.创建本地源 【只在server节点】【**parcel.sha1 名字要改为 parcel.sha**】

```shell
mv CDH-5.14.2-1.cdh5.14.2.p0.3-el7.parcel* manifest.json /opt/cloudera/parcel-repo/
```

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_125322.74m7d1pllvo0.webp)

### 2.6 安装CDH

a.启动服务
在cm-server启动server和agent服务，在其他节点启动agent服务

```shell
/opt/cm-5.14.1/etc/init.d/cloudera-scm-server start
/opt/cm-5.14.1/etc/init.d/cloudera-scm-agent start
```

安装服务时执行脚本需要这个工具：

```shell
yum -y install perl perl-devel
```

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_125449.2vw666yi07c0.webp)

安装完成后如果发现 cm-server 节点的硬盘空间不够时，可以扩展磁盘，操作如下：

```shell
fdisk /dev/sda
m # look for help
n # create new partition
p # choose as primary partition
enter # default partition number
enter # default start section
enter # default end section, all the free space
w     # save changes

reboot 

lsblk -f # find that the new partition has no type and no mount point 

mkfs -t xfs /dev/sdax # sdax is the new assigned partition

# 临时挂载
mount /dev/sdax / # mount the new partition to / point 
# 永久挂载
[root@cm-server ~]# blkid
/dev/sda1: UUID="4a42a150-2185-4cd4-b1e4-72cb397b2be1" TYPE="xfs"
/dev/sda2: UUID="G2rfmX-Kao9-HeBI-Jvdc-tuaM-I64S-EITPrk" TYPE="LVM2_member"
/dev/sda3: UUID="911209f1-bb4f-4564-abaa-df657148bd05" TYPE="xfs"
/dev/sda4: UUID="3faef55d-490b-4d57-a69c-fd29214075b2" TYPE="xfs"
/dev/sr0: UUID="2020-11-04-11-36-43-00" LABEL="CentOS 7 x86_64" TYPE="iso9660" PTTYPE="dos"
/dev/mapper/centos-root: UUID="d48f3a9f-9143-40a9-b919-411d5aedc781" TYPE="xfs"
/dev/mapper/centos-swap: UUID="10fce261-d155-4f79-8c04-eda586780382" TYPE="swap"

vi /etc/fstab
# 按照格式填写
```

这里永久挂载存在问题，每次重启需要重新执行。

服务均启动后，可以浏览器访问cm-server的7180端口,用户名/密码为admin/admin

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_125807.2ak90vkk8qtc.webp)

接受协议继续

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_125818.6lpbqe9duxw0.webp)

可以选择适用60天

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_125824.3xcxj28bdfs0.webp)

提示一些涉及许可证的信息

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_125830.4iz3br0t9fy0.webp)

勾选管理的主机继续操作

受控主机不够时检查日志【/opt/cm-5.14.1/log/cloudera-scm-agent/cloudera-scm-agent.log 】

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_125847.4ogbd9zq07i0.webp)

选择CDH-5.14版本

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_125914.24rwfhrnd0w0.webp)

parcel安装

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130509.3lu78hctmim0.webp)

主机正确性检查

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130527.bs3mitg6ehk.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130608.1lmr1nn7ots0.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130549.69cuxsje9m80.webp)

群集设置（选择安装的服务）

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130637.5v1hygh92lo0.webp)

自定义角色分配，选择安装在哪个节点上

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130644.51ckncyt8540.webp)

数据库设置
需要提前创建数据库及授权其他节点可以正常连接

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130826.6j5k2hd9dzk0.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130848.3ioo6wh9kr40.webp)

审核更改

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130925.6kphglcg6q40.webp)

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_130932.1s3ptdlcr21s.webp)

集群安装

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_131127.3qgzypx80ko0.webp)

完成安装

![img](https://cdn.jsdelivr.net/gh/HongdaChen/image-home@master/20210612/snipaste_20220414_131544.1dr9uc8bnbkw.webp)

后期可添加服务

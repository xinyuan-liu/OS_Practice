# 第四次作业

## 分布式文件系统

### HDFS

一个HDFS集群有一个NameNode，用来存放整个文件系统的元数据，同时控制client对整个文件系统的访问。其余的节点为DataNode，用来存放实际数据的。一个文件在HDFS中会被划分为多个block，分散存储在各个DataNode上。NameNode维护这一文件的各个block与DataNode之间的映射关系。用户在访问文件时，先请求NameNode打开文件，获取这个文件的元数据，然后直接联系DataNode去对block进行读写。

可以看出，HDFS过于中心化，高度依赖NameNode，这样导致系统的可靠性降低，一旦NameNode出现故障则该系统就会瘫痪。同时，用于所有的访问请求都需要通过NameNode进行解析和控制，NameNode的性能会成为整个系统的性能瓶颈。当然，这些问题可以通过引入新的的机制来解决。

使用方式：
```
# 格式化一个新的分布式文件系统：
$ bin/hadoop namenode -format
# 启动Hadoop守护进程：
$ bin/start-all.sh
# 将输入文件拷贝到分布式文件系统：
$ bin/hadoop fs -put conf input
# 运行发行版提供的示例程序：
$ bin/hadoop jar hadoop-*-examples.jar grep input output 'dfs[a-z.]+'
# 将输出文件从分布式文件系统拷贝到本地文件系统查看：
$ bin/hadoop fs -get output output 
$ cat output/*
# 在分布式文件系统上查看输出文件：
$ bin/hadoop fs -cat output/*
# 完成全部操作后，停止守护进程：
$ bin/stop-all.sh
```
### GlusterFS

GlusterFS采用弹性哈希算法在存储池中定位数据，而不是采用集中式或分布式元数据服务器索引。诸如HDFS之类的分布式文件系统，元数据服务器通常会导致I/O性能瓶颈和单点故障问题。GlusterFS中，数据的存储位置信息不需要查看索引或者向其他服务器查询。这种设计机制并行化了数据访问，但是在客户端的开销会比较大。

使用方式：
```
$ gluster peer probe server2 # add server2 to the storage pool
$ gluster volume create gv0 replica 2 server1:/data/brick1/gv0 server2:/data/brick1/gv0 # create the copying volume
$ gluster volume start gv0 # start the copying volume
$ mount -t glusterfs server1:/gv0 /mnt # mount the colume in clients
$ gluster volume info # see the information of a volume
$ gluster peer status # see the nodes' information
```
## 联合文件系统

### AUFS

AUFS是一种联合文件操作系统。联合文件系统将多个目录合并成一个虚拟文件系统,成员目录称为虚拟文件系统的一个分支（branch）。
每个branch可以指定readwrite/whiteout‐able/readonl权限，一般情况下,aufs只有最上层的branch具有读写权限,其余branch均为只读权限，对只读层文件的修改实际上体现为读写层对只读层的覆盖。

使用方式：
```
sudo mount -t aufs -o br=/a:/b:/c none /aufs
```

## 安装配置一种分布式文件系统

使用glusterfs，在前两台虚拟机上安装GlusterFS-server：
```
sudo add-apt-repository ppa:gluster/glusterfs-3.10
sudo apt update
sudo apt install glusterfs-server
```

在第三台虚拟机上安装GlusterFS-client
```
sudo add-apt-repository ppa:gluster/glusterfs-3.10
sudo apt update
sudo apt install glusterfs-client
```

两台机器互相侦听，并创建brick文件夹
```
sudo gluster peer probe 192.168.199.122
mkdir brick
```
```
sudo gluster peer probe 192.168.199.123
mkdir brick
```

在任意一台机器上，新建一个Volume
```
sudo gluster volume create myVolume replica 2 192.168.199.122:/home/liuxinyuan/brick 192.168.199.123:/home/liuxinyuan/brick force
```
这里使用replica模式作为容错的需要，数据在两台机器上互相备份。

使用force选项强制要求GlusterFS在root分区存放brick

启动新建的Volume
```
sudo gluster volume start myVolume
```
输入`gluster volume info`查看Volume的信息
```
Volume Name: myVolume
Type: Replicate
Volume ID: 8fd3384c-38db-03a4-3a2d-777fa33scec
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: 192.168.199.122:/home/liuxinyuan/brick
Brick2: 192.168.199.123:/home/liuxinyuan/brick
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
```

在第三台机器上挂载这个Volume:
```
mkdir gfsVolume
sudo mount -t glusterfs 192.168.199.122:/myVolume /home/liuxinyuan/gfsVolume
```
在第三台机器上能够正常进行读写。

为测试容错机制，关掉一台机器，在第三台机器上仍能够正常读写。

## 在docker中挂载glusterfs

创建一个ubuntu镜像，安装nginx和glusterfs-client：
```
FROM ubuntu:xenial
RUN apt -y update && apt install -y nginx glusterfs-client
```
在容器启动时，将glusterfs挂载到容器里，并启动nginx
```
CMD mkdir /gfsVolume && mount -t glusterfs 192.168.199.122:/myVolume /gfsVolume && cp /gfsVolume/index.html /var/www/html/index.html && nginx -g 'daemon off;'
```
容器启动时，要加入```--privileged```选项才能正确执行挂载命令。

## 用联合文件系统制作 Docker 镜像

创建一个基础镜像为ubuntu的docker镜像：
```
sudo docker create -it --name aaa ubuntu /bin/bash
```
启动容器
```
sudo docker start -i aaa
```
在另一个终端中，使用```df -hT```查看挂载情况，其中
```
none           aufs      22575612 11676688   9729108  55% /var/lib/docker/aufs/mnt/fe9145cba94f7e5d752164b5890f6fc421ff9b4fd9e82b048868a275b0366c03
```
就是docker镜像的文件系统的挂载记录

查看文件系统的层级结构
```
cd /var/lib/docker/aufs
cat layer/fe9145cba94f7e5d752164b5890f6fc421ff9b4fd9e82b048868a275b0366c03
```
输出结果为
```
fe9145cba94f7e5d752164b5890f6fc421ff9b4fd9e82b048868a275b0366c03-init
7ecefbb6c23799e205b5f02546d846fef910611fddfa10f6c581ac712c6f94f0
c66da2a8ae044a0fd47da1f954628b553d6f77d24b7a582dabe0326f02f143ff
5fdf67f34dcb41a8c37da3758fb296af541c732d7c72d28bfedff0ddb9285332
91f8e29bc6e3454e4d2fd0542ef33094fad0629eb918b631a5711ac19ceb8859
f158bb4afce879312ded3268d7939ef03d2e5b4696e29ba0ff0a9f3311874ab8
```
其中```fe9145cba94f7e5d752164b5890f6fc421ff9b4fd9e82b048868a275b0366c03-init```为容器初始化过程中生成的，不需要拷贝。将其他几层拷贝出来。

在容器中安装vim：
```
apt update
apt install vim
```
将最高层```fe9145cba94f7e5d752164b5890f6fc421ff9b4fd9e82b048868a275b0366c03```拷贝出来。

挂载该文件系统。
```
sudo mount -t aufs -o br=/home/liuxinyuan/=ro\
:/home/liuxinyuan/4=ro:/home/liuxinyuan/3=ro\
:/home/liuxinyuan/2=ro:/home/liuxinyuan/1=ro\
:/home/liuxinyuan/0=ro none /home/liuxinyuan/myMount
```
打包并创建新镜像：
```
sudo tar -c myMount | docker import - bbb
```
运行容器，能正常使用vim。

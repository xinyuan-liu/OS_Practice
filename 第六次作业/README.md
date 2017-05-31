# 第六次作业

## 阅读Paxos算法的材料并用自己的话简单叙述

Paxos 算法用于处理基于消息传递机制的分布式系统通信过程中，可能出现的一致性问题。
首先将议员的角色分为proposers，acceptors，和learners（允许身兼数职）。proposers提出提案，提案信息包括提案编号和提议的value；acceptor收到提案后可以接受（accept）提案，若提案获得多数acceptors的接受，则称该提案被批准（chosen）；learners只能“学习”被批准的提案。划分角色后，就可以更精确的定义问题

决议（value）只有在被proposers提出后才能被批准（未经批准的决议称为“提案（proposal）”）；

在一次Paxos算法的执行实例中，只批准（chosen）一个value；

learners只能获得被批准（chosen）的value。

通过一个决议分为两个阶段：

prepare阶段：

proposer选择一个提案编号n并将prepare请求发送给acceptors中的一个多数派；

acceptor收到prepare消息后，如果提案的编号大于它已经回复的所有prepare消息，则acceptor将自己上次接受的提案回复给proposer，并承诺不再回复小于n的提案；

批准阶段：

当一个proposer收到了多数acceptors对prepare的回复后，就进入批准阶段。它要向回复prepare请求的acceptors发送accept请求，包括编号n和根据P2c决定的value（如果根据P2c没有已经接受的value，那么它可以自由决定value）。
在不违背自己向其他proposer的承诺的前提下，acceptor收到accept请求后即接受这个请求。

这个过程在任何时候中断都可以保证正确性。例如如果一个proposer发现已经有其他proposers提出了编号更高的提案，则有必要中断这个过程。因此为了优化，在上述prepare过程中，如果一个acceptor发现存在一个更高编号的提案，则需要通知proposer，提醒其中断这次提案。


## 模拟Raft协议工作的一个场景并叙述处理过程

Raft把集群中的节点分为三种状态： Leader 、 Follower 、 Candidate ，理所当然每种状态负责的任务也是不一样的，Raft运行时提供服务的时候只存在Leader与Follower两种状态；

Leader（领导者）：负责日志的同步管理，处理来自客户端的请求，与Follower保持这heartBeat的联系；

Follower（追随者）：刚启动时所有节点为Follower状态，响应Leader的日志同步请求，响应Candidate的请求，把请求到Follower的事务转发给Leader；

Candidate（候选者）：负责选举投票，Raft刚启动时由一个节点从Follower转为Candidate发起选举，选举出Leader后从Candidate转为Leader状态；

一个简单的例子：

1. 任何一个服务器都可以成为一个候选者Candidate，它向其他服务器Follower发出要求选举自己的请求：
![img1](https://raw.githubusercontent.com/xinyuan-liu/OS_Practice/master/第六次作业/img/raft1.png)

2. 其他服务器同意了，发出OK。
![img2](https://raw.githubusercontent.com/xinyuan-liu/OS_Practice/master/第六次作业/img/raft2.png)

注意如果在这个过程中，有一个Follower宕机，没有收到请求选举的要求，因此候选者可以自己选自己，只要达到N/2 + 1 的大多数票，候选人还是可以成为Leader的。

3. 这样这个候选者就成为了Leader领导人，它可以向Follower们发出指令，比如进行日志复制。
![img3](https://raw.githubusercontent.com/xinyuan-liu/OS_Practice/master/第六次作业/img/raft3.png)

4. 以后通过心跳进行日志复制的通知
![img4](https://raw.githubusercontent.com/xinyuan-liu/OS_Practice/master/第六次作业/img/raft4.png)

5. 如果一旦这个Leader当机崩溃了，那么Follower中有一个成为候选者，发出邀票选举。
![img5](https://raw.githubusercontent.com/xinyuan-liu/OS_Practice/master/第六次作业/img/raft5.png)

6. Follower同意后，其成为Leader，继续承担日志复制等指导工作。
!![img6](https://raw.githubusercontent.com/xinyuan-liu/OS_Practice/master/第六次作业/img/raft6.png)

## 简述Mesos的容错机制并验证

### Mesos的容错机制

#### Master

Mesos使用热备份（hot-standby）设计来实现Master节点集合。一个Master节点与多个备用（standby）节点运行在同一集群中，并由开源软件Zookeeper来监控。Zookeeper会监控Master集群中所有的节点，并在Master节点发生故障时管理新Master的选举。建议的节点总数是5个，实际上，生产环境至少需要3个Master节点。 Mesos决定将Master设计为持有软件状态，这意味着当Master节点发生故障时，其状态可以很快地在新选举的Master节点上重建。 Mesos的状态信息实际上驻留在Framework调度器和Slave节点集合之中。当一个新的Master当选后，Zookeeper会通知Framework和选举后的Slave节点集合，以便使其在新的Master上注册。彼时，新的Master可以根据Framework和Slave节点集合发送过来的信息，重建内部状态。

#### Framework

Framework调度器的容错是通过Framework将调度器注册2份或者更多份到Master来实现。当一个调度器发生故障时，Master会通知另一个调度来接管。需要注意的是Framework自身负责实现调度器之间共享状态的机制。

#### Slave（Agent）

Mesos实现了Slave的恢复功能，当Slave节点上的进程失败时，可以让执行器/任务继续运行，并为那个Slave进程重新连接那台Slave节点上运行的执行器/任务。当任务执行时，Slave会将任务的监测点元数据存入本地磁盘。如果Slave进程失败，任务会继续运行，当Master重新启动Slave进程后，因为此时没有可以响应的消息，所以重新启动的Slave进程会使用检查点数据来恢复状态，并重新与执行器/任务连接。

### 实验验证

这里主要验证master出错的情况下Mesos的应对机制。

安装zookeeper：
```
wget http://mirror.nexcess.net/apache/zookeeper/stable/zookeeper-3.4.10.tar.gz
tar -zxf zookeeper-3.4.10.tar.gz
```

在三台机器上分别修改zookeeper的配置文件
```
dataDir=/var/lib/zookeeper

server.1=192.168.199.108:2888:3888
server.2=192.168.199.109:2888:3888
server.3=192.168.199.110:2888:3888
```

在三台机器上分别启动master：
```
mesos master --zk=zk://192.168.199.108:2181,192.168.199.109:2181,192.168.199.110:2181/mesos --quorum=2 --ip=127.0.0.1  --hostname=mas1 --work_dir=./mesos --log_dir=./mesos
```

日志中
```
A new leading master (UPID=master@192.168.199.108:5050) is detected
```
显示192.168.199.108被选举为leader。

在192.168.199.108机器上kill掉mesos master进程。
```
A new leading master (UPID=master@192.168.199.110:6060) is detected
The newly elected leader is master@192.168.199.110:6060 with id 55a901ac-4352-457f-9942-7315450300c0
```
192.168.199.110被选举为新的leader。

## 综合作业

### 创建docker镜像

镜像中需要安装etcd、Jupiter、openssh-server等程序。
```

FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y wget

RUN wget -P /root https://github.com/coreos/etcd/releases/download/v3.1.7/etcd-v3.1.7-linux-amd64.tar.gz && tar -zxf /root/etcd-v3.1.7-linux-amd64.tar.gz -C /root
RUN ln -s /root/etcd-v3.1.7-linux-amd64/etcd /usr/local/bin/etcd && ln -s /root/etcd-v3.1.7-linux-amd64/etcdctl /usr/local/bin/etcdctl

RUN apt-get update
RUN apt-get install -y ssh
RUN apt-get install -y openssh-server
RUN apt-get install -y python3-pip
RUN pip3 install jupyter
RUN apt-get install -y sudo
ADD /mnt/code.py /home/admin/code.py
RUN useradd admin
RUN echo "admin:admin" | chpasswd
RUN echo "admin   ALL=(ALL)       ALL" >> /etc/sudoers
RUN mkdir /var/run/sshd

EXPOSE 22
USER admin
WORKDIR /home/admin
CMD ["/bin/bash"]
```

### 搭建 Calico 容器网络
与执行上次作业相同的命令，创建一个容器网络
```
docker network create --driver calico --ipam-driver calico-ipam --subnet=192.0.2.0/24 calico
```

### GlusterFS分布式存储

这一步的配置过程与第四次作业类似。具体过程不再重复。

### 互相免密码ssh登录

使用GlusterFS提供的分布式存储共享各个节点的私钥。需要执行命令：
```
ssh-keygen -f /home/admin/.ssh/id_rsa -t rsa && cat /home/admin/.ssh/id_rsa.pub >> /home/admin/gfsVolume/authorized_keys && /etc/init.d/ssh start
```
生成公私钥，并将公钥附加到分布式存储卷中。

### 启动etcd

使用Python的subprocess包。
```
def run_etcd(ip):
    args = ['etcd', '--name', 'p'+ip[-1], '--initial-advertise-peer-urls', 'http://'+ip+':2380','--listen-peer-urls', 'http://'+ip+ ':2380','--listen-client-urls', 'http://'+ip+':2379,http://127.0.0.1:2379','--advertise-client-urls', 'http://'+ip+':2379','--initial-cluster-token', 'etcd-cluster-hw5','--initial-cluster', '192.0.2.100=http://192.0.2.100:2380,192.0.2.101=http://192.0.2.101:2380,192.0.2.102=http://192.0.2.102:2380' ,'--initial-cluster-state', 'new']
    subprocess.Popen(args)
```
在容器启动时调用这个函数即可。

### 更新host表
```
def update_host:
    f=open("host","w")
    err=0
    for i in range(n):
        flag=os.system('etcdctl get /leader/192.0.1.10' + str(i))
        if flag==0:
            f.write("192.0.1.10" + str(i)+" host0\n")
            break
    cnt=1
    for i in range(0,n):
        flag=os.system('etcdctl get /follower/192.0.1.10' + str(i))
        if flag==0:
            f.write("192.0.1.10" + str(i)+" host"+str(cnt)+"\n")
            cnt=cnt+1
    f.close()
    os.system("mv host /etc/hosts")
```

### 守护程序
```
while True:
    try:
        stats_reponse = urllib.request.urlopen(stats_request)
    except urllib.error.URLError as e:
        print('[WARN] ', e.reason)
        print('[WARN] Wating etcd...')

    else:
        stats_json = stats_reponse.read().decode('utf-8')
        data = json.loads(stats_json)
        if data['state'] == 'StateLeader':
            if leader_flag == 0: #如果是第一次成为leader，需要启动jupyter
                leader_flag = 1 
                args = ['jupyter', 'notebook', '--NotebookApp.token=', '--ip=0.0.0.0', '--port=8888']
                subprocess.Popen(args)
                os.system('etcdctl set /leader/' + ip + ' ' + "1 --ttl 30")
            elif data['state'] == 'StateFollower':
                os.system('etcdctl set /follower/' + ip + ' ' + "1 --ttl 30")
        update_host(n)
        time.sleep(10)
```

### framework

framework基本上仍然可以使用上一次作业的scheduler.py，不区分是否运行Jupiter notebook即可。

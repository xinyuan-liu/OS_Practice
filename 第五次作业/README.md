# 第五次作业
## Linux内核如何对 IP 数据包进行处理

Linux内核通过Netfilter进行IP数据包处理，Netfilter为多种网络协议各提供了一套钩子函数，在内核态提供数据包过滤、内容修改等功能；netfilter 向用户态暴露了一些接口，用户态程序可以通过它们来控制内核的包路由规则。其中，iptables 就是一个为用户提供 IP 数据包管理功能的用户态程序。

一个数据包被处理的具体流程为：

数据包将触发挂在PREROUTING上的回调函数，如果该节点的规则允许通过，将进行网络地址转换。进行路由判断，如果是本机，则触发INPUT上的回调函数，通过过滤规则后，放行并进入本机上层协议栈；如果不是本机，则触发FORWARD上的回调函数，通过过滤规则和网络地址转换后，就进入POSTROUTING的处理阶段。由本机上层协议栈发出的数据包，需要经过路由判断下一个到达的网络节点，然后进行网络地址转换接着就和FORWARD过来的包一样，规则过滤，网络地址转换进入POSTROUTING阶段封包发出。

iptables还提供了一些更抽象的表供用户使用：

filter：包含INPUT链、OUTPUT链、FORWARD链，根据规则过滤特定的包。

nat：包含OUTPUT链、PREROUTING链、POSTROUTING链，在相应位置进行网络地址转换。

mangle：对包头进行修改，例如更改路由tag等。

## 使用 iptables

### 拒绝来自某一特定IP地址的访问

拒绝本机IP的访问：

```
sudo iptables -A INPUT -s 192.168.199.116 -j REJECT
```

删除这条规则，使用命令
```
sudo iptables -D INPUT -s 192.168.199.116 -j REJECT
```

### 拒绝来自某一特定MAC地址的访问

利用iptables提供的mac模块，与之前类似的：
```
sudo iptables -A INPUT -m mac --mac-source ac:bc:32:d2:6d:b3  -j DROP
```

### 只开放本机的http服务，其余协议与端口均拒绝

只接收80端口的访问，其他端口全部drop：
```
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -P INPUT DROP
```
此时仅能通过80端口访问服务器。

### 拒绝回应来自某一特定IP地址的ping命令

使用 -p icmp 选项过滤icpm包，icmp-type设置为8。

```
sudo iptables -A INPUT -s 192.168.199.116 -p icmp --icmp-type 8 -j ACCEPT
```

## Linux网络设备工作原理

### Bridge

Bridge 是一种以协议独立的方式将两个以太网段连接在一起的方法，工作在 OSI 模型的数据链路层。它根据 MAC 地址转换表提供的信息来转发帧，工作过程类似于交换机。所有协议可以通过 Bridge 透明地进行。

Linux内核通过一个虚拟的网桥设备来实现桥接的，这个设备可以绑定若干个以太网接口设备，从而将它们桥接起来。如下图所示：

[img]

网桥设备 br0 绑定了 eth0 和 eth1。对于网络协议栈的上层来说，只看得到 br0，因为桥接是在数据链路层实现的，上层不需要关心桥接的细节。于是协议栈上层需要发送的报文被送到 br0，网桥设备的处理代码再来判断报文该被转发到 eth0 或是 eth1或者两者皆是；反过来，从 eth0 或从 eth1 接收到的报文被提交给网桥的处理代码，在这里会判断报文该转发、丢弃、或提交到协议栈上层。

### VLAN

VLAN即虚拟局域网，一个vlan能够模拟一个常规的交换网络，实现了将一个物理的交换机划分成多个逻辑的交换网络。而不同的VLAN之间如果要进行通信就要通过三层协议来实现，在二层协议里插入额外的VLAN tag，同时保持和传统二层设备的兼容性。VLAN设备是以母子关系成对出现的，母设备相当于现实世界中的交换机TRUNK口，用于连接上级网络，子设备相当于普通接口用于连接下级网络。当一个子设备有一包数据需要发送时，数据将被加入VLAN Tag然后从母设备发送出去。当母设备收到一包数据时，它将会分析其中的VLAN Tag，如果有对应的子设备存在，则把数据转发到那个子设备上并根据设置移除VLAN Tag，否则丢弃该数据。

### veth

VETH 即 virtual Ethernet，是一种 Linux 专门为容器技术设计的网络通信机制。它的作用是把从一个 network namespace 发出的数据包转发到另一个 namespace。VETH 设备总是成对出现的，一个是 container 之中，另一个在 container 之外，即在真实机器上能看到的。它能够反转通讯数据的方向，需要发送的数据会被转换成需要收到的数据重新送入内核网络层进行处理，从而间接的完成数据的注入。

## Calico容器网络的收发数据包的过程

calico是纯三层的SDN 实现，它基于BPG 协议和Linux自身的路由转发机制，不依赖特殊硬件，容器通信也不依赖iptables NAT或Tunnel 等技术。能够方便的部署在物理服务器、虚拟机（如 OpenStack）或者容器环境下。同时calico自带的基于iptables的ACL管理组件非常灵活，能够满足比较复杂的安全隔离需求。

每个主机上都部署了calico/node作为虚拟路由器，并且可以通过calico将宿主机组织成任意的拓扑集群。当集群中的容器需要与外界通信时，就可以通过BGP协议将网关物理路由器加入到集群中，使外界可以直接访问容器IP，而不需要做任何NAT之类的复杂操作。

当容器创建时，calico为容器生成veth pair，一端作为容器网卡加入到容器的网络命名空间，并设置IP和掩码，一端直接暴露在宿主机上，并通过设置路由规则，将容器IP暴露到宿主机的通信路由上。于此同时，calico为每个主机分配了一段子网作为容器可分配的IP范围，这样就可以根据子网的CIDR为每个主机生成比较固定的路由规则。

当容器需要跨主机通信时，主要经过下面的简单步骤：

容器流量通过veth pair到达宿主机的网络命名空间上。

根据容器要访问的IP所在的子网CIDR和主机上的路由规则，找到下一跳要到达的宿主机IP。

流量到达下一跳的宿主机后，根据当前宿主机上的路由规则，直接到达对端容器的veth pair插在宿主机的一端，最终进入容器。

## weave网络通信模型

weave通过在docker集群的每个主机上启动虚拟的路由器，将主机作为路由器，形成互联互通的网络拓扑，在此基础上，实现容器的跨主机通信。

在每一个部署Docker的主机（可能是物理机也可能是虚拟机）上都部署有一个W（即weave router，它本身也可以以一个容器的形式部署）。weave网络是由这些weave routers组成的对等端点（peer）构成，并且可以通过weave命令行定制网络拓扑。

每个部署了weave router的主机之间都会建立TCP和UDP两个连接，保证weave router之间控制面流量和数据面流量的通过。控制面由weave routers之间建立的TCP连接构成，通过它进行握手和拓扑关系信息的交换通信。控制面的通信可以被配置为加密通信。而数据面由weave routers之间建立的UDP连接构成，这些连接大部分都会加密。这些连接都是全双工的，并且可以穿越防火墙。 

## weave 与 calico 的比较

### weave

weave默认基于UDP承载容器之间的数据包，并且可以完全自定义整个集群的网络拓扑，比较灵活。

weave自定义容器数据包的封包解包方式，不够通用，传输效率比较低，性能上的损失也比较大。

集群配置比较负载，需要通过weave命令行来手工构建网络拓扑，在大规模集群的情况下，加重了管理员的负担。
 
### calico

跨主机通信时，整个通信路径完全没有使用NAT或者UDP封装，性能上的损耗确实比较低

calico目前只支持TCP、UDP、ICMP、ICMPv6协议，而不支持其他四层协议。

基于三层实现通信，在二层上没有任何加密包装，因此只能在私有的可靠网络上使用。

流量隔离基于iptables实现，并且从etcd中获取需要生成的隔离规则，有一些性能上的隐患。

## 编写一个mesos framework

安装etcd，并关闭etcd服务。
```
apt install etcd
service etcd stop
```
创建etcd集群环境：
```
etcd --name node0 --initial-advertise-peer-urls http://192.168.199.108:2380 \
--listen-peer-urls http://192.168.199.108:2380 \
--listen-client-urls http://192.168.199.108:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://192.168.199.108:2379 \
--initial-cluster-token etcd-cluster-hw5 \
--initial-cluster node0=http://192.168.199.108:2380,node1=http://192.168.199.109:2380,node2=http://192.168.199.110:2380 \
--initial-cluster-state new
```
在另两台机器上运行类似的命令。

退出swarm集群网络：
```
docker swarm leave --force
```

重启docker daemon，使docker支持etcd：
```
service docker stop
dockerd --cluster-store etcd://192.168.199.108:2379 &
```
在另两台机器上运行类似的命令。

安装calico并启动：
```
wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calicoctl/releases/download/v1.1.3/calicoctl
chmod +x /usr/local/bin/calicoctl
calicoctl node run --ip 192.168.199.108 --name node0
```

添加IP池：
```
cat << EOF | calicoctl create -f -
- apiVersion: v1
  kind: ipPool
  metadata: 
    cidr: 192.0.2.0/24
EOF
```

创建一个容器网络：
```
docker network create --driver calico --ipam-driver calico-ipam --subnet=192.0.2.0/24 calico
```

制作docker镜像：
```
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install ssh

RUN useradd -m calico
RUN echo "admin:calico" | chpasswd

RUN mkdir /var/run/sshd

USER root
EXPOSE 22
CMD ["/usr/sbin/sshd","-D"]
```

```
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install ssh
RUN apt-get install python3-pip
RUN pip3 install jupyter

RUN useradd -m calico
RUN echo "admin:calico" | chpasswd
RUN mkdir /var/run/sshd

USER calico
EXPOSE 22
WORKDIR /home/calico
CMD ["jupyter","notebook","--NotebookApp.token=","--ip=192.0.2.1","--port=8888"]
```

在三台机器上分别运行master和agent，并在agent上设置端口转发：
```
configurable-http-proxy --default-target=http://192.0.2.100:8888 --port=8888
```

最后启动scheduler.py，即可访问jupyter notebook。

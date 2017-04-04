# 第三次作业
## 安装配置Docker

在本地安装docker，运行docker info结果：

## docker基本命令

### docker run

用法:
```docker run [OPTIONS] IMAGE [COMMAND] [ARG...]```

该命令首先会从特定的image创之上创建一层可写的Container，然后通过start命令来启动它。

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：

1.检查本地是否存在指定的镜像，不存在就从公有仓库下载

2.利用镜像创建并启动一个容器

3.分配一个文件系统，并在只读的镜像层外面挂载一层可读写层

4.从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去

5.从地址池配置一个 ip 地址给容器

6.执行用户指定的应用程序

7.执行完毕后容器被终止

### 开启/停止/重启container（start/stop/restart）

用法：
```docker start/stop/restart [OPTIONS] CONTAINER [CONTAINER...]```


容器可以通过run新建一个来运行，也可以重新start已经停止的container，
但start不能够再指定容器启动时运行的指令，因为docker只能有一个前台进程。

容器stop（或Ctrl+D）时，会在保存当前容器的状态之后退出，
下次start时保有上次关闭时的更改。

### docker images

列出本地的所有镜像。

用法：
```docker images [OPTIONS] [REPOSITORY[:TAG]]```

选项：

```
  -a, --all             展示所有，默认隐藏中间过程镜像
      --digests         显示摘要
  -f, --filter value    根据条件过滤输出
      --format string   用go模板格式化输出
      --no-trunc        不要裁剪输出（id）
  -q, --quiet           只打印镜像ID
```
### docker commit

将容器的状态保存为镜像。

用法：
```docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]```

选项：
```
  -a, --author string    作者
  -c, --change value     使用Dockerfile中的指令
  -m, --message string   说明
  -p, --pause            提交时暂停容器，默认开启
```

### docker network

docker network 命令下有6个子命令，分别是：

 * docker network connect

 将一个容器连接入网络。

 用法：
```docker network connect [OPTIONS] NETWORK CONTAINER```

 * docker network create

 创建一个网络。

 用法：
```docker network create [OPTIONS] NETWORK```

 * docker network disconnect

 将一个容器断开网络。

 用法：
 ```docker network disconnect [OPTIONS] NETWORK CONTAINER```

 * docker network inspect
 
 展示一个或多个网络的详细信息。

 用法：
```docker network inspect [OPTIONS] NETWORK [NETWORK...]```

 * docker network ls 

 列举网络。
 
 用法：
 ```docker network ls [OPTIONS]```
 
 * docker network rm
 
 移除一个或多个网络。

 用法：
 ```docker network rm NETWORK [NETWORK...]```

## 创建Docker镜像和搭建nginx服务器

* 创建和启动镜像

 创建一个基础镜像为ubuntu的docker镜像，镜像名为ubuntu_nginx，并将容器的80端口映射到宿主机的8080端口。

 ```docker run -i -t --name ubuntu_nginx -p 8080:80 ubuntu /bin/bash```
 
* 安装nginx服务器

 ```
 apt-get update
 apt-get install nginx
 ```
 
* 启动nginx服务器，利用tail命令将访问日志输出到标准输出流，编辑web服务器主页。

 ```
 nginx
 tail -f /var/log/nginx/access.log
 ```
 
* 创建network

 ```
 exit   #停止容器
 docker commit ubuntu_nginx ubuntu_nginx2   #保存镜像
 docker network create -d bridge network_bridge #创建网络
 docker run --name nginx ubuntu_nginx2 nginx -g 'daemon off;' &    #后台运行容器
 sudo docker network connect network_bridge nginx   #将容器连接入网络
 ```
 
 成功将容器加入自定义的bridge网络，IP地址为172.18.0.2。
 
 在宿主机成功访问nginx服务器。
 
## Docker的网络模式

### Bridge模式

Brdige桥接模式为Docker Container创建独立的网络栈，
保证容器内的进程组使用独立的网络环境，
实现容器间、容器与宿主机之间的网络栈隔离。
另外，Docker通过宿主机上的网桥(docker0)来连通容器内部的网络栈与宿主机的网络栈，
实现容器与宿主机乃至外界的网络通信。
Docker为每个Bridge网络分配一个网段，一个Bridge网络内的容器在同一个网段内。
容器也可以加入多个bridge网络。
 
### Host模式

Host模式不为容器创建一个隔离的网络环境，
该模式下的Docker Container会和host宿主机共享同一个网络namespace，
故Docker Container可以和宿主机一样，
使用宿主机的eth0，实现和外界的通信。
换言之，Docker Container的IP地址即为宿主机eth0的IP地址。
采用host模式的Docker Container，
可以直接使用宿主机的IP地址与外界进行通信，
若宿主机的eth0是一个公有IP，那么容器也拥有这个公有IP。
同时容器内服务的端口也可以使用宿主机的端口，无需额外进行NAT转换。
当然，有这样的方便，肯定会损失部分其他的特性，
最明显的是Docker Container网络环境隔离性的弱化，
即容器不再拥有隔离、独立的网络栈。
另外，使用host模式的Docker Container虽然可以让容器内部的服务和传统情况无差别、
无改造的使用，但是由于网络隔离性的弱化，
该容器会与宿主机共享竞争网络栈的使用；
另外，容器内部将不再拥有所有的端口资源，
原因是部分端口资源已经被宿主机本身的服务占用。

### Null模式

Null模式下容器仅有一个本地回环网络127.0.0.1，
没有任何网络配置。另外，容器不能通过网络管理指令来断开null模式下的网络。

### Overlay模式

Overlay模式主要提供了跨主机的容器通信的解决方案，
Docker的Overlay模式使用docker内置的swarm来管理结点。
部署时，使用一台主机作为swarn的管理节点，其他主机作为worker节点加入集群。
同时，docker提供了docker_gwbridge使集群内的容器可以和集群外的网络通信。

## mesos与docker的交互

* docker.cpp中定义了Docker类，这个类相当于一个Docker的API，
封装了一些Docker的命令，比如create、run、stop、kill等等。
 
* spec.cpp主要负责解析Info信息。
* executor.cpp实现了一个mesos的framework的executor，负责调用Docker类中的方法管理Docker，运行task。

### Run函数

 1. 检查containerInfo中是否有Docker
```Cpp
  if (!containerInfo.has_docker()) {
    return Failure("No docker info found in container info");
  }
```
 2. 构建命令参数
```Cpp
  vector<string> argv;
  argv.push_back(path);
  argv.push_back("-H");
  argv.push_back(socket);
  argv.push_back("run");

  if (dockerInfo.privileged()) {
    argv.push_back("--privileged");
  }
```
 3. 检查资源信息，设置命令行参数。
```Cpp
  if (resources.isSome()) {
    // TODO(yifan): Support other resources (e.g. disk).
    Option<double> cpus = resources.get().cpus();
    if (cpus.isSome()) {
      uint64_t cpuShare =
        std::max((uint64_t) (CPU_SHARES_PER_CPU * cpus.get()), MIN_CPU_SHARES);
      argv.push_back("--cpu-shares");
      argv.push_back(stringify(cpuShare));
    }

    Option<Bytes> mem = resources.get().mem();
    if (mem.isSome()) {
      Bytes memLimit = std::max(mem.get(), MIN_MEMORY);
      argv.push_back("--memory");
      argv.push_back(stringify(memLimit.bytes()));
    }
  }
```
 4. 检查环境变量，设置命令行参数。
 5. 检查磁盘挂载信息，设置命令行参数。
 6. 配置网络、hostname、端口映射。
 7. 检查和设置外部设备参数。
 8. 检查和重写entrypoint。
```Cpp
if (commandInfo.shell()) {
    argv.push_back("--entrypoint");
    argv.push_back("/bin/sh");
}
```
 9. 指定容器名和镜像名。
```Cpp
argv.push_back("--name");
argv.push_back(name);
argv.push_back(image);
```
 10. 添加容器运行的bash命令和参数。
```Cpp
  if (commandInfo.shell()) {
    if (!commandInfo.has_value()) {
      return Failure("Shell specified but no command value provided");
    }

    // Adding -c here because Docker cli only supports a single word
    // for overriding entrypoint, so adding the -c flag for /bin/sh
    // as part of the command.
    argv.push_back("-c");
    argv.push_back(commandInfo.value());
  } else {
    if (commandInfo.has_value()) {
      argv.push_back(commandInfo.value());
    }

    foreach (const string& argument, commandInfo.arguments()) {
      argv.push_back(argument);
    }
  }
```
 11. 运行容器。
```Cpp
Try<Subprocess> s = subprocess(
      path,
      argv,
      Subprocess::PATH("/dev/null"),
      _stdout,
      _stderr,
      nullptr,
      environment);
```

## Framework

和上次作业一样，也使用了pymesos的包。

与上次作业不同的是，使用Docker作为容器，framework的executor部分不需要自己完成，使用mesos提供的Docker的executor即可。

关键代码：

```Python
			#设定DockerInfo
			DockerInfo = Dict()
			DockerInfo.image = 'ubuntu_nginx2'
			DockerInfo.network = 'HOST'

			#设定ContainerInfo
			ContainerInfo = Dict()
			ContainerInfo.type = 'DOCKER'
			ContainerInfo.docker = DockerInfo

			#设定CommandInfo
			CommandInfo = Dict()
			CommandInfo.shell = False
			CommandInfo.value = 'nginx'
			CommandInfo.arguments = ['-g', "'daemon off;'"]

			#设定task信息
			task = Dict()
			task_id = str(uuid.uuid4())
			task.task_id.value = task_id
			task.agent_id.value = offer.agent_id.value
			task.name = 'nginx'
			task.container = ContainerInfo
			task.command = CommandInfo

			task.resources = [
				dict(name='cpus', type='SCALAR', scalar={'value': TASK_CPU}),
				dict(name='mem', type='SCALAR', scalar={'value': TASK_MEM}),
			]
			#启动Task
			driver.launchTasks(offer.id, [task], filters)
```

为支持Docker，需要在运行Agent时加入`--containerizers=docker,mesos --image_providers=docker --isolation=docker/runtime`命令行参数。

由于我的燕云服务器一直用不了，所以这次作业还是在本地完成的，也就没有把nginx一直架到公网IP上。

运行结果：

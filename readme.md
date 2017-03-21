# 第二次作业

## Mesos组成结构

Mesos主要由以下几部分组成：

### 1.Master

Master是整个系统的核心，负责整个系统资源的管理、分配和调度。Master负责管理各个Agent(Slave)的资源，并分配给在系统上运行的各个Framework。Master实时更新各个Agent(Slave)的资源信息，并负责分配给各个Framework。

Master部分的代码主要在src/master目录下。

### 2.Agent(Slave)

Agent(Slave)负责为系统上运行的任务在本机分配固定的资源，这一过程以容器的方式完成。同时，Agent(Slave)还要实时地将本机的可用资源情况报告给Master。

Agent(Slave)部分的代码主要位于src/slave目录下。

### 3.Framework

Framework是指由用户提供的外部计算框架。

Framework的scheduler模块向Master注册并申请资源，之后，Master向scheduler返回当前系统的最大可用资源。Scheduler接受后，向Master返回要执行的任务和每个任务所需要占用的资源。之后，Master再通知各个Agent分配资源给Framework的executor。

总的来说，Framework分为两部分，即scheduler和executor。Scheduler负责资源的申请、Framework内部资源的分配，Executor负责任务的执行。

### 4.Zookeeper

Zookeeper用于管理多个Master，并保证在Master宕机后启用备用Master，确保了系统的可用性。

这部分代码主要位于src/zookeeper目录下。

## 工作流程

系统中各个Agent(Slave)会实时地将本机可用的资源报告给Master。Master中维护系统可用资源的列表。当用户提交了新的任务给Framework的时候，Framework按Master提供
的资源总数，为每个任务分配一定数量的资源，并返回给Master。之后，Master分配资源并通知各个Agent(Slave)在本机分配相应数量的资源，之后启动Framework的executor模块开始执行任务。

## Spark On Mesos的运行过程

Spark On Mesos运行过程中，不再由Spark来负责资源的监视和具体的分配的工作，这一部分工作交由Mesos来完成，而其余部分包括通信等等任务仍然由Spark来完成。

运行时，Spark根据Mesos系统中可用资源的情况，为各个任务分配一定数量的资源，并将结果返回给Masos，Mesos完成资源分配之后通知Spark executor执行任务。

## Mesos与传统操作系统的对比

Mesos与传统操作系统相似的地方主要在于Mesos和传统操作系统都设计资源的管理和分配。传统操作系统是在一台机器上运行，而Mesos是将多台机器连接到一起形成一个集群，在整个集群内部进行
资源管理和分配，同时，Mesos也是要基于传统的操作系统运行的。另外，Mesos的资源分配过程是一个多层次、多步骤的过程，其中涉及Master对资源的监视、Master与Framework进行资源的协商以及Master通知Agent(Slave)
进行资源分配等等过程，而传统操作系统资源分配的过程基本上属于简单的按需分配。

## Master和Agent(Slave)的初始化过程

### Master的初始化过程

1.处理命令行参数、设置环境变量：main.cpp 150行到260行

2.Log build information：262行到271行

3.初始化libprocess库

```cpp
  if (!process::initialize(
          "master",
          READWRITE_HTTP_AUTHENTICATION_REALM,
          READONLY_HTTP_AUTHENTICATION_REALM)) {
    EXIT(EXIT_FAILURE) << "The call to `process::initialize()` in the master's "
                       << "`main()` was not the function's first invocation";
  }
```

4.初始化logging

```cpp
  logging::initialize(argv[0], flags, true); // Catch signals.
```

5.初始化防火墙规则：295行到312行

6.初始化模块、创建匿名模块：314行到350行

7.初始化Hook、Allocator、ZooKeeper等。

8.若有多个Master，竞争成为Leader：438行到461行

9.初始化Master对象并开始执行Master进程：543到561行

### Agent(Slave)的初始化过程

1.处理命令行参数、设置环境变量：main.cpp 99行到212行

2.初始化libprocess库：214行到240行

```cpp
  if (ip_discovery_command.isSome() && ip.isSome()) {
    EXIT(EXIT_FAILURE) << flags.usage(
        "Only one of `--ip` or `--ip_discovery_command` should be specified");
  }

  if (ip_discovery_command.isSome()) {
    Try<string> ipAddress = os::shell(ip_discovery_command.get());

    if (ipAddress.isError()) {
      EXIT(EXIT_FAILURE) << ipAddress.error();
    }

    os::setenv("LIBPROCESS_IP", strings::trim(ipAddress.get()));
  } else if (ip.isSome()) {
    os::setenv("LIBPROCESS_IP", ip.get());
  }

  os::setenv("LIBPROCESS_PORT", stringify(port));

  if (advertise_ip.isSome()) {
    os::setenv("LIBPROCESS_ADVERTISE_IP", advertise_ip.get());
  }

  if (advertise_port.isSome()) {
    os::setenv("LIBPROCESS_ADVERTISE_PORT", advertise_port.get());
  }
```

3.Log build information：243行到252行


4.初始化logging

```cpp
  logging::initialize(argv[0], flags, true); // Catch signals.
```

5.初始化防火墙规则

6.初始化模块、创建匿名模块

7.初始化Hook。

8.初始化Fetcher、Containerizer、MasterDetector、GarbageCollector、ResourceEstimator等。它们分别用于下载框架到本地、提供容器化的运行环境、探测网络中的Master、垃圾回收、
可用资源估计。

9.初始化Agent(Slave)对象并开始执行Agent(Slave)进程。

## Mesos的资源调度算法

Mesos默认的资源调度算法是HierarchicalDRF算法。DRF算法全称为Dominant Resource Fairness，即主导资源公平算法。这一算法的代码位于src/master/allocator/mesos/hierarchical.cpp中。

主导资源被定义为某框架所请求的资源中，相对量（即占总可用资源比例）最大的那种。

举例说明，假设系统中共有9 CPUs 和18 GB RAsM，有两个user（framework）分别运行了两种任务，分别需要的资源量为<1 CPU, 4 GB> 和 <3 CPUs, 1 GB>。对于用户A，每个task要消耗总CPU的1/9和总内存的2/9，
因而A的支配性资源为内存；对于用户B，每个task要消耗总CPU的1/3和总内存的1/18，因而B的支配性资源为CPU。DRF将均衡所有用户的支配性资源，即：A获取的资源量为：<3 CPUs，12 GB>，可运行3个task；
而B获取的资源量为<6 CPUs, 2GB>，可运行2个task，这样分配，每个用户获取了相同比例的支配性资源，即：A获取了2/3的RAMs，B获取了2/3的CPUs。（参考 http://dongxicheng.org/apache-mesos/mesos-scheduler/ ）

我认为，这种算法的优势在于能够在保证相对公平的情况下较大程度地利用系统的资源，避免了主要占用某一种资源的任务过多地占用系统资源使得其他任务无法被执行或执行效率非常低。

同时，Mesos的资源调度涉及到多级分配的问题。这导致在实际应用中可能出现在某一个节点资源未被全部分配完，然而剩下的资源过于碎片化，不足以分配给新的任务，造成效率低下。

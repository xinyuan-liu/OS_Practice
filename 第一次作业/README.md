#第一次作业

##虚拟机和容器

虚拟机：一般来讲，虚拟机指通过软件模拟的具有完整硬件系统功能的、运行在一个完全隔离环境中的完整计算机系统。虚拟机是一个用软件模拟出来的物理计算机，为上层的提供的是指令集（ISA）和IO硬件层次的接口。

容器：容器为用户程序提供了一个隔离的运行环境，容器内的程序不会影响到其他容器和宿主机的内容，为上层提供的是一个操作系统和运行环境。容器和宿主机共享一个操作系统内核，用宿主机操作系统负责调度系统资源。

二者都是一种虚拟化技术，都可以为用户提供一个隔离的运行环境。相比而言，虚拟机提供了更底层的虚拟化，支持对不同的操作系统，甚至不同的体系结构的虚拟化。而容器则更加侧重于提供一个与宿主机相似且隔离的运行环境。容器的虚拟化程度和隔离程度都不如虚拟机高，因此也能获得体积和性能上的优势。

##Spark on Mesos

部署：按mesos的官方文档编译安装mesos。下载Spark的binary并解压，按官方文档进行配置文件和环境变量的设置。这一步参考了互联网上其他一些对mesos和Spark的文档。
```bash
export MESOS_NATIVE_JAVA_LIBRARY=/usr/local/lib/libmesos.so
export SPARK_EXECUTOR_URI=/home/liuxinyuan/spark-2.1.0-bin-hadoop2.7.tgz
```
使用spark-submit将example中的JavaWordCount作为一个任务提交给mesos运行。
```bash
./bin/spark-submit \
  --class org.apache.spark.examples.JavaWordCount \
  --master mesos://127.0.0.1:5050 \
  --total-executor-cores 4 \
  file:///home/liuxinyuan/spark-2.1.0-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.1.0.jar\
  /home/liuxinyuan/spark-2.1.0-bin-hadoop2.7/test.txt
```

运行结果：
1 CPU核：2.832S
2 CPU核：3.214S
4 CPU核：4.098S


# Docker

## 1. 背景
在日常的实际开发，我们可能会遇上以下一系列的问题。
* 同一份代码或同一个可执行程序，在不同的机器下运行出不一样的结果。
* 多个服务同时部署在同一台机器下，但各服务要求对系统或环境的设置各不相同。
* 当生产环境出现故障时，只有在生产环境上最容易定位问题。
* 多个程序同时部署在同一台机器下，一个有性能问题的程序占用了所有的物理资源，导致另一程序无法正常运行。
* 当攻击者对程序漏洞进行攻击时（如get shell），会直接危害到整台机器的安全。

简而言之，Docker可以说是一个平台或者一个工具，为一个软件从开发到发布的流程提供了一致性。
用于消除配置环境差异带来的软件开发成本，因为其发布实质上是将环境与代码一起发布。
此外，还能使多个应用程序可以在相互隔离的容器中互不干扰地运行。

## 2. 核心原理
Docker-CE 目前版本组件丰富，功能复杂，本章不过多讨论。
只讨论其虚拟化的实现是通过哪些核心技术来支撑的。

### 2.1 Linux Namespace
Linux Namespace是Linux提供的一种内核级别环境隔离的方法。
从Linux 3.8开始，非特权进程可以创建用户命名空间，在这些命名空间中它们拥有完全权限，以及拥有在该命名空间下创建子空间的权限。

Linux Namespace 有如下种类：

|分类	|系统调用参数	|相关内核版本|
|-------|-------|-------------------|
|Mount namespaces	|CLONE_NEWNS	|Linux 2.4.19|
|UTS namespaces	|CLONE_NEWUTS	|Linux 2.6.19|
|IPC namespaces	|CLONE_NEWIPC	|Linux 2.6.19|
|PID namespaces	|CLONE_NEWPID	|Linux 2.6.24|
|Network namespaces	|CLONE_NEWNET	|始于Linux 2.6.24 完成于 Linux 2.6.29|
|User namespaces	|CLONE_NEWUSER	|始于 Linux 2.6.23 完成于 Linux 3.8|

要使用Linux Namespace, 主要通过调用以下3个内核API：
* clone() – 实现线程的系统调用，用来创建一个新的进程，并可以通过设计上述参数达到隔离
* unshare() – 使某进程脱离某个namespace
* setns() – 把某进程加入到某个namespace

TODO: 以下文本未校对，可能出现较严重的语法错误                        
#### 2.1.1 Mount namespaces（CLONE_NEWNS，Linux 2.4.19）
隔离了一组进程看到的文件系统挂载点集。因此，不同安装命名空间中的进程可以具有文件系统层次结构的不同视图。通过添加mount命名空间，mount（） 和umount（） 系统调用将停止在系统上所有进程可见的全局挂载点集上运行，而是执行仅影响与调用进程关联的挂载命名空间的操作。

mount命名空间的一个用途是创建类似于chroot jails的环境。但是，与使用chroot（）系统调用相比，mount命名空间是一个更安全，更灵活的工具。mount命名空间的其他更复杂的用法也是可能的。例如，可以在主从关系中设置单独的安装命名空间，以便挂载事件自动从一个命名空间传播到另一个命名空间; 例如，这允许安装在一个命名空间中的光盘设备自动出现在其他命名空间中。

Mount命名空间是第一种在Linux上实现的命名空间，出现在2002年。这个事实说明了相当通用的“NEWNS”名字对象（“新命名空间”的缩写）：当时似乎没有人在想其他，将来可能需要不同类型的命名空间。

#### 2.1.2 UTS namespaces（CLONE_NEWUTS，Linux 2.6.19）
通过uname（） 系统调用隔离两个系统标识符 - nodename和 domainname -returned ; 使用sethostname（）和 setdomainname（）系统调用设置名称。在容器的上下文中，UTS名称空间功能允许每个容器具有自己的主机名和NIS域名。这对于基于这些名称定制其操作的初始化和配置脚本非常有用。术语“UTS”派生自传递给uname（）系统调用的结构的名称 ：struct utsname。该结构的名称又来自“UNIX分时系统”。

#### 2.1.3 IPC namespaces（CLONE_NEWIPC，Linux 2.6.19）
隔离了某些进程间通信（IPC）资源，即System V IPC对象和（自Linux 2.6.30）POSIX消息队列。这些IPC机制的共同特征是IPC对象由文件系统路径名以外的机制识别。每个IPC名称空间都有自己的一组System V IPC标识符和自己的POSIX消息队列文件系统。

#### 2.1.4 PID namespaces（CLONE_NEWPID，Linux 2.6.24）
隔离进程ID号空间。换句话说，不同PID名称空间中的进程可以具有相同的PID。PID命名空间的主要优点之一是容器可以在主机之间迁移，同时为容器内的进程保留相同的进程ID。PID命名空间还允许每个容器具有自己的 init（PID 1），即管理各种系统初始化任务的“所有进程的祖先”，并在终止时收获孤立的子进程。

从特定PID名称空间实例的角度来看，进程有两个PID：名称空间内的PID，以及主机系统上名称空间外的PID。PID命名空间可以嵌套：一个进程将为层次结构的每个层提供一个PID，从它所驻留的PID命名空间开始到根PID命名空间。进程可以看到（例如，通过/ proc / PID查看并使用kill（）发送信号）仅包含在其自己的PID名称空间中的进程以及嵌套在该PID名称空间下方的名称空间。

#### 2.1.5 Network namespaces （CLONE_NEWNET，在Linux 2.4.19 2.6.24中启动，主要由Linux 2.6.29完成）
提供与网络相关的系统资源的隔离。因此，每个网络命名空间都有自己的网络设备，IP地址，IP路由表，/ proc / net目录，端口号等。

网络命名空间从网络角度使容器变得有用：每个容器可以拥有自己的（虚拟）网络设备和自己的应用程序，这些应用程序绑定到每个命名空间的端口号空间; 主机系统中合适的路由规则可以将网络分组引导到与特定容器相关联的网络设备。因此，例如，可以在同一主机系统上具有多个容器化Web服务器，每个服务器在其（每个容器）网络命名空间中绑定到端口80。

#### 2.1.6 User namespaces （CLONE_NEWUSER，在Linux 2.6.23中启动，在Linux 3.8中完成）
隔离用户和组ID号空间。换句话说，进程的用户和组ID在用户命名空间的内部和外部可以是不同的。这里最有趣的情况是进程可以在用户命名空间外具有普通的非特权用户ID，同时在命名空间内具有用户ID 0。这意味着该进程对用户命名空间内的操作具有完全root权限，但对于命名空间外的操作没有特权。


#### 2.1.7 命名空间使用Demo：执行一个在指定命名空间下的bash
TODO: demo代码编写及执行

### 2.2 CGroups

### 2.3 Union Filesystem

### 2.4 简单总结
Linux Namespace: 用于实现操作系统级别资源的隔离，如主机名、域名、PID、文件系统等。

CGroups: 物理资源的隔离, 如CPU、硬盘、内存等。

Union Filesystem: 镜像文件的合并。

## 3. Docker架构
https://docs.docker.com/engine/docker-overview/

## 4. 极简使用指南

## 5. 与其他虚拟化技术的比较

## Reference
* Docker核心技术 https://draveness.me/docker
* Docker official documentation https://docs.docker.com
* Docker容器与容器云，浙江大学SEL实验室著
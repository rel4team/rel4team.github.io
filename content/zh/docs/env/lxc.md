---
title: "lxd 安装环境"
weight: 1
# bookToc: true
# bookHidden: false
# bookComments: false
---

>参考资料
>1. [https://documentation.ubuntu.com/lxd/latest/tutorial/first_steps/](https://documentation.ubuntu.com/lxd/latest/tutorial/first_steps/)
>2. [https://documentation.ubuntu.com/server/how-to/containers/lxd-containers/](https://documentation.ubuntu.com/server/how-to/containers/lxd-containers/)
## lxd 和 lxc

LXC(Linux Container) 是一个基于 linux 的容器技术。它利用 linux 的 cgroup 和 namespace 技术来在一台电脑上运行多个不同的 linux 系统。容器借助宿主机的 Linux 内核运行。

LXD(Linux Container Daemon) 是一个开源的容器管理工具，它提供了丰富的指令来管理 Linux 容器。


LXC 的优点：

* 灵活度高：可以在同一台电脑上运行不同的 linux 系统，比如 ubuntu 上运行 fedora、alpine、archlinux

* 资源占用率少：lxc 基于 cgroup 和 namespace 实现，利用宿主机内核，所以不需要虚拟内核。所以占用内存少且性能好。


lxc 的优点是可以直接虚拟出一个完整的操作系统实例，系统可以持久化存储。


lxc 目前的可以正常访问，不需要很依赖科学上网。


**lxc 中的容器被称为 instance，下面的案例中同样使用 instance 来介绍容器。**



## 安装 lxd

下面介绍使用 ubuntu2404 来安装 lxd，并且初始化设备。也可以根据参考资料 1 中描述的信息来安装。

### 安装 lxd 环境

```shell
sudo snap install lxd
```
### 初始化 lxd

在安装结束后，需要初始化本机的 lxc 服务（**如果只是连接到远程的 lxc 服务，在出现选项时使用 Ctrl + C 退出即可，需要 lxd 产生特定的证书文件**）

```shell
sudo lxd init
```
### 配置 lxd 权限

默认情况下，执行 lxc 操作系统使用 root 用户，如果不想频繁的执行 sudo 指令，那么可以将当前用户加入到 lxd 用户组中，那么当前用户就拥有了 lxc 的控制权限。后续可以免去频繁使用 sudo 指令来执行 lxc 的烦恼。 

```shell
adduser $(whoami) lxd
```


### 配置远程连接（Optional）

默认情况下，在配置完 lxd 后会在本机的 8443 端口开启一个远程服务，其他的电脑可以连接当前本机的 lxd 服务来操控或连接 lxc 服务。


在执行过上面描述的 init 后（即使是 Ctrl + C）会在  `~/snap/lxd/common/config/client.crt` 生成证书（仅限 snap 安装的 lxd, 如果是其他的系统或者安装方式，需要自行查找证书文件的位置），将证书传输到服务器中进行信任。


#### 在 server 中

```shell
# 信任证书
lxc config trust add client.crt
```
#### 在 client 电脑上

```shell
# 在 client 上连接远程服务
lxc remote add <remote_name> <remote_ip>

# 将特定的 <remote> 变为默认 remote
lxc remote switch <remote_name>
```
lxc 进行操作的时候如果没有选择 remote 的时候就会使用默认的 remote，默认的 remote 是  `local` ，比如下面的  `lxc image list ubuntu:` 这里的  `ubuntu` 就是  `remote` 之一，也可以使用  `lxc image list local` 来查看本机已经下载的镜像。

可以利用  `lxc remote list` 来查看当前电脑所有的 remote。


## 创建新的 instance 

### 查看可以使用的镜像列表

```shell
# 查看 ubuntu 提供的可以使用的云镜像
lxc image list ubuntu:
# 查看 canonical 提供的镜像
lxc image list images:

# 如果上述的镜像都不能满足需求，可以利用下面的指令来获取可以用的镜像仓库
lxc remote list
```
### 使用一个镜像创建一个 Instance 

利用 lxc launch 指令创建一个新的 instance，这里创建了一个新的 ubuntu，版本是 ubuntu22.04，instance 名称是 rel4-test-env

```shell
lxc launch ubuntu:22.04 rel4-test-env
```
## 运行指令

在创建新的 instance 之后可以利用 lxc exec <instance> -- <cmd> 来执行命令，这里会直接获取到 instance 的最高权限，可以在容器初期完成一些初始化，或者执行一些权限指令。不过在 instance 中默认附带了一个名为 ubuntu 的账户(uid = 1000, gid = 1000)。可以利用下面几条指令登录。

### lxc exec 方式（推荐）

```shell
# 作为 root 用户登录，没有标准的初始化流程，没有用户环境变量初始化
lxc exec rel4-test-env -- bash
# 作为 root 用户登录，有初始化流程，但是会保持在 /root 文件夹下
# 这里的 su root，也可以更换为 su ubuntu, 不过并不会更改到 /home/ubuntu
lxc exec rel4-test-env -- su root
# 利用标准的账户登录流程登录，执行标准的初始化流程，切换到指定用户的 home 目录下
lxc exec rel4-test-env -- login -f root

# 下面使用推荐的 login -f 方式登录 ubuntu 用户
lxc exec rel4-test-env -- login -f ubuntu
```
### lxc shell 方式

lxc shell 是 lxc exec 的封装，默认登录 root 用户

```shell
lxc shell rel4-test-env
```


### lxc console 方式

lxc console 类似于使用串口连接了此电脑，需要在 console 连接后输入 用户名和密码登录，但是默认情况下账号和密码未知，且需要手动输入 <ctrl>+a q 退出。

```shell
lxc console rel4-test-env
```


## 宿主机与 instance 共享

lxc 中的 instance 支持共享宿主机的资源，可以利用 lxc config device add 来添加需要共享的资源。下面将用几个案例来描述如何共享宿主机的资源。

### 共享文件夹

instance 可以和宿主机共享一个文件夹，下面使用一个案例来共享宿主机的文件夹，这里为一个叫做 rel4-test-env 的 instance 添加了一个名为 sharedir 的 disk 设备。在 instance 中可以访问 /home/ubuntu/test-mnt 文件夹来访问宿主机中 /tmp/instance-share-dir 文件夹下的内容。


**这里的** `shift=true`**表示将文件夹在宿主机下的权限和 instance 中的权限进行映射，方便文件在宿主机和 guest 中访问的时候都可以保持权限，不会出现某一方需要 sudo 的情况, 不过还是推荐在宿主机和 instance 中都使用 uid=1000, gid=1000 的账户。instance 中为名为 ubuntu 的用户，在宿主机中一般为手动创建的第一个用户。**

```shell
lxc config device add rel4-test-env sharedir disk \
  source=/tmp/instance-share-dir \
  path=/home/ubuntu/test-mnt \
  shift=true
```
### 共享 disk 设备

如果需要将宿主机中插入的真实设备直通给 instance，也可以使用上述共享文件夹的方式进行共享的方式进行。下面将宿主机中 /dev/sda1 直通给 instance，并 mount 到 instance 中的 /home/ubuntu/test-mnt-sda1。


```shell
lxc config device add rel4-test-env test-mnt-sda1 disk \
  source=/dev/sda1 \
  path=/home/ubuntu/test-mnt-sda1 \
  shift=true
 
```


### 将 instance 的端口映射到宿主机上

在一些情况下，会需要将 instance 中的端口暴露出来，比如需要直接使用 ssh 来接 instance，这个时候需要将 instance 中的 22 端口映射到宿主机中。比如 instance 中开启了网页服务或者数据库服务，就需要将特定的端口映射到宿主机中。下面利用一个案例将 instance 中的 22 端口映射到宿主机的 5522 端口上，方面从外面进行连接。创建的设备名称为 ssh，设备类型为 proxy。


**这里的 listen 指的是宿主机中的监听范围和端口，0.0.0.0 表示监听所有来源，如果只想让本机进行访问可以填写为 127.0.0.1（这里是计算机网络的知识，可以去网络搜索获得更相信的信息。），connect 为 instance 中的监听范围和端口，一般情况下使用 127.0.0.1 即可。**

```shell
lxc config device add rel4-test-env ssh proxy \
  listen=tcp:0.0.0.0:5522 \
  connect=tcp:127.0.0.1:22
```


### 直通网卡

上述中描述的情况是将 instance 中的端口映射到宿主机上，如果需要 instance 可以直接利用宿主机中特定的网卡来进行通讯，可以将宿主机的网卡直接映射到 instance 中。


下面将宿主机中的 enp7s0 网卡直通给 instance，设备类型为 nic，nic类型为物理设备，设备名称为 eth0。


**这种方式会导致宿主机丢失对于网卡的访问，如果只有一张网卡不建议使用这种方式。更建议使用 macvlan 的方式。**

```shell
lxc config device add rel4-test-env eth0 nic \
  nictype=physical \
  parent=enp7s0
```


### 在 lxd 中 mount

在 lxc 的 instancce 中需要进行 mount 的时候会提示 mount 没有权限，这是因为 lxc 对于 mount 等敏感的操作具有一定的限制，如果想要在 lxc 的 instace 中进行 mount 操作，需要给 lxc instace 对应的权限，并且把宿主机的 loop 文件映射给宿主机。


```plain
# 设置 instance 权限并重启
lxc config set <instance> security.privileged true
lxc config set <instance> security.syscalls.intercept.mount true
lxc restart <instace>

# 映射 loop 文件(如果提示了没有 loop，可以映射对应的 loop 文件)
lxc config device add <instance> loop-control unix-char path=/dev/loop-control
lxc config device add <instance> loop0 unix-block path=/dev/loop0
lxc config device add <instance> loop1 unix-block path=/dev/loop1
lxc config device add <instance> loop2 unix-block path=/dev/loop2
```


## 传输文件到 lxc 中

如果不想共享整个文件夹，但是又想将一个文件（夹）传输到 instance 中，那么可以利用 lxc file push 来将文件从 host 传输到 instance 中

```shell
lxc file push <host_file> <instance_name>/<instance_file>
```


同理 也可以使用 lxc file pull 来将文件从 lxc 中拉取出来


```shell
lxc file pull <instance_name>/<instance_file> <host_file>
```


lxc 还可以利用 lxc file edit 来编辑 instance 中的文件


```shell
lxc file edit <instance_name>/<instance_file>
```



## 安装 rel4 开发环境

此时已经拥有了一个完整的操作系统，可以参考本地安装的文档安装环境。
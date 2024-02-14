# Docker简介

Docker 是一个开源的应用容器引擎，基于 Go 语言 并遵从 Apache2.0 协议开源。

Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。

Docker的应用场景：

* Web 应用的自动化打包和发布。

* 自动化测试和持续集成、发布。

* 在服务型环境中部署和调整数据库或其他的后台应用。

Docker的优点：

* 快速，一致地交付应用程序
* 响应式部署和扩展

* 在同一硬件上运行更多工作负载

## 虚拟化

以单个物理硬件系统为基础创建多个模拟环境或专用资源。

称为"Hypervisor" （虚拟机监控程序）的软件可直接连接到硬件，从而将一个系统划分为不同的、单独安全环境，即虚拟机（VM）。

虚拟机监控程序能够将计算机资源与硬件分离并适当分配资源

配备了虚拟机监控程序的物理硬件叫做"主机"，而使用其资源的虚拟机则被称为虚拟客户机。

这些虚拟客户机将计算资源（如 CPU、内存和存储器）视为一组可进行重新分配的资源。操作员可以控制 CPU、内存、存储器和其他资源的虚拟实例，以便虚拟客户机能在需要时收到所需资源。

## Docker架构

Docker 包括三个基本概念:

* 镜像（Image）：Docker 镜像（Image）由多个层组成，每层叠加之后，从外部看来就如一个独立的对象。镜像内部是一个精简的操作系统（OS），同时还包含应用运行所必须的文件和依赖包。

* 容器（Container）：镜像是静态的定义，容器是镜像运行时的实体。

    容器可以被创建、启动、停止、删除、暂停等。

    * 交互式容器：退出交互会话，容器也随之退出
    * 守护式容器：容器可以在后台持续运行

* 仓库（Repository）：保存镜像。

Docker 使用客户端-服务器 (C/S) 架构模式，使用远程API来管理和创建Docker容器。

# Docker安装和启动

安装：

~~~bash
# 安装docker
yum install docker
# 确认版本
docker -v
~~~

systemctl命令是系统服务管理器指令，它是 service 和 chkconfig 两个命令组合。

~~~bash
# 启动
systemctl start docker
# 停止
systemctl stop docker
# 重启
systemctl restart docker
# 查看状态
systemctl status docker
# 开机启动
systemctl enable docker
~~~

辅助信息

~~~bash
#查看docker概要信息
docker info
#查看docker帮助文档
docker –-help
~~~



# Docker镜像操作

**查看镜像**

~~~bash
docker images
~~~

* REPOSITORY：镜像所在的仓库名称
* TAG：镜像标签
* IMAGE ID：镜像ID
* CREATED：镜像的创建日期（不是获取该镜像的日期）
* SIZE：镜像大小

**搜索镜像**

~~~bash
docker search imageName
~~~

**配置镜像下载加速**

~~~bash
#镜像下载加速 添加下面代码块内容
vi /etc/docker/daemon.json
#使配置文件生效
systemctl daemon-reload
#重启docker
systemctl restart docker
~~~

~~~json
{
  "registry-mirrors": ["https://9cpn8tt6.mirror.aliyuncs.com"]
 }
~~~

**拉取镜像**

~~~bash
docker pull 镜像名称
~~~

**删除镜像**

~~~bash
# 根据IMAGE ID 删除
docker rmi  ImageID
# 删除所有镜像
docker rmi `docker images -q`
~~~

# Docker容器操作

**查看容器**

~~~bash
# 查看所有容器
docker ps -a
# 查看最后一次运行的容器
docker ps -l
# 查看停止的容器
docker ps -f status=exited
~~~

**创建容器**

~~~bash
docker run  [参数] <imageName ：tag>  [命令]  : 根据镜像创建容器
参数说明:
-i:表示创建并运行容器
-t:表示容器启动后会进入其命令行。加入这两个参数后，容器创建就能登录进去。即分配一个伪终端。
--name :为创建的容器命名。
-v:表示目录映射关系（前者是宿主机目录，后者是映射到宿主机上的目录,中间用冒号隔开），可以使用多个－v做多个目录或文件映射。注意：最好做目录映射，在宿主机上做修改，然后共享到容器上。
-d:在run后面加上-d参数,则会创建一个守护式容器在后台运行（这样创建容器后不会自动登录容器，如果只加-i -t两个参数，创建后就会自动进去容器）。
-p:表示端口映射，前者是宿主机端口，后者是容器内的映射端口。可以使用多个－p做多个端口映射
~~~

**删除容器**

~~~bash
# 根据容器ID或name删除容器
docker rm <containerName|containerID>
# 删除所有容器
docker rm `docker ps -a -q`
~~~

只能删除停止的容器

**开启关闭容器(守护容器)**

~~~bash
docker start <containerName>
docker stop <containerName>
~~~

**进入退出容器**

~~~bash
# 进入容器
docker exec -it <containerName> /bin/bash
# 退出容器 在容器的伪终端输入
exit
~~~

**文件拷贝**

~~~bash
# path1是容器中的文件路径 path2是宿主机文件路径
docker cp containerName:path1  path2
docker cp path2  containerName:path1
~~~

注意：容器与容器之间不可以相互拷贝文件

**目录映射**

在创建容器时：

~~~bash
docker run -di –-name=新容器名称 -v /宿主机路径:/新容器路径 镜像名称
# 示例：
docker run -di --name=mycentos3 -v /usr/local/han:/usr/local/qing centos
~~~

注意：如果映射过去的不是一个文件，而是含有多级的文件夹，那么在容器中打开这个文件夹可能存在权限不足的情况

这是因为CentOS7中的安全模块selinux把权限禁掉了，我们需要添加参数 –privileged=true 来解决挂载的目录没有权限的问题

~~~bash
docker run -di --name=mycentos3 -v /usr/local/han:/usr/local/qing --privileged=true centos
~~~

**查看容器IP**

~~~bash
# 查看容器的ip信息
docker inspect <containerName>
# 只查看容器的ip地址
docker inspect --format='{{.NetworkSettings.IPAddress}}' <containerName>
~~~



# 部署应用

**MySQL**

~~~bash
# 下载镜像
docker pull mysql
# 创建容器
docker run -di --name mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
-p 代表端口映射，格式为 宿主机映射端口:容器运行端口
-e 代表添加环境变量 MYSQL_ROOT_PASSWORD是root用户的登陆密码
~~~

 **Tomcat**

~~~bash
docker pull tomcat

docker run -di --name=tomcat -p 9000:8080 -v /usr/local/han:/usr/local/tomcat/webapps --privileged=true tomcat
~~~

**Nginx**

~~~bash
docker pull nginx
docker run -di --name=nginx -p 80:80 nginx /bin/bash
~~~

**Redis**

~~~bash
docker pull redis
docker run -di --name=redis -p 6379:6379 redis
~~~




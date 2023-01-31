# 分布式文件系统

分布式文件系统通过计算机网络统一管理系统下节点服务器的的资源

并对外提供统一的资源访问接口

相比传统的单机文件管理，它具有以下的优势：

* 可扩展：扩展只需要将新的节点服务器并入系统
* 高可用：在分布式文件系统中，高可用性包含两层，一是整个文件系统的可用性，二是数据的完整和一致性

* 低成本：分布式存储系统的自动容错和自动负载平衡允许在成本较低服务器上构建分布式存储系统。此外，线性可扩展性还能够增加和降低服务器的成本。

* 弹性存储： 可以根据业务需要灵活地增加或缩减数据存储以及增删存储池中的资源，而不需要中断系统运行

主流的分布式文件存储系统：

GFS、HDFS、Ceph、Lustre、MogileFS、MooseFS、FastDFS、TFS、GridFS等。

# FastDFS简介

FastDFS是一个开源的轻量级分布式文件系统，为互联网应用量身定做，简单、灵活、高效，采用C语言开发，由阿里巴巴开发并开源。

FastDFS对文件进行管理，功能包括：文件存储、文件同步、文件访问（文件上传、文件下载、文件删除）等，解决了大容量文件存储的问题，特别适合以文件为载体的在线服务，如相册网站、文档网站、图片网站、视频网站等等。

FastDFS充分考虑了冗余备份、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

## 架构

* Client客户端来访问文件管理系统，FastDFS提供了专用的API来访问FastDFS系统

* Tracker集群中所有tracker地位平等，功能相同。负责管理所有的storage节点，记录storage节点服务器的存储空间,状态,负载等。

    客户端请求Tracker进行文件上传、下载

    Tracker通过调度Storage完成文件上传、下载

* storage节点负责保存文件。所有的storage节点分为不同的组，不同组的storage保存不同的内容，实现纵向扩容。同一个组的所有storage节点都保存相同的内容，实现横向扩容（备份），同一个组下的所有storage节点地位平等，没有主从之分。

![FastDFS架构](https://gitee.com/wangziming707/note-pic/raw/master/img/FastDFS%E6%9E%B6%E6%9E%84.png)

## 文件上传下载流程

* 文件上传
    1. Storage Server 定时向 Tracker Server 发送自己的存储信息
    2. Client 调用 FastDFS API 发送上传连接请求给 Tracker Server
    3. Tracker Server 查询可用的 Storage Server 信息
    4. ’Tracker Server 将该信息( Storage 的 ip 和端口号 )返回给 Client 
    5. Client 调用 FastDFS API 上传文件给 Storage Server
    6. Storage Server 生成文件 id ( file_id )
    7. Storage Server 将上传内容写入磁盘
    8. Storage Server 返回 file_id  给 Client 
    9. Client 层将存储文件信息写入数据库

![FastDFS文件上传流程](https://gitee.com/wangziming707/note-pic/raw/master/img/FastDFS%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0%E6%B5%81%E7%A8%8B.png)

* 文件下载
    1. Storage Server 定时向 Tracker Server 发送自己的存储信息
    2. Client 调用 FastDFS API 发送下载连接请求给 Tracker Server
    3. Tracker Server 查询可用的 Storage Server 信息( 检验同步状态 )
    4. ’Tracker Server 将该信息( Storage 的 ip 和端口号 )返回给 Client 
    5. Client 调用 FastDFS API 发送 file_id( 组名、路劲、文件名 )  给 Storage Server
    6. Storage Server 通过 file_id 查找文件
    7. Storage Server 将 查找到的文件( file_content ) 返回给 Client

![FastDFS文件下载流程](https://gitee.com/wangziming707/note-pic/raw/master/img/FastDFS%E6%96%87%E4%BB%B6%E4%B8%8B%E8%BD%BD%E6%B5%81%E7%A8%8B.png)

# 安装

## 依赖

~~~bash
yum install -y gcc gcc-c++

yum -y install libevent

yum groupinstall "Development Tools" "Server platform Development"

yum install pcre-devel zlib zlib-devel openssl openssl-devel
~~~

## Iibfastcommon

libfastcommon是FastDFS官方提供的，libfastcommon包含了FastDFS运行所需要的一些基础库

~~~bash

# 解压安装包
tar -zxvf libfastcommonV1.0.7.tar.gz -C /usr/local/
# 编译并安装
cd /usr/local/libfastcommon-1.0.7/
./make.sh
./make.sh install

#拷贝libfastcommon.so 到/usr/lib

cp /usr/lib64/libfastcommon.so /usr/lib

~~~

## FastDFS

~~~bash
# 解压
tar -zxvf FastDFS_5.05.tar.gz -C /usr/local
# 安装编译安装
cd /usr/local/FastDFS/
./make.sh && ./make.sh install
#安装目录下的conf下的文件拷贝到/etc/fdfs/下
cp /usr/local/FastDFS/conf/* /etc/fdfs/
~~~

# 配置启动

## tracker

拷贝配置模板：

~~~bash
# /etc/fdfs目录下
cp tracker.conf.sample tracker.conf
~~~

配置：

~~~properties
# 需要配置的属性

# home地址
base_path=/home/fastdfs/tracker
~~~

创建配置的地址：

~~~bash
mkdir -p /home/fastdfs/tracker
~~~

启动tracker服务：

~~~bash
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
~~~

## storage

拷贝配置模板：

~~~bash
# /etc/fdfs目录下
cp storage.conf.sample storage.conf
~~~

配置：

~~~properties
# 需要配置的属性

# home地址
base_path=/home/fastdfs/storage
# 文件存放地址
store_path0=/home/fastdfs/storage/files
# tracker服务ip地址
tracker_server=192.168.134.128:22122
~~~

创建配置的地址：

~~~bash
mkdir -p /home/fastdfs/storage
mkdir /home/fastdfs/storage/files
~~~

启动storage服务

~~~bash
/usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart
~~~

## 测试

 使用FastDFS自带工具测试

~~~bash
#/etc/fdfs/ 目录下拷贝一份新的client配置文件
cp client.conf.sample client.conf
~~~

配置：

~~~properties
base_path=/home/fastdfs/client
tracker_server=192.168.134.128:22122
~~~

创建配置的文件夹：

~~~bash
mkdir /home/fastdfs/client
~~~

测试：

~~~bash	
/usr/bin/fdfs_test /etc/fdfs/client.conf upload /usr/local/share/123.jpg
~~~

## 配置开机启动

* 编辑/etc/rc.d/rc.local 文件

~~~ properties
# fastdfs start
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
/usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart
# nginx start
/usr/local/nginx/sbin/nginx
~~~

*  给rc.local 文件增加可执行的权限

~~~bash
chmod +x /etc/rc.d/rc.local
~~~



# FastDFS整合Nginx

## 架构

![FastDFS整合Nginx架构图](https://gitee.com/wangziming707/note-pic/raw/master/img/FastDFS%E6%95%B4%E5%90%88Nginx%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

## 安装整合

* 解压

~~~bash
tar -zxvf fastdfs-nginx-module_v1.16.tar.gz -C /usr/local
~~~

*   修改文件目录 src/config文件，将文件中的所有 /usr/local/ 路径改为 /usr/

* 将fastdfs-nginx-module/src下的mod_fastdfs.conf拷贝至/etc/fdfs/下

~~~bash
cp mod_fastdfs.conf /etc/fdfs/
~~~

* 修改mod_fastdfs.conf

~~~properties
base_path=/home/fastdfs/nginx_mod
# 如果有多个tracker，全部写上
tracker_server=192.168.134.128:22122
url_have_group_name = true
store_path0=/home/fastdfs/storage/files
~~~

* 创建配置的文件夹

~~~bash
mkdir /home/fastdfs/nginx_mod
~~~

* 为nginx添加module

~~~bash
# 在nginx解压包文件夹中：
./configure \
--prefix=/usr/local/nginx \
--add-module=/usr/local/fastdfs-nginx-module/src
# 编译 
make && make install
~~~

* 拷贝配置文件到 /etc/fdfs   

~~~bash
# 进入cd /usr/local/fastdfs-5.05/conf/路径下
cp http.conf mime.types /etc/fdfs/
~~~

* 修改nginx配置文件

~~~nginx
location ~ /group[1-9]/M0[0-9]{
    ngx_fastdfs_module;
}
~~~

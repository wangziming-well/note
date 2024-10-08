# Linux简介

由于系统的稳定性和安全性，linux系统被称为程序代码运行的最佳操作系统环境

为了安装和使用liunx系统，我们需要安装：

* VMware：虚拟机在windows操作系统下为Linux系统运行提供环境
* Xshell：远程操作Linux虚拟机系统
* Xftp：远程操作Linux系统文件

# Linux命令

## 基础命令

* `ls`显示指定目录下的所有文件或目录，如果未指定目录，则为当前目录
    * `-a`显示所有文件目录，包括隐藏文件目录
    * `-l`显示详细信息，相当于命令`ll`
* `cd`切换到指定目录，不能操作文件
* `pwd`查看当前目录结构
* `ps -ef`查看进程信息
* `whoami`显示当前用户名称
* `clear`清屏
* `reboot` `init 6`重启
* `shutdown now` `init 0`关机
* `man 命令`查看命令的详情
* `date`查看日期

## 文件\目录

* `touch`创建文件，如果文件名中有`space`文件名需要加上单引号或者双引号
* `mkdir`创建目录，如果目录名中有`space`目录名需要加上单引号或者双引号

* `rm`删除文件或目录
    * 直接跟文件名，删除文件
    * `-f`不给提示直接删除
    * `-d`删除目录
    * `-rf`强制删除目录，不带任何提示
* `cp`
    * `cp 文件名 路径`将指定文件复制到指定路径下
    * `cp -r 目录名 路径`将指定目录复制到指定路径下
* `mv 路径1 路径2`将路径1指定的文件或目录移动到路径2指定的文件或目录，并重命名
* `find 目录 文件|目录`查找指定目录下的文件或目录

* `chmod u|g|o +|-|= r|w|x 文件|目录`设置指定文件或路径的权限

* 权限：Linux中文件权限有10个字段：
    * 第一个字段 
        * `-`代表文件
        * `d`代表目录
    * 第二三四个字段(u)
        * `-`表示没有权限
        * `x`读
        * `r`写
        * `w`可执行
    * 第五六七个字段(g)
        * `-`表示没有权限
        * `x`读
        * `r`写
        * `w`可执行
    * 第八九十个字段(o)
        * `-`表示没有权限
        * `x`读
        * `r`写
        * `w`可执行

| #    | 权限           | rwx  | 二进制 |
| :--- | :------------- | :--- | :----- |
| 7    | 读 + 写 + 执行 | rwx  | 111    |
| 6    | 读 + 写        | rw-  | 110    |
| 5    | 读 + 执行      | r-x  | 101    |
| 4    | 只读           | r--  | 100    |
| 3    | 写 + 执行      | -wx  | 011    |
| 2    | 只写           | -w-  | 010    |
| 1    | 只执行         | --x  | 001    |
| 0    | 无             | ---  | 000    |

## 文件内容

* `cat`查看文件内容 只能操作文件
* `more`分页查看文件中的数据信息
    * `enter`一行一行查看剩下的文件内容
    * `space`一页一页查看剩下的文件内容
*  `head [-number] 文件名`查看文件的前指定行，默认前10行
* `tail [-number] 文件名`查看文件的最后指定行，默认最后10行
* `vi/vim 文件名`编辑指定文件
    * `i`进入编辑模式
    * `Esc`最后行模式
        * `:q!`退出不保存
        * `:wq!`退出并保存

* `> 文件名`清空文件
  
* `grep `进行过滤
    * `grep '查找内容' 指定文件`查找指定文件，将所有查找内容所在行显示
* `cat 文件名 | grep 查找内容 |grep 查找内容`可以进行多次过滤

## 用户

* `su 用户名`切换用户

* `useradd 用户名`添加用户
* `passwd 用户名`为新增的用户添加密码
* `userdel 用户名`删除用户
* `groupadd 用户组名`创建一个用户组
* `usermod -G 用户组名 用户名`将制定用户添加到指定用户组
* `groupdel 用户组名`删除指定用户组



# 防火墙

* `systemctl status firewalld.service`查看防火墙状态
* `systemctl stop firewalld.service`关闭防火墙
* `systemctl disable firewalld.service`永久关闭防火墙



## 压缩\解压缩

* `gzip 文件名`压缩指定文件
* `tar`打包文件或目录（保留源文件），并提供压缩解压缩
    * `-c`产生.tar 的打包文件
    * `-v`显示压缩过程信息
    * `-f`指定压缩后的压缩文件名
    * `z`打包的同时压缩
    * `x`解压.tar.gz压缩文件
    * `-C`指定解压到指定目录



## 软件包管理

### RPM

* `rpm -qa`查看所有安装的软件包
* `rpm -e`卸载
* `rpm -ivh`安装

### yum

* `yum list`查看所有安装的包
* `yum remove`卸载
* `yum install`安装






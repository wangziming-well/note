# Git简介

是一个分布式版本管理工具

追踪目录（文件夹）和文件的修改历史



# 基础命令

~~~bash
#设置全局用户
git config --global user.name "用户名"
git config --global user.email "电子邮件"

#建立仓库,在指定的目录文件夹下执行该命令，将建立该文件夹的仓库
git init

#将文件添加到暂存区
git add 
{ .    						#添加当前目录下的所有文件
| dir1[,dir2.. ]   			#添加指定目录包括子目录
| fileName1 [,fileName2..]} #添加指定文件

# 将暂存区指定文件提交到仓库，如果不指定文件，提交暂存区所有文件
# -a 参数设置修改文件后不需要执行 git add 命令，直接来提交
git commit [file1] [file2]...   -am '提交描述' 

# 查看仓库提交记录
git log

# 查看差异
git diff <filename>

# 回退版本
#mixed 为默认，可以不用带该参数，用于重置暂存区的文件与上一次的提交(commit)保持一致，工作区文件内容保持不变。
# hard 参数撤销工作区中所有未提交的修改内容，将暂存区与工作区都回到上一次版本，并删除之前的所有信息提
# soft 参数用于回退到某个版本
git reset 
[--soft | --mixed | --hard] 
[HEAD]
# HEAD可以填commit版本号，或者版本号后几位，或者用下面符号：
- HEAD 表示当前版本
- HEAD^ 上一个版本
- HEAD^^ 上上一个版本
- HEAD^^^ 上上上一个版本
- 以此类推...

- HEAD~0 表示当前版本
- HEAD~1 上一个版本
- HEAD^2 上上一个版本
- HEAD^3 上上上一个版本
- 以此类推...



# 查看历史操作记录
git reflog
~~~

# 分支管理

~~~bash
# 创建分支
git branch <branchname>

# 删除分支
git branch -d <branchname>

# 切换分支
git checkout <branchname>

# 创建并切换分支
git checkout -b <branchname>

# 合并分支 当前分支与指定分支，如果为指定，默认如主分支
git merge <branchname>

# 查看所有分支
git branch

~~~

## 合并冲突

创建分支后，当前分支和主分支在同一文件的同一地方都做了修改

此时合并当前分支会出现合并冲突，分支进入(master|MERGING)提示正在合并

Git会将冲突的地方标记在文件中，此时根据需要进行修改

修改完成后提交该文件,告诉Git冲突解决，git会用此时提交的文件作为合并后的文件

~~~bash
git add fileName
git commit
~~~



# 远程仓库

~~~bash
# 查看所有的远程方库
git remote [-v]

# 添加一个远程仓库
git remote add <shortname> <url>

# 删除远程仓库
git remote rm [别名]

# 复制远程仓库到本地
git clone <repository> <directory>

#远程仓库下载新分支与数据
#执行完后需要执行git merge 合并远程分支到你所在的分支。
git fetch <remote>

#推送当前分支到远程仓库并合并到指定远程分支(不需要手动merge)
# 如果本地分支和要指定的远程分支名称相同，branch可以省略
git push <remote> <branch>

# 拉取远程仓库的指定分支到本地当前分支并合并
git push <remote> <branch>
~~~

**注意：**在执行pull之后，进行下一次push之前，如果其他人进行了推送内容到远程数据库的话，那么你的push将被拒绝。此时需要先再pull，在尝试push










# Git简介

是一个分布式版本管理工具

追踪目录（文件夹）和文件的修改历史

## 工作区域

git分为三个工作区域：

* 工作区(Working Directory)即仓库所在的文件夹
* 暂存区(Staging Area /Index)用于保存即将提交到git仓库的修改内容
* 本地仓库 (Local Repository)通过`git init`创建的仓库，是git存储代码和版本信息的主要位置

当修改完工作区的文件后，需要用`git add`命令将其添加到暂存区，然后再用`git commit`命令将暂存区的修改提交到本地仓库中。

## Git中文件状态

* 未跟踪 Untrack ：新创建的还未被git管理的文件
* 未修改 Unmodified：已经被git管理，但内容没有发生变化的文件
* 已修改 Modified：与之前版本不同，但是还未添加到暂存区中的问及那
* 已暂存 Staged：修改后，并且已经添加到暂存区内的文件

图示如下：

![git文件状态图](https://gitee.com/wangziming707/note-pic/raw/master/img/git%E6%96%87%E4%BB%B6%E7%8A%B6%E6%80%81%E5%9B%BE.png)

# 基础命令

## 设置全局用户

~~~shell
git config --global user.name "用户名"
git config --global user.email "电子邮件"
~~~

## 创建仓库

~~~shell
#建立仓库,在指定的目录文件夹下执行该命令，将建立该文件夹的仓库
git init [repoName]
#通过远程仓库创建本地仓库
git clone {remoteRepoUrl}
~~~

## 将文件添加到仓库

~~~shell
# 查看仓库状态
git status

#将文件添加到暂存区,可以使用通配符*匹配添加多个文件 
git add 
{ .    						#添加当前目录下的所有文件
| dir1[,dir2.. ]   			#添加指定目录包括子目录
| fileName1 [,fileName2..]} #添加指定文件
git add *.txt

# 取消暂存
git restore --staged <file>...
# 查看暂存区
git ls-files
# 提交 只会提交暂存区中的文件，而不会提交工作区的其他文件 如果不指定文件，提交暂存区所有文件
git commit [file1] [file2]
git commit -m <message>
git commit -a # -a 参数设置修改文件后不需要执行 git add 命令，直接来提交
# 示例
git commit -am "第一次提交"

# 查看仓库提交记录
git log
git log --online #查看简洁提交记录
~~~



## 版本回退

~~~bash
# 回退版本
# soft 参数用于回退到某个版本，并且保留工作区和暂存区的所有修改内容
# hard 参数用于回退到某个版本，并且丢弃工作区和暂存区的所有修改内容
#mixed 为默认参数，参数用于回退到某个版本，并且保留工作区而丢弃暂存区的所有修改内容

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

soft和mixed参数的使用场景一般是在进行多次提交后，发现有些提交没有太大意义，可以合并成一个版本时，可以用这两个参数来进行回退之后再重新提交，在重新提交之前，mixed模式需要执行一下`git add`操作将变动的文件重新添加到暂存区

hard参数的使用场景一般是需要放弃目前本地的所有修改内容时，需要谨慎使用，如果误操作了，可以使用`git reflog`命令查看操作的历史记录，找到误操作之前的版本号，然后用`git reset`命令回退到误操作之前就可以。

## 查看文件差异

`git diff`命令可以查看：

* 工作区、暂存区、本地仓库之间的差异
* 不同版本之间的差异
* 文件在两个分支之间的差异

~~~shell
# 查看差异 无参数时默认比较工作区和暂存区之间的差异,没有指定文件时默认比较整个版本库
git diff [filename]
git diff [filename] HEAD # 比较工作区和版本库之间的差异
git diff [filename] --cached # 比较暂存区和版本库之间的差异
git diff [filename] 版本id 版本id  # 比较两个版本之间的差异
git diff [filename] 分支名 分支名  # 比较两个分支之间的差异
#例如
git diff 6a0cc18  99ae446
git diff HEAD  6a0cc18
git diff HEAD~ HEAD
~~~

## 从版本库中删除文件

有两种方式：

* 直接删除文件，然后提交：

    ~~~shell
    rm filename
    git add.
    git commit -m "commit"
    ~~~

* 使用`git rm`命令：

    ~~~shell
    git rm filename # 将指定文件从工作区和暂存区删除
    git rm --cached filename # 将指定文件从暂存区删除,不删除工作区的
    git commit -m "commit"

## `.gitignore`

`.gitignore`文件可以让我们忽略掉一些不应该被加入到版本库中的文件，一般来说，应该忽略下面文件：

* 系统或软件自动生成的文件
* 编译产生的中间文件和结果文件
* 系统运行中生成的日志文件、缓存文件、临时文件
* 设计身份、密码、口令、密钥等敏感信息文件

只需要在`.gitignore`文件中列出需要忽略的文件的模式，这些文件就不会被提交到版本库

该文件有如下规则：

* 任何以井号（#）开头的行都会被认为是注释，Git会忽略这些行
* 使用标准的Blob模式匹配，例如：
    * 星号`*`匹配零个或多个任意字符
    * 问号`?`匹配任意一个字符
    * 方括号`[]`匹配指定范围内的任意一个字符

* 双星号`**`匹配任意数量的中间目录
* 感叹号`!`表示取反，表示不忽略指定文件/目录

~~~shell
# 可以使用简单的文件名或路径匹配规则来指定要忽略的文件和目录
# 忽略所有 .log 文件
*.log
# 忽略特定文件
temp.txt
# 忽略特定目录
build/

# 忽略所有 .log 文件
*.log
# 忽略所有以 temp 开头的文件
temp*
# 忽略所有以 a 开头、任意一个字符结尾的文件
a?
# 忽略 a、b 或 c 开头的文件
[a-c]*

# 在文件名后面加上斜杠（/）可以指定要忽略的目录。
# 忽略所有的日志目录
logs/
# 在规则前加上感叹号（!）可以指定不忽略的文件或目录。这在处理嵌套目录时特别有用。
# 忽略所有 .log 文件，但不忽略 debug.log
*.log
!debug.log
# 。
# 忽略任何位置的临时文件
**/temp/*
# 忽略所有目录下的 .DS_Store 文件
**/.DS_Store
~~~

注意事项:

* `.gitignore`文件必须放在Git仓库的根目录中，或放在任何子目录中以定义该目录特定的忽略规则。

* `.gitignore`文件中的规则是相对于其所在目录的。

* 如果一个文件已经被Git追踪，修改.gitignore 文件不会使其停止被追踪，需要手动将文件从版本库中删除(通过`git rm --cached <filename>`)

# 远程仓库

## 使用ssh方式访问远程仓库

### 生成ssh密钥

到用户目录的`.ssh`文件夹下(Windows为`C:\Users\xxxx\.ssh`)执行下面命令：

~~~shell
ssh-keygen -t rsa -b 4096
~~~

这个命令会生成下面密钥文件到`.ssh`文件夹下：

~~~shell
id_rsa # 私钥文件
id_rsa.pub # 公钥文件
~~~

我们需要将公钥文件的内容，复制下来，粘贴到远程仓库管理网站的相关地方，以github为例：

在`个人头像->settings->SSH and GPG keys`下的页面中点击 new SSH key按钮添加公钥

### 关联ssh密钥到github

如果是第一次生成密钥，并且使用的是默认名称，那么下面步骤可以忽略。

否则，需要在`.ssh`文件夹下配置一个`config`文件，该文件内容如下：

~~~shell
# github
Host github.com
HostName github.com
PreferredAuthentications publicKey
IdentityFile ~/.ssh/[私钥文件名]
~~~

该文件指示在访问github时，使用指定的密钥

这样就可以使用ssh链接来访问远程仓库了，例如：

~~~shell
git clone git@github.com:wangziming-well/note.git
~~~

## 关联远程仓库

`git init`命令创建的本地仓库默认是没有关联任何远程仓库的。

`git remote add `命令为当前的本地仓库关联一个远程仓库

~~~shell
# 添加一个远程仓库
git remote add <shortname> <url>
#例如
git remote add origin git@github.com:wangziming-well/note.git
# 查看所有的远程方库
git remote [-v]
# 删除远程仓库
git remote rm [别名]

# 复制远程仓库到本地
git clone <repository> <directory>
~~~

## 同步机制

git是一个分布式的版本控制系统，我们的本地仓库和远程仓库是两个独立的仓库。

我们在本地仓库的任何修改都不会影响到远程仓库，反之亦然。

所以需要使用一种机制来同步远程仓库和本地仓库的状态。这就是下面命令：

* `git push`将本地仓库的修改同步到远程

* `git pull`将远程仓库的修改同步到本地，执行该命令是git会自动为我们执行一次合并操作

    如果远程仓库的内容和本地仓库内容没有冲突的话，那么合并操作会成功，否则就会失败，需要手动处理冲突。

* `git fetch`命令也是拉取远程仓库到本地，但不会自动执行合并操作，需要我们手动合并

~~~shell
git push <远程仓库名> <远程分支名>:<本地分支名>
git push -u origin main:main # 将本地仓库的main分支推送到远程仓库的main分支
git push -u origin main # 如果本地仓库的分支和远程仓库分支相同，可以省略，只写一个main

# 拉取远程仓库的指定分支到本地当前分支并合并
git pull <远程仓库名> <远程分支名>:<本地分支名>
# 远程仓库名和分支名可以省略，此时默认拉取origin的main分支并合并到本地main
git pull 

#远程仓库下载新分支与数据
#执行完后需要执行git merge 合并远程分支到你所在的分支。
git fetch <远程仓库名> <远程分支名>:<本地分支名>
~~~

**注意：**在执行pull之后，进行下一次push之前，如果其他人进行了推送内容到远程数据库的话，那么你的push将被拒绝。此时需要先再pull，在尝试push



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









# Git工具

## posh-git

window环境下安装posh-git:

* 在PowerShell中运行以下命令来安装posh-git：I

    ~~~shell 
    Install-Module -Name posh-git -Scope CurrentUser
    ~~~

* 安装完成后，重启PowerShell并在仓库目录下输入以下命令启用posh-git：

    ~~~shell
    Import-Module posh-git
    ~~~

### 安装问题解决

如果执行`Import-Module`时抛出异常：

~~~shell
Import-Module : 无法加载文件 C:\Users\xxx\Documents\WindowsPowerShell\Modules\posh-git\1.1.0\posh-g
~~~

说明当前的PowerShell不支持脚本执行策略。

可以以管理员身份执行PowerShell，然后执行下面命令：

~~~shell
 Set-ExecutionPolicy RemoteSigned
~~~

**中文乱码问题**（如果需要）

- 在PowerShell中输入命令`$env:LESSCHARSET='utf-8'`并执行，以解决中文乱码问题。
- 若要永久解决，可以将该命令添加到PowerShell的配置文件（如`Microsoft.PowerShell_profile.ps1`）中。

## git-lfs



## git crypt

window环境下安装配置：

* 在git crypt对应的<a href="https://github.com/oholovko/git-crypt-windows/releases">github仓库</a>中下载对应的`git-crypt.exe`可执行文件

* 将该可执行文件放入git文件夹的bin目录中，以加入环境变量

### 首次加密

* 在需要加密的仓库根目录执行：

  ~~~shell
  git-crypt init
  ~~~

  

* 在仓库的根目录下新建一个名为“.gitattributes”的文件。

* 打开“.gitattributes”文件，设置加密范围，语法如下：

  ~~~shell
  文件名或文件范围 filter=git-crypt diff=git-crypt
  ~~~

  可以使用表示的bash通配符进行配置，例如：

  ~~~shell
  FT/file01.txt filter=git-crypt diff=git-crypt   #将特定文件加密，这里加密的是FT文件夹下的file01.txt
  *.java filter=git-crypt diff=git-crypt   #将 .java类型文件加密
  G* filter=git-crypt diff=git-crypt       #将 文件名为 G 开头的文件加密
  ForTest/** filter=git-crypt diff=git-crypt #将 ForTest 文件夹下的文件加密 
  ~~~

* 进行文件加密，在仓库根目录执行下面命令完成加密：

  ~~~shell
  git-crypt status
  ~~~

加密执行后，在您的本地仓库仍能明文方式打开和编辑这些加密文件，这是因为您本地仓库有密钥存在。

这时你可以使用add 、commit、push组合将仓库推送到代码托管仓库，此时加密文件将一同被推送。

加密文件在代码托管仓库中将以加密二进制方式存储，无法直接查看。如果没有密钥，就算将其下载到本地，也无法解密。

**说明**：`git-crypt status`只会加密本次待提交的文件，对本次未发生修改的历史文件不会产生加密作用，Git会对此设定涉及的未加密文件做出提示（见上图中的Warning），如果想将仓库中的对应类型文件全部加密，请使用`git-crypt status -f`

在让团队合作中 -f （强制执行）具有一定的风险，请谨慎使用。

执行下面命令导出加密密钥，以供其他电脑解密用

~~~shell
git-crypt export-key /path/to/keyfile
~~~

### 解密

拉取有加密文件的仓库到本地后，在仓库根目录执行下面命名：

~~~shell
git-crypt unlock /C/test/KeyFile     #请将 /C/test/KeyFile 更换为您实际的密钥存储路径
~~~

初始化git-crypt并解密

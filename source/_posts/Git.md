---
title: Git
tags: Git
categories: Git
encrypt: 
enc_pwd: 
abbrlink: 25246
date: 2016-03-11 21:48:27
summary_img:
---

## 一 git简介

​	Git 是一个开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目。

Git 与 SVN 区别点：

- **1、Git 是分布式的，SVN 不是**：这是 Git 和其它非分布式的版本控制系统，例如 SVN，CVS 等，最核心的区别。
- **2、Git 把内容按元数据方式存储，而 SVN 是按文件：**所有的资源控制系统都是把文件的元信息隐藏在一个类似 .svn、.cvs 等的文件夹里。
- **3、Git 分支和 SVN 的分支不同：**分支在 SVN 中一点都不特别，其实它就是版本库中的另外一个目录。
- **4、Git 没有一个全局的版本号，而 SVN 有：**目前为止这是跟 SVN 相比 Git 缺少的最大的一个特征。
- **5、Git 的内容完整性要优于 SVN：**Git 的内容存储使用的是 SHA-1 哈希算法。这能确保代码内容的完整性，确保在遇到磁盘故障和网络问题时降低对版本库的破坏。

## 二 git工作流程

一般工作流程如下：

- 克隆 Git 资源作为工作目录。
- 在克隆的资源上添加或修改文件。
- 如果其他人修改了，你可以更新资源。
- 在提交前查看修改。
- 提交修改。
- 在修改完成后，如果发现错误，可以撤回提交并再次修改并提交。

## 三 git的工作区,暂存区,版本库

我们先来理解下 Git 工作区、暂存区和版本库概念：

- **工作区** 就是你在电脑里能看到的目录。

- **暂存区** 英文叫 stage 或 index。一般存放在 .git 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。

- **版本库** 工作区有一个隐藏目录 .git，这个不算工作区，而是 Git 的版本库。

  ```
  版本库存了很多东西，最重要的就是stage(或index)的暂存区，还有git为我们自动创建的第一个分支master,以及指向master的指针叫HEAD

  git add 实际是把文件添加到暂存区，Git commit 是把暂存区的所有文件提交到当前分支（在分支中有版本的概念）
  ```

  ​

![img](/images/git/q.jpg)

- 图中左侧为工作区，右侧为版本库。在版本库中标记为 "index" 的区域是暂存区（stage/index），标记为 "master" 的是 master 分支所代表的目录树。
- 图中我们可以看出此时 "HEAD" 实际是指向 master 分支的一个"游标"。所以图示的命令中出现 HEAD 的地方可以用 master 来替换。
- 图中的 objects 标识的区域为 Git 的对象库，实际位于 ".git/objects" 目录下，里面包含了创建的各种对象及内容。
- 当对工作区修改（或新增）的文件执行 git add 命令时，暂存区的目录树被更新，同时工作区修改（或新增）的文件内容被写入到对象库中的一个新的对象中，而该对象的ID被记录在暂存区的文件索引中。
- 当执行提交操作（git commit）时，暂存区的目录树写到版本库（对象库）中，master 分支会做相应的更新。即 master 指向的目录树就是提交时暂存区的目录树。
- 当执行 git reset HEAD 命令时，暂存区的目录树会被重写，被 master 分支指向的目录树所替换，但是工作区不受影响。
- 当执行 git rm --cached <file> 命令时，会直接从暂存区删除文件，工作区则不做出改变。
- 当执行 git checkout . 或者 git checkout -- <file> 命令时，会用暂存区全部或指定的文件替换工作区的文件。这个操作很危险，会清除工作区中未添加到暂存区的改动。
- 当执行 git checkout HEAD . 或者 git checkout HEAD <file> 命令时，会用 HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和以及工作区中的文件。这个命令也是极具危险性的，因为不但会清除工作区中未提交的改动，也会清除暂存区中未提交的改动。

## 四 创建仓库

## git init

Git 使用 **git init** 命令来初始化一个 Git 仓库，Git 的很多命令都需要在 Git 的仓库中运行，所以 **git init** 是使用 Git 的第一个命令。

在执行完成 **git init** 命令后，Git 仓库会生成一个 .git 目录，该目录包含了资源的所有元数据，其他的项目目录保持不变。

### 使用方法

使用当前目录作为Git仓库，我们只需使它初始化。

```
git init
```

该命令执行完后会在当前目录生成一个 .git 目录。

使用我们指定目录作为Git仓库。

```
git init newrepo
```

初始化后，会在 newrepo 目录下会出现一个名为 .git 的目录，所有 Git 需要的数据和资源都存放在这个目录中。

如果当前目录下有几个文件想要纳入版本控制，需要先用 git add 命令告诉 Git 开始对这些文件进行跟踪，然后提交：

```
$ git add *.c
$ git add README
$ git commit -m '初始化项目版本'
```

以上命令将目录下以 .c 结尾及 README 文件提交到仓库中。

------

## git clone

我们使用 **git clone** 从现有 Git 仓库中拷贝项目（类似 **svn checkout**）。

克隆仓库的命令格式为：

```
git clone <repo>
```

如果我们需要克隆到指定的目录，可以使用以下命令格式：

```
git clone <repo> <directory>
```

**参数说明：**

- **repo:**Git 仓库。
- **directory:**本地目录。

比如，要克隆 Ruby 语言的 Git 代码仓库 Grit，可以用下面的命令：

```
$ git clone git://github.com/schacon/grit.git
```

执行该命令后，会在当前目录下创建一个名为grit的目录，其中包含一个 .git 的目录，用于保存下载下来的所有版本记录。

如果要自己定义要新建的项目目录名称，可以在上面的命令末尾指定新的名字：

```
$ git clone git://github.com/schacon/grit.git mygrit
```

## 配置

git 的设置使用 git config 命令。

显示当前的 git 配置信息：

```
$ git config --list
credential.helper=osxkeychain
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true
core.ignorecase=true
core.precomposeunicode=true
```

编辑 git 配置文件:

```
$ git config -e    # 针对当前仓库 
```

或者：

```
$ git config -e --global   # 针对系统上所有仓库
```

设置提交代码时的用户信息：

```
$ git config --global user.name "runoob"
$ git config --global user.email test@runoob.com
```

如果去掉 **--global** 参数只对当前仓库有效。

## 五 git基本操作

### 5.1 版本创建与回退

创建版本

```
git add code.text
git commit -m '版本1'
```

查看版本

```
git log
git log --pretty=oneline
打印的：commit后的为版本的序列号

git reflog  查看之前的记录
```

版本回退

```
git reset --hard HEAD^  //HEAD^ ,HEAD^^

HEAD表示当前最新版本，HEAD^表示当前版本的前一个版本，HEAD^^前前版本，或HEAD~1前一版本，HEAD~2当前版本的前2个版本

当有两个版本 v1和v2  使用回退从v2推到v1 则在回去v2 使用以下命令
git reset --hard 版本号   commit后的为版本号（前7位即可）

查看之前的版本：
git reflog
```

撤销修改未add到暂存区的文件（丢弃工作区的改动（未add的））

```
git checkout -- 文件名
```

撤销修改暂存区的文件（丢弃暂存区的文件）

```
1 git reset HEAD 文件名   把暂存区的修改撤掉，重新放回工作区
2 git checkout -- 文件名  把他工作区的修改撤回
```

撤销修改提交到版本库的

```
1 回退版本
2 git reset HEAD 文件名   把暂存区的修改撤掉，重新放回工作区
3 git checkout -- 文件名  把他工作区的修改撤回
```

对比文件的不同

```
1 对比工作区和某个版本的文件不同
git diff HEAD -- code.txt
2 对比两个版本的文件不同
git diff HEAD HEAD~1 -- code.txt
```

删除文件

````
1 工作区
 直接删除即可
````

## 六 分支管理

![img](/images/git/ma.png)

当我们新建分支时，如dev,git新建一个指针dev,指向master相同的提交，再把HEAD指向dev，表示当前分支在dev上，

不过从现在开始，对工作区的修改和提交就是针对dev分支了，如新的提交后，dev指针前移，而master的指针不变

![img](/images/git/ma2.png)

假如我在dev上的工作完成了，就可以吧dev合并到master上，最简单的就是直接把master指向dev的当前提交，就完成了合并

```
查看分支和当前在哪个分支
git branch
创建分支
git branch  dev
创建+切换到分支
git checkout -b test
切换分支
git checkout dev
切换回master
git checkout master
将某个分支合并到master(在master分支下操作)
git merge test
合并某分支到当前分支
git merge <name>
删除分支
git branch -d test
```

冲突合并

![img](/images/git/mer.png)

两个分支对同一文件修改，这种情况没法快速合并，只能试图把各自的修改合并起来，但冲突

解决冲突

合并后手动修改文件后再次合并

````
解决冲突合并后
git  log --graph --pretty=oneline
可以查看冲突合并的情况
````

```
禁用快速合并的合并
git merge --no-ff -m '禁用快速合并的合并' dev
```

bug分支

开发时遇见bug，通常是每个bug一个分支，修复后合并

如你正在开发任务，且代码不可提交（未完成），但需修复bug

```
1 将当前的代码储藏起来（dev）
git stash
2 切换到master 创建bug分支，修复任务，然后合并
3 切换回dev 找回工作现场
git stash list 列出以保存的工作现场
git stash pop  恢复现场
```

```
推送分支
git push origin 分支名称
```



## 七 github

```
1 github创建仓库
2 添加ssh账户
需要将这台电脑的ssh公钥添加到github
3 git clone 
```

windows生成ssh

```
在Windows下查看[c盘->用户->自己的用户名->.ssh]下是否有"id_rsa、id_rsa.pub"文件，如果没有需要从第一步开始手动生成,有的话直接跳到第二步
1 
打开Git Bash，在控制台中输入以下命令:
ssh-keygen -t rsa -C "youremail@example.com"
密钥类型可以用 -t 选项指定。如果没有指定则默认生成用于SSH-2的RSA密钥。这里使用的是rsa。
同时在密钥中有一个注释字段，用-C来指定所指定的注释，可以方便用户标识这个密钥，指出密钥的用途或其他有用的信息。所以在这里输入自己的邮箱或者其他都行,当然，如果不想要这些可以直接输入：ssh-keygen
输入完毕后按回车，程序会要求输入一个密码，输入完密码后按回车会要求再确认一次密码，如果不想要密码可以在要求输入密码的时候按两次回车，表示密码为空，并且确认密码为空，此时[c盘>用户>自己的用户名>.ssh]目录下已经生成好了。
2 将SSH添加到版本管理仓库
不同的版本管理代码仓库都大同小异，这里以Github举例，登录Github。打开setting->SSH keys，点击右上角 New SSH key，把[c盘->用户->自己的用户名->.ssh]目录下生成好的公钥"id_rsa.pub"文件以文本打开复制放进 key输入框中，再为当前的key起一个title来区分每个key。
```

````
推送分支
git push origin 分支名称
本地分支跟踪远程分支
git branch --set-upstream-to=origin/smart smart
拉取代码
git pull  origin 分支名称
````


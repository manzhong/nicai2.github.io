---
title: Anaconda
tags: Python
categories: Python
abbrlink: 41489
date: 2019-07-19 22:48:19
summary_img:
encrypt:
enc_pwd:
---

## 一 简介和安装

### 1 简介

Anaconda（[官方网站](https://link.jianshu.com/?t=https%3A%2F%2Fwww.anaconda.com%2Fdownload%2F%23macos)）就是可以便捷获取包且对包能够进行管理，同时对环境可以统一管理的发行版本。Anaconda包含了conda、Python在内的超过180个科学包及其依赖项。

### 2 Anaconda、conda、pip、virtualenv的区别

### 3 安装

**配置数据源**

```shell
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/bioconda
conda config --set show_channel_urls yes #设置搜索时显示通道地址
conda config --get channels #查看已配置的源
```

## 二 使用

**管理conda**

```shell
conda --version
conda update conda #更新
conda --help
rm -rf ~/anaconda3 #卸载
```

**管理环境**

```shell
conda create --name <env_name> <package_names> #创建环境

<env_name>即创建的环境名。建议以英文命名，且不加空格，名称两边不加尖括号“<>”。
<package_names>即安装在环境中的包名。名称两边不加尖括号“<>”。
如果要安装指定的版本号，则只需要在包名后面以=和版本号的形式执行。如：conda create --name python2 python=2.7，即创建一个名为“python2”的环境，环境中安装版本为2.7的python。
如果要在新创建的环境中创建多个包，则直接在<package_names>后以空格隔开，添加多个包名即可。如：conda create -n python3 python=3.5 numpy pandas，即创建一个名为“python3”的环境，环境中安装版本为3.5的python，同时也安装了numpy和pandas。
--name同样可以替换为-n

source activate <env_name> #切换环境 window 为 activate <env_name>
source deactivate #退出环境
conda info --envs #显示已创建的环境 *为激活的环境
conda create --name <new_env_name> --clone <copied_env_name> #复制环境 复制的环境配置相同
conda remove --name <env_name> --all #删除环境
```

**管理包**

```shell
conda search --full-name <package_full_name>  #精确查找可供安装的包 没有的话无法使用conda install 安装
conda search <text> #模糊查找 没有的话无法使用conda install 安装
conda list #获取当前环境中已安装的包
conda list -n my_env  # 列出 my_env环境中所有的安装包
conda list --export > package-list.txt  # 把当前环境中的所有包导出list，以备后续使用
conda create -n my_env --file package-list.txt  # 对导出的包list重新安装到my_env 环境中
conda install package_name  #在env_name环境中安装包
pip install package_name #当使用conda install无法进行安装时，可以使用pip进行安装
conda install --name <env_name> <package_name> #指定环境安装包
# pip只是包管理器，无法对环境进行管理。因此如果想在指定环境中使用pip进行安装包，则需要先切换到指定环境中，再使用pip命令安装包。
#pip可以安装一些conda无法安装的包；conda也可以安装一些pip无法安装的包。因此当使用一种命令无法安装包时，可以尝试用另一种命令。
conda remove --name <env_name> <package_name> #卸载指定环境中的包
conda uninstall --name <env_name> <package_name>#卸载指定环境中的包
conda remove <package_name> #卸载当前环境的包
conda uninstall <package_name> #卸载当前环境的包
conda update --all #更新所有的包 在安装Anaconda之后执行上述命令更新Anaconda中的所有包至最新版本，便于使用。
conda update <package_name> #更新指定的包
conda update pandas numpy matplotlib #更新多个包
#从Anaconda.org安装包
#浏览器输入 http://anaconda.org 找到包,复制安装命令到终端即可
```




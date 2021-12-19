---
title: Python随笔
tags: Python
categories: Python
abbrlink: 42665
date: 2021-10-19 18:05:18
summary_img:
encrypt:
enc_pwd:
---

### 1 jupyter 安装多内核

```shell
# 如3.6 前提安装好anaconda和jupyter
1: conda create -n py3.6 python=3.6
2: conda activate py3.6 (退出 conda deactivate)
3: pip install ipykernel -i https://pypi.tuna.tsinghua.edu.cn/simple
4: conda deactivat
5: python -m ipykernel install --name py3.6 (退出虚拟环境后运行)
ps卸载内核: jupyter kernelspec remove py3.6
```

### 2 pip install name 指定源

```
pip install name -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 3 Excel xlsx file; not supported

```shell
1:pip uninstall xlrd
2:pip install xlrd=='1.2.0' -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 4 pandas报DataFrame object has no attribute 'as_matrix'解决办法

```
NDFrame.as_matrix is deprecated. Use NDFrame.values instead (:issue:18458).
在新版中as_matrix 已被移除
使用df.values 替代
```


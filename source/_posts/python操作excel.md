---
title: python操作excel
tags:
  - Python
categories: Python
encrypt: 
enc_pwd: 
abbrlink: 38481
date: 2020-06-01 23:14:07
summary_img:
---

## 一、概述

操作 excel 是程序员经常要遇到的场景。因为产品、运营的数据都是以这种格式存储。所以，当程序员拿到这些数据肯定要解析，甚至需要把结果输出成 excel 文件。

下面就介绍如果用 Python 方面的读、写 excel 文件。

## 二、openpyxl

> A Python library to read/write Excel 2010 xlsx/xlsm files

借助 Python 的三方库 `openpyxl` ，让操作 excel 变得简单。

1. 安装：`pip install openpyxl`
2. 文档：[官方文档](https://openpyxl.readthedocs.io/en/stable/)
3. 示例代码：

```python
# coding=utf-8
from openpyxl import Workbook
wb = Workbook()

# 选择 sheet
ws = wb.active

# 设置值到某一个单元格（cells）
ws['A1'] = 42

# Python 的数据类型可以自动转换
import datetime
ws['A2'] = datetime.datetime.now()

# 存储文件
wb.save("sample.xlsx") # 默认保存到当前目录下。文件名称为 sample.xlsx
```

## 读数据

```python
from openpyxl import load_workbook

wb = load_workbook('sample.xlsx') # 读取文件
sheet = wb.get_sheet_by_name("Sheet") # 根据 sheet 名称获取，返回 Worksheet 对象
columns = sheet['A'] # 选择一列
for fi_column in columns:
    # 遍历这列的所有行
    print fi_column.value # 每一个fi_column是 Cell 对象
```

## 写数据

```python
from openpyxl import Workbook
wb = Workbook()
# 选择 sheet
ws = wb.create_sheet()
# result_list ->[[第一行数据], [第二行数据], ...]
for fi_result in result_list:
    ws.append(fi_result) # 每行的数据
# 存储文件
wb.save("test.xlsx")
```

## 更多 API

- Worksheet.columns()：获取 sheet 所有列
- Worksheet.iter_cols()：通过列截断
- Worksheet.rows()：获取 sheet 所有行
- Worksheet.iter_rows()：通过行截断
- Worksheet.cell()：操作单元格
- Workbook.save()：存储文件
- workbook.Workbook.create_sheet()：创建新的 sheet
- Workbook.sheetnames()：获取 sheet 名称


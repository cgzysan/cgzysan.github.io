---
title: python实现excel转json
date: 2018-04-02 23:35:56
tags: python excel转json
---

### 前言

> 工作项目中，合作方提供了一张Excel的数据表，需要将其转成 Json 文件，正好最近学习了python，就决定用 python 来试试水，实现这个功能。

<!-- more -->

#### 工作环境

- python 3.6
- python 对 Excel 的操作，需要依赖 xlrd, xlwt 这两个模块，xlrd 是读 Excel 的模块，xlwt 是写 Excel 的模块 
- pip install xlrd
- pip install xlwt

拿到 Excel 表仔细扫了一眼，我发现表格中有一列数据基本就没有，但是这一列又是比较重要的一项，内容被包含在了另一列的数据中，所以我决定先将 Excel 表重新用 python 写一遍，所以将 xlwt 包也加了进来，如果只是读 Excel 文件，只需要 xlrd 就够了。

#### xlrd

打开Excel文件读取数据

```python
data = xlrd.open_workbook(file_path)	# 返回一个xlrd.book.Book实例
```

获取 book 中一个工作表

```python
table = data.sheets()[0]					#通过索引顺序获取，返回xlrd.sheet.Sheet实例
table = data.sheet_by_index(sheet_indx))	#通过索引顺序获取，返回xlrd.sheet.Sheet实例
table = data.sheet_by_name(sheet_name)		#通过名称获取，返回xlrd.sheet.Sheet实例
```

获取 book 中所有工作表的名称

```python
 names = data.sheet_names()    #返回book中所有工作表名字的集合list
```

获取该 sheet 中的有效行数

```python
nrows = table.nrows
```

获取该行中所有的单元格对象组成的集合list

```python
table.row(rowx)		#返回的带有数据的类型
```

获取该行中所有单元格数据组成的集合list

```python
table.row_values(rowx, start_colx=0, end_colx=None)
```

#### xlwt

新建 excel 文件

```python
file = xlwt.Workbook(encoding="utf-8")	#默认是 ascii
```

新建 sheet 表

```python
table = file.add_sheet(sheet_name, cell_overwrite_ok=False)	#返回xlwt.Worksheet.Worksheet实例，如果需要对一个单元格重复操作，需要将 cell_overwrite_ok=True
```

向 x 行 y 列 写入数据 value

```python
table.write(x, y, value)
```

保存文件

```python
file.save(file_name)
```

#### 对Excel表添加数据

```python
def completion_excel():
    file_path = input("输入需要操作的Excel文件的绝对路径>>>")
    data = achieve_data(file_path)
    if data is not None:
        worksheets = data.sheet_names()
        print("包含的表单:")
        for index, sheet in enumerate(worksheets):
            print(index, sheet)
        choose = input("请输入表单对应的编号>>>")
        table = data.sheet_by_index(int(choose))

        # 创建Excel表
        workbook = xlwt.Workbook(encoding="utf-8")
        # 创建 sheet
        worksheet = workbook.add_sheet("sheet1")
        # 先写表头
        titles = table.row_values(0)
        for k, v in enumerate(titles):
            worksheet.write(0, k, v)
        for i in range(1, table.nrows):
            row = table.row_values(i)
            # 补全年级的表
            if row[4] == "":
                course = row[6]
                if "年级" in course:
                    n = course.index("年级")
                    row[4] = course[n-1:n+3]
                else:
                    row[4] = "初中"

            for x, y in enumerate(row):
                worksheet.write(i, x, y)
        workbook.save("Excel_test.xls")
```

#### 将新的Excel文件转成Json

```python
def excel2json():
    file_path = input("输入需要转成Json的Excel文件的路径>>>")
    data = achieve_data(file_path)
    if data is not None:
        # 抓取所有sheet页的名称
        worksheets = data.sheet_names()
        print("包含的表单:")
        for index, sheet in enumerate(worksheets):
            print(index, sheet)
        choose = input("请输入表单对应的编号>>>")
        table = data.sheet_by_index(int(choose))
        # 获取到数据的表头
        titles = table.row_values(0)
        result = {}
        result["middle_school"] = []
        # excel文件表有 10196 行，所以做10196次循环
        for i in range(1, table.nrows):
            row = table.row_values(i)
            tmp = {}
            for index, title in enumerate(titles):
                if title == "id":
                    tmp[title] = int(row[index])
                else:
                    tmp[title] = row[index]
            result["middle_school"].append(tmp)
        with open("middle_school.txt", 'w', encoding='utf-8') as f:
            json.dump(result, f)
```

[完整代码地址](https://github.com/cgzysan/pystudy/blob/master/day16/excel2json.py)
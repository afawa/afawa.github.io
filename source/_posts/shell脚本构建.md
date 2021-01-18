---
title: shell脚本构建
date: 2021-01-08 17:24:10
tags: shell programming
categories: programming language
toc: true
---

## 构建基本shell脚本

### 显示输出，引用变量

1. `echo` 命令，使用""和‘’进行区分
2. `echo -n` 随后的输出不会换行
3. `$name`，`${name}` 引用变量 ，使用`\$`进行转义

<!--more-->

### 命令替换

1. 两种方法，反引号`和$( )格式
2. 命令替换会创建一个子shell来运行对应的命令，所以无法使用在脚本中所创建的变量
3. 在命令提示符下使用路径./运行命令的话，也会创建出子shell，要是运行命令的时候不加入路径，就不会创建子shell。如果使用的是shell内建命令，不会涉及子shell

### 重定向

1. \>\> 重定向输出并且不会清空文件原有的内容

2. 内联输入重定向符号 <<，除了这个符号，你必须指定一个文本标记来划分输入数据的开始和结尾 
    ``` bash
    $ wc << EOF
    > test string 1
    > test string 2
    > test string 3
    > EOF
    ```

### 执行数学运算

1. 第一种方式使用`expr`命令，例如`expr 1 + 5`，`expr 2 \* 5`，在shell中使用需要进行命令替换

2. 第二种方式使用方括号，例如 `var=$[1 + 5]`，`var=$[2 * 5]`

3. bash shell 原生只支持整形运算

4. 在bash中使用浮点运算的一种方式是使用内建计算器bc，在脚本中使用bc，可以使用命令替换+管道的方式，例如 
    ``` bash
    var1=100
    var2=45
    var=$(echo "scale=4; $var1 / $var2" | bc)
    ```
   或者使用内联输入重定向，例如
   ``` bash
    var=$( bc << EOF
    scale=4
    a1 = ($var1 * $var2)
    b1 = ($var3 * $var4)
    a1 + b1
    EOF
    )
   ```

### 退出脚本

   1. linux提供了专门的变量`$?`来存储上一个命令的退出状态码，可以直接用`echo $?`查看
   2. 默认情况下shell脚本会以脚本的最后一个命令的退出码退出
   3. 可以使用`exit`命令指定退出码
   4. 退出码范围 0~255

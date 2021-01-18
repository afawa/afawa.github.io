---
title: shell脚本结构化命令
date: 2021-01-08 18:21:18
tags: shell programming
categories: programming language
toc: true
---

## 结构化命令

### `if-then` 语句

``` bash
if command
then
    commands
fi
```

<!--more-->

1. 如果`if`后的命令的推出码是0，则会运行`then`部分。

2. 运行`if`语句中的错误信息依然会显示，可以用某些方法避免

3. 另一种形式
    ``` bash
    if command; then
        commands
    fi
    ```

### `if-then-else` 语句

``` bash
if command
then
    commands
else
    commands
fi
```

1. `if`中的语句的退出码不为0时，运行`else`中的语句

### 嵌套if

``` bash
if command1
then
    commands
elif command2
    commands
fi
```
### `test` 命令

如果`test`命令中列出的条件成立，`test`命令就会退出并返回退出码0。如果condition部分本身为空，那么`test`以非零返回。

``` bash
# test 命令
test condition
# test 命令结合if-else
if test condition
then
    commands
fi
```

bash 提供了另一种测试方式，不需要声明`test`命令。方括号定义了测试条件，第一个方括号后和第二个方括号前必须加上一个空格。

``` bash
if [ condition ]
then
    commands
fi
```

1. `test`可以测试一个变量是否为空，不为空则返回0
2. 数值比较
   1. `n1 arg n2`，其中`n1`，`n2`是数值，`arg`是选项，`arg`包括 `-eq`, `-ge`, `-gt`, `-le`, `-lt`, `-ne` （相等，大于等于，大于，小于等于，小于，不等于）。
   2. bash shell只能处理整数，`test`命令不能处理浮点数。
3. 字符串比较
   1. 字符串相等性。`str1 = str2` 判断两个字符串是否相等，`str1 != str2` 判断两个字符串是否不相等
   2. 字符串顺序。`str1 \> str2` , `str1 \< str2`。大于号小于号必须使用转义，不然会被理解成重定向符号。在test中，根据ASCII标准进行排序，`sort`和`test`对于大写小写的判断是反的。
   3. 字符串大小。`-n str1` 检查str1的长度是否为0 ，`-z str1` 检查str1的长度是否不为0。
4. 文件比较
   1. 检查目录。`-d file`，检查file是否存在并是一个目录。
   2. 检查对象是否存在。`-e file`，检查file是否存在。
   3. 检查文件。`-f file`，检查file是否存在并是一个文件。
   4. 检查是否可读。`-r file`，检查file是否存在并可读。
   5. 检查空文件。`-s file`，检查file是否存在并非空。
   6. 检查是否可写。`-w file`，检查file是否存在并可写。
   7. 检查文件是否可以执行。`-x file`，检查file是否存在并可执行。
   8. 检查所属关系。`-O file`，检查file是否存在并属于当前用户所有。
   9. 检查默认属主关系。`-G file`，检查file是否存在并且默认组与当前用户相同。
   10. 检查文件日期。`file1 -nt file2`，检查file1是否比file2新。`file1 -ot file2`，检查file1是否比file2旧。

### 复合条件测试

1. 允许使用 `&&` 和 `||` 连接多个condition。

### `if-then` 高级特性

1. 双括号。双括号命令运行在比较过程中使用高级数学符号。（几乎是所有c中的运算都能用）并且在双括号中不需要对>和<进行转义。
    ``` bash
    var=10
    if (( var ** 2 < 1000 ))
    then
        var=$(( var-- ))
    fi
    ```
2. 双方括号。双方括号中可以使用模式匹配。例如 [[ $USER == r* ]]，这里使用了==，右边的r*就是一个模式。

### `case` 命令

类似switch命令，格式如下

``` bash
case variable in
pattern1 | pattern2) commands1;;
pattern3) commands2;;
*) commands3;;
esac
```

### `for` 命令

``` bash
for var in list
do 
    commands
done
```

在`list`参数中，需要提供迭代中要用到的一系列值。

1. 读取列表中的值。`for`命令最基本的用法就是遍历`for`命令本身所定义的一系列值。
   在最后一次迭代后，`$test`的值会在shell脚本的剩余部分一直保持有效。
    ``` bash
    for test in aaa bbb ccc ddd
    do
        echo $test
    done
    ```

2. 读取列表中的复杂值。

    如果列表中的值存在单引号，可以用如下两种方式：1.使用转义，2.使用双引号。
    ``` bash
    for test in I don\'t know if "this'll" work
    do
        echo $test
    done
    ```

   如果存在空格，必须使用双引号把这个值框起来。

3. 从变量读取列表

    ``` bash
    list="aaa bbb ccc"
    list=$list" ddd"
    for test in list
    do
        echo $test
    done
    ```

4. 从命令读取值

    ``` bash
    file=testfile.txt
    for test in $(cat $file)
    do
        echo $test
    done
    ```

5. 更改字段分隔符

   内部字段分隔符，`IFS`环境变量定义了bash shell用作字段分隔符的一系列字符。默认情况下，bash shell会将下列字符当作字段分隔符：1. 空格，2. 制表符，3. 换行符

   1. 修改`IFS`，例如，`IFS=$'\n'`。这里注意用`$'\n'`，`$'string'`是用来表示带转义序列的字符串文字的语法 (called ANSI C-quoted strings)
   2. 指定多个的时候只需要简单拼接。`IFS=$'\n':;"`

6. 用通配符读取目录

   必须使用通配符，它会强制shell使用文件扩展匹配。

   ``` bash
    for file in /home/yiming/*
    do
        if [ -d "$file" ]
        then
            echo "$file is a directory"
        fi
    done
   ```

   注意这里"$file"应对文件名带空格的情况。同时可以使用多个通配符拼接，通配符加目录一起使用

### C风格`for`
``` bash
for (( variable assignment ; condition ; iteration process ))
```

1. 变量赋值可以用空格

2. 条件中的变量不已美元开头

3. 迭代过程的算式不使用expr表达式

``` bash
for (( i=1 ; i<=10 ; i++ ))
do
    echo $i
done

for (( a=1, b=10; a<=10; a++, b-- ))
do
    echo "$a - $b"
done
```

### `while` 命令

1. 基本格式

``` bash
while test command
do
    other command
done
```

   这里的`test command`和`if-else`语句一模一样。

``` bash
var1=10
while [ $var -gt 1 ]
do
    var1=$(( var - 1 ))
done
```
   
2. 使用多个测试命令

   while 命令允许你在while语句定义多个测试命令，只有最后一个测试命令的退出状态码会被用来决定什么时候退出循环。

   ``` bash
   var1=10
   while echo $var1
            [ $var1 -ge 0 ]
   do
        var=$(( var1 - 1 ))
   done 
   ```

### `until` 命令

``` bash
until test commands
do
    other commands
done
```

本质是和`while`相反的语句。

### 循环控制

1. `break`命令。直接使用`break`命令会跳出当前正在执行的循环。可以使用 `break n`来指定跳出的层数，默认n=1也就是跳出当前循环。
2. `continue`命令。和`break`相同，可以使用`continue n`的语法指定继续执行哪一级的循环。

### 处理循环的输出

在shell脚本中可以对循环的输出使用管道或进行重定向。可以通过在`done`命令后添加一个处理命令来实现。

``` bash
# 重定向例子
for (( a=1; a<=10; a++))
do
    echo $a
done > output.txt

# 也可以使用重定向进行输入，然后用read命令处理
input="users.csv"
while IFS=',' read -r userid name
do
    echo "adding $userid"
    useradd -c "$name" -m $userid
done < "$input"

# 管道例子
for (( a=1; a<=10; a++))
do
    echo $a
done | sort
```

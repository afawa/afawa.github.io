---
title: shell脚本中使用函数
date: 2021-01-19 00:36:09
tags: shell programming
categories: programming language
toc: true
---
## shell脚本使用函数

### 基本的脚本函数

#### 创建函数

有两种形式可以用来在bash shell中创建函数。

1. 第一种格式采用关键字`function`，后跟分配给该代码块的函数名。

   ``` bash
   function name {
   	commands
   }
   ```

   `name`属性定义了赋予函数的唯一名称。脚本中定义的每个函数都必须有一个唯一的名称。

<!--more-->

2. 第二种格式更接近于其他编程语言中定义函数的方法。

   ``` bash
   name() {
   commands
   }
   ```

   函数名后的括号表明正在定义一个函数。

#### 使用函数

``` bash
#!/bin/bash
function func1 {
	echo "example of function"
}
count=1
while [ count -le 5 ]
do
	func1
	count=$[ $count + 1 ]
done
```

直接使用函数名调用即可，注意函数的声明一定要在函数调用之前。如果定义了两个相同名称的函数，后面的函数会覆盖前面的函数，并且不会出现报错。

### 函数返回值

bash shell会把函数当作一个小型脚本，运行结束时会返回一个退出码。

#### 默认退出状态码

默认情况下，函数的退出码时函数中最后一条命令返回的退出状态码。在函数执行后，可以用标准变量`$?`来确定函数的退出状态码。

``` bash
function func1 {
	echo "normal command"
	ls -l badfile
}

function func2 {
	ls -l badfile
	echo "normal"
}
```

执行`func1`得到的函数返回值是1，因为函数的最后一条命令没有正常退出，但是运行`func2`时返回值为0，即使在运行`func2`的时候有命令没有正常退出。所以使用函数的默认返回值是很危险的。

#### 使用`return`命令

bash shell使用`return`命令来退出函数并返回特定的退出状态码。

``` bash
function db1 {
	read -p "Enter a value:" value
	echo "doubling the value"
	return $[ $value * 2 ]
}
```

采用这种方式时需要注意两点

1. 函数一结束就取返回值，否则`$?`变量会被覆盖
2. 退出码必须是0~255

#### 使用函数输出

可以把函数的输出保存在变量中，例如`result=$(func)`这种方式。

``` bash
function db1 {
	read -p "Enter a value:" value
	echo $[ $value * 2 ]
}
result=$(db1)
```

这种方法比较泛用。

### 在函数中使用变量

#### 向函数传递参数

bash shell会将函数当作小型脚本来对待，这意味着可以像普通脚本那样向函数传递参数。函数可以使用标准的参数函数变量来表示命令行上传给函数的参数（`$0 $1 $#`）。在脚本中指定函数时，必须将参数和函数放在同一行，例如

``` bash
func1 $value1 10
```

下面是一个例子

``` bash
function addem {
	if [ $# -eq 0 ] || [ $# -gt 2 ]
	then
		echo -1
	elif [ $# -eq 1 ]
		echo $[ $1 + $1 ]
	else
		echo $[ $1 + $2 ]
	fi
}
```

由于函数使用特殊参数环境变量作为自己的参数值，因此它无法直接获取脚本在命令行中的参数值。

#### 在函数中处理变量

1. 全局变量。如果在脚本的主体部分定义了一个全局变量，那么可以在函数内读取它的值。类似的，如果在函数内定义了一个全局变量，可以在脚本的主体部分读取它的值。默认情况下，在脚本中定义的任何变量都是全局变量。

   ``` bash
   function db1 {
   	value=$[ $value * 2 ]
   }
   
   read -p "Enter a value:" value
   db1
   echo "The new value is: $value"
   ```

   这样其实很危险，如果想要在不同的shell脚本中使用函数的话。要求需要知道函数中使用了哪些变量。

2. 局部变量。想要声明局部变量，只要在变量声明的前面加上`local`就可以了，也可以在变量的赋值时使用`local`。

### 数组变量和函数

#### 向函数传递数组参数

将数组变量当作单个参数传递的话，并不会起作用

``` bash
function testit {
	echo "The parameter are: $@"
	thisarray=$1
	echo "The received array is ${thisarray[*]}"
}
myarray=(1 2 3 4 5)
echo "The original array is: ${myarray[*]}"
testit myarray
```

这段代码的输出如下

``` bash
The origin array is: 1 2 3 4 5
The parameter are: 1
The received array is 1
```

这表明如果试图将数组变量作为函数参数，函数只会取数组变量的第一个值。

要解决这个问题，必须将该数组变量的值分解为单个的值，然后将这些值作为参数。在函数内部，可以将这些值重新组合成一个新的变量。下面是一个例子。

``` bash
#!/bin/bash
function testit {
	local newarray
	local sum=0
	newarray=($(echo "$@"))
	echo "The new array value is: ${newarray[*]}"
	for value in ${newarray}
	do
		sum=$[ $sum + $value ]
	done
	echo $sum
}
myarray=(1 2 3 4 5)
echo "The original array is: ${myarray[*]}"
result=$(testit ${myarray[*]})
echo "result is $result"
```

这样的输出就是正确的了。

#### 从函数返回数组

函数用`echo`语句来按照顺序输出数组中的值，在调用处进行处理就可以了。

``` bash
#!/bin/bash
function arraydblr {
	local origarray
	local newarray
	local elements
	local i
	origarray=($(echo "$@"))
	newarray=($(echo "$@"))
	elements=$[ $# - 1 ]
	for (( i=0; i <= $elements; i++ ))
	do
		newarray[$i]=$[ ${origarray[$i]} * 2 ]
	done
	echo ${newarray[*]}
}

myarray=(1 2 3 4 5)
echo "The original array is: ${myarray[*]}"
result=($(testit ${myarray[*]}))
echo "The new array is: ${result[*]}"
```

### 函数递归

注意`local`的使用

``` bash
function factorial {
	if [ $1 -eq 1 ]
	then
		echo 1
	else
		local temp=$[ $1 - 1 ]
		local result=$(factorial $temp)
		echo $[ $result * $1 ]
	fi
}
```

### 创建库

使用函数库的关键在于`source`命令。`source`命令会在当前shell上下文中执行命令（`./lib.sh`命令没有用因为会创建新的shell运行，函数没有装载到当前的上下文）。`source`命令有个快捷的别名，点操作符，`. ./lib.sh`。

### 在命令行上使用函数

#### 在命令行上创建函数

1. 单行方式在命令行创建函数。

   ``` bash
   $ function div { echo $[ $1 / $2 ]; }
   ```

   注意每条命令之后都要加上分号。

2. 多行方式定义函数。

   ``` bash
   $ function div {
   > echo $[ $1 / $2 ]
   > }
   $
   ```

3. 小心函数名和已有的命令名重合。

#### 在.bashrc中定义函数

1. 直接在.bashrc中定义函数

2. 定义在其他文件中，然后在.bashrc中使用`source`语句装载

   ``` bash
   if [ -r /home/yiming/lib.sh ]
   then
   	. /home/yiming/lib.sh
   fi
   ```


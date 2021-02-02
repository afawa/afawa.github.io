---
title: 'sed,gawk进阶'
date: 2021-01-21 01:35:43
tags: linux command line
categories: linux command line
toc: true
---
# `sed`，`gawk`进阶

## `sed`进阶

### 多行命令

`sed`编辑器包含了三个可用来处理多行文本的特殊命令。

1. `N`：将数据流的下一行加进来创建一个多行组来处理。
2. `D`：删除多行组中的一行
3. `P`：打印多行组中的一行

<!--more-->

#### `next`命令

1. 单行的`next`命令。小写的`n`命令告诉`sed`编辑器移动到数据流中的下一文本行，而不用重新回到命令的最开始再执行一遍。通常`sed`再移动到数据流中的下一个文本行之前，会在当前行上执行完所有定义好的命令，`n`命令改变了这个流程。

   ```bash
   $ cat data1.txt
   This is the header line.
   
   This is a data line.
   
   This is the last line.
   $ sed '/header/{n;d}' data1.txt
   This is the header line.
   This is a data line.
   
   This is the last line.
   $ 
   ```

2. 合并文本行。`n`命令会将数据流中的下一文本航班移动到`sed`编辑器的工作空间（模式空间）。多行版本的`next`命令（`N`命令）会将下一文本行添加到模式空间已有的文本后。这样的作用是将数据流中的两个文本行合并到同一模式空间中。文本行仍然采用换行符分隔，但`sed`编辑器将两行文本当成一行来处理。下面是一个例子

   ```bash
   $ cat data2.txt
   This is the header line.
   This is the first data line.
   This is the second data line.
   This is the last data line.
   $ sed '/first/{N;s/\n/ /}' data2.txt
   This is the header line.
   This is the first data line. This is the second data line.
   This is the last data line.
   $ 
   ```

   如果要再数据文件中查找一个可能会分散在两行中的文本短语的话，这是个很实用的应用程序。下面是一个例子

   ```bash
   $ cat data4.txt
   On Tuesday, The Linux System
   Administrator's group meeting will be held.
   All System Administrators should attend.
   $ sed 'N
   > s/System\nAdministrator/Desktop\nUser/
   > s/System Administrator/Desktop User/
   > ' data4.txt
   On Tuesday, The Linux Desktop
   User's group meeting will be held.
   All System Administrators should attend.
   $ sed 's/System Administrator/Desktop User/
   > N
   > s/System\nAdministrator/Desktop\nUser/
   > ' data4.txt
   On Tuesday, The Linux Desktop
   User's group meeting will be held.
   All Desktop Users should attend.
   $ 
   ```

   这里要注意一点，如果`sed`执行到最后一行的时候，这时`N`命令因为无法读取下一行会使得`sed`编辑器停止允许下面的命令，这时如果在`N`命令之后如果有针对单行的命令就无法允许，所以要记得把单行命令写在`N`命令之前，多行命令写在`N`命令之后。

#### 多行删除命令

`sed`编辑器提供了多行删除命令`D`，它只删除模式空间的第一行。该命令会删除到换行符（含换行符）为止的所有字符。

```bash
$ sed 'N;/System\nAdministrator/D' data4.txt
Administrator's group meeting will be held.
All System Administrators should attend.
$ 
```

下面的例子是删除数据流中出现在第一行的空行

```bash
$ cat data5.txt

This is the header line.
This is a data line.

This is the last line.
$ sed '/^$/{N;/header/D}' data5.txt
This is the header line.
This is a data line.

This is the last line.
$ 
```

注意这个程序只能删除第一个空行，如果出现了两个联系的空行，结果如下：

```bash
$ cat data5.txt


This is the header line.
This is a data line.

This is the last line.
$ sed '/^$/{N;/header/D}' data5.txt


This is the header line.
This is a data line.

This is the last line.
$ 
```

解释如下：首先`sed`尝试取匹配`^$`，发现第一行就是一个空行，匹配成功，开始执行`{}`中的命令，`sed`会将第二行添加入模式空间然后执行匹配，发现无法匹配。接着`sed`从第三行开始匹配空行，直到最后。所以一次`D`命令都没有执行。

`D`命令的独特之处在于强制`sed`编辑器返回到脚本的起始处，对同一模式空间中的内容重新执行这些命令（不会读取新的文本行）。下面是`man sed`对于`d`和`D`的解释：

> d      Delete pattern space.  Start next cycle.
> D      If pattern space contains no newline, start a normal new cycle as if the d command was issued. Otherwise, delete text in the pattern space up to the first newline, and restart cycle with the resultant pattern space, without reading a new line of input.

#### 多行打印命令

多行打印命令`P`只打印模式空间中的第一行。

### 保持空间

模式空间是一块活跃的缓冲区，在`sed`编辑器执行命令时它会保存带检查的文本。但它不是`sed`编辑器保存文本的唯一空间。`sed`编辑器有另一块称作保持空间的缓冲区域。在处理模式空间中的某些行时，可以用保持空间来临时保存一些行。有5条命令可以用来操作保持空间。

1. `h`。将模式空间复制到保持空间。
2. `H`。将模式空间附加到保持空间。
3. `g`。将保持空间复制到保持空间。
4. `G`。将保持空间附加到模式空间。
5. `x`。交换模式空间和保持空间的内容。

可以灵活使用上面的命令，例如

```bash
$ sed -n '/first/{h;n;p;g;p}' data2.txt
This is the second data line.
This is the first data line.
$ 
```

可以达成翻转输出的效果。

### 排除命令

排除命令`!`，让原本会起作用的命令不起作用（反作用）。下面时一个例子。

```bash
$ sed -n '/header/!p' data2.txt
This is the first data line.
This is the second data line.
This is the last data line.
$ 
```

看一下之前的例子的改写

```bash
$ sed '$!N
> s/System\nAdministrator/Desktop\nUser/
> s/System Administrator/Desktop User/
> ' data4.txt
On Tuesday, The Linux Desktop
User's group meeting will be held.
All Desktop Users should attend.
$ 
```

这里使用了`$!N`表示在最后一行时不会执行`N`，后面的命令也不会因为`N`命令的失败而停止。

下面时翻转输出的例子

```bash
$ cat data2.txt
This is the header line.
This is the first data line.
This is the second data line.
This is the last data line.
$ sed -n '{1!G;h;$p}' data2.txt
This is the last data line.
This is the second data line.
This is the first data line.
This is the header line.
$ 
```

当然对于翻转文件已经存在已有的命令`tac`了（正好时`cat`反过来写）。

### 改变流

通常，`sed`编辑器会从脚本的顶部开始，一直执行到脚本的结尾（`D`命令是个例外）。`sed`编辑器提供了一个方法来改变脚本的执行流程，其结果与结构化编程类似。

#### 分支

`sed`编辑器提供了一种方法，可以基于地址、地址模式或地址区间排除一整块命令。这允许只对数据流中的特定行执行一组命令。

分支命令`b`的格式如下：

```bash
[address]b [label]
```

其中`address`命令决定了哪些行的数据会触发分支命令。`label`参数定义了要跳转到的位置。如果没有`label`参数，会之间跳转脚本的结尾。

```bash
$ sed '{2,3b; s/This is/Is this/; s/line./test?/}' data2.txt
Is this the header test?
This is the first data line.
This is the second data line.
Is this the last data test?
$ 
```

如果不想直接跳转到脚本的结尾，则需要定义一个跳转的标签。从冒号开始，最多7个字符长度。

```bash
$ sed '{/first/b jump1; s/This is the/No jump on/;:jump1;s/This is the/Jump here on/}' data2.txt
No jump on header line.
Jump here on first data line.
No jump on second data line.
No jump on last data line.
$ 
```

如果匹配了，分支语句会跳转到标签处，如果没有匹配则会一直运行下去（包括标签后的语句）。

同时，可以使用分支语句达到循环的效果。

```bash
$ echo "This, is, a, test, to, remove, commas." | sed -n '{:start;s/,//1p;/,/b start}'
This is, a, test, to, remove, commas.
This is a, test, to, remove, commas.
This is a test, to, remove, commas.
This is a test to, remove, commas.
This is a test to remove, commas.
This is a test to remove commas.
$ 
```

#### 测试

测试命令`t`也可以用来改变`sed`编辑器脚本的执行流程。测试命令会根据替换命令的结果跳转到某个标签，而不是根据地址进行跳转。如果替换命令成功匹配并替换了一个模式，测试命令就会跳转到指定的标签。测试命令和分支命令有相同的格式。

测试命令提供了一种简单的`if-then`语句。例如，已经做了一个替换，不需要再做另一个替换，那么就可以使用测试命令。

```bash
$ sed '{
> s/first/matched/
> t
> s/This is the/No match on/
> }' data2.txt
No match on header line.
This is the matched data line.
No match on second data line.
No match on last data line.
$ echo "This, is, a, test, to, remove, commas." | sed -n '{
> :start
> s/,//1p
> t start
> }'
This is, a, test, to, remove, commas.
This is a, test, to, remove, commas.
This is a test, to, remove, commas.
This is a test to, remove, commas.
This is a test to remove, commas.
This is a test to remove commas.
$ 
```

### 模式替代

#### `&`符号

`&`符号可以用来代替替换命令中匹配的模式。

```bash
$ echo "The cat sleeps in his hat." | sed 's/.at/"&"/g'
The "cat" sleeps in his "hat".
$ 
```

#### 替换单独的单词

使用`\(\)`和`\1\2`组合。

```bash
$ echo "The System Administrator manual" | sed '
> s/\(System\) Administrator/\1 User/'
The System User manual
$ echo "1234567" | sed '{
> :start
> s/\([0-9]\+\)\([0-9]\{3\}\)/\1,\2/
> t start}'
1,234,567
$ 
```

### 在脚本中使用`sed`

#### 使用包装脚本

实现`sed`编辑器脚本的过程很繁琐，可以将`sed`编辑器命令放到shell包装脚本中。不用每次都重新键入`sed`脚本。

#### 重定向`sed`输出

可以在脚本中使用`$()`将`sed`的输出保持在变量中，以备后用。

## `gawk`进阶

### 使用变量

`gawk`编程语言支持两种不同类型的变量：

1. 内建变量
2. 自定义变量

`gawk`有一些内建变量。这些变量存放用来处理数据文件中的数据字段和记录的信息。也可以在`gawk`程序中创建自己的变量。

#### 内建变量

`gawk`程序使用内建变量来引用程序数据里的一些特殊功能。

1. 字段和记录分隔符变量。数据字段是由字段分隔符来划定的。默认情况下，字段分隔符是一个空白字符（空格或者制表符）。下面是`gawk`数据字段和记录变量

   |     变量      |                       描述                       |
   | :-----------: | :----------------------------------------------: |
   | `FIELDWIDTHS` | 由空格分割的一列数字，定义了每个数据字段确切宽度 |
   |     `FS`      |                  输入字段分隔符                  |
   |     `RS`      |                  输入记录分隔符                  |
   |     `OFS`     |                  输出字段分隔符                  |
   |     `ORS`     |                  输出记录分隔符                  |

   默认情况下，`OFS`是一个空格，如果输入`print $1,$2,$3`就会看到`data1 data2 data3`。

   ```bash
   $ cat data1
   data11,data12,data13,data14,data15
   data21,data22,data23,data24,data25
   data31,data32,data33,data34,data35
   $ gawk 'BEGIN{FS=","; OFS="--"} {print $1,$2,$3}' data1
   data11--data12--data13
   data21--data22--data23
   data31--data32--data33
   $ 
   ```

   `FIELDWIDTHS`变量运行不依赖字段分隔符来读取记录。在一些应用程序中，数据并没有使用字段分隔符，而是被放置在了记录中的特定列。这种情况下，必须设定`FIELDWIDTHS`变量来匹配数据在记录中的位置。

   ```bash
   $ cat data1b
   1005.3247596.37
   155-2.349194.00
   05810.1298100.1
   $ gawk 'BEGIN{FIELDWIDTHS="3 5 2 5"} {print $1,$2,$3,$4}' data1b
   100 5.324 75 96.37
   155 -2.34 91 94.00
   058 10.12 98 100.1
   $ 
   ```

   一旦设定了`FIELDWIDTHS`的值，就不能再改变了，这种方法不适应于变长字段。

   变量`RS`和`ORS`定义了`gawk`程序如何处理数据流中的记录。默认情况下，`gawk`将`RS`和`ORS`设为换行符。默认的`RS`值表明，输入数据流中的每行新文本就是一条新纪录。

2. 数据变量。除了字段和记录分隔符变量外，`gawk`还提供了其他一些内建变量来帮助了解数据发生了什么变化，并提取shell环境的信息。

   |     变量     |                        描述                         |
   | :----------: | :-------------------------------------------------: |
   |    `ARGC`    |                 当前命令行参数个数                  |
   |   `ARGIND`   |              当前文件在`ARGV`中的位置               |
   |    `ARGV`    |                包含命令行参数的数组                 |
   |  `CONVFMT`   |             数字的转换格式，默认`%.6g`              |
   |  `ENVIRON`   |        当前shell环境变量及其值组成的关联数组        |
   |   `ERRNO`    |       当读取或关闭文件发生错误时的系统错误号        |
   |  `FILENAME`  |             用作输入的数据文件的文件名              |
   |    `FNR`     |           当前数据文件中已处理过的记录数            |
   | `IGNORECASE` | 设置为非0值的时候，忽略`gawk`命令中出现的字符大小写 |
   |     `NF`     |                数据文件中的字段总数                 |
   |     `NR`     |                 已处理的输入记录数                  |
   |    `OFMT`    |           数字的输出格式，默认值为`%.6g`            |
   |  `RLENGTH`   |           由`match`函数所匹配的字符串长度           |
   |   `RSTART`   |         由`match`函数所匹配的字符串起始位置         |

   其中，`ARGC`和`ARGV`变量允许从shell中获得命令行参数的总数已经它们的值。注意`gawk`并不会把程序脚本当作命令参数的一部分。同时注意在`gawk`中引用变量并不需要`$`符号。

   ```bash
   $ gawk 'BEGIN{print ARGC,ARGV[1]}' data1
   2 data1
   $ 
   ```

   `ENVIRON`变量中存储了当前shell的环境变量，使用字符串作为索引。

   ```bash
   $ gawk 'BEGIN{print ENVIRON["HOME"]}'
   /home/yiming
   $ 
   ```

   要在程序中跟踪数据字段和记录时，变量`FNR`、`NF`和`NR`用起来比较方便。`NF`变量可以在不知道具体位置的情况下指定记录中的最后几个字段。

   ```bash
   $ gawk 'BEGIN{FS=":";OFS=":"} {print $1,$(NF-1)}' /etc/passwd
   root:/root
   daemon:/usr/sbin
   bin:/bin
   sys:/dev
   sync:/bin
   games:/usr/games
   [...]
   $ 
   ```

   `FNR`变量含有当前数据文件中已处理过的记录数，`NR`变量包含已处理的输入记录数。

   ```bash
   $ gawk 'BEGIN{FS=","} {print $1,"FNR="FNR,"NR="NR}' data1 data1
   data11 FNR=1 NR=1
   data21 FNR=2 NR=2
   data31 FNR=3 NR=3
   data11 FNR=1 NR=4
   data21 FNR=2 NR=5
   data31 FNR=3 NR=6
   $ 
   ```

#### 自定义变量

`gawk`自定义的变量名可以是任意数目的字母、数字和下划线，但不能以数字开头且区分大小写。

1. 在脚本中给变量赋值。在`gawk`中给变量赋值和在shell脚本中赋值类似。同时，在`gawk`的赋值语句中可以使用算术运算来处理数字值（数学运算包括`%`取模和`^ **`求幂）。

   ```bash
   $ gawk 'BEGIN{testing="This is a test";print testing;testing=45;print testing}'
   This is a test
   45
   $ gawk 'BEGIN{x=4;x=x*2+3;print x}'
   11
   $ 
   ```

2. 在命令行上给变量赋值。在命令行进行赋值，允许你在正常代码之外赋值，即时改变变量的值。

   ```bash
   $ cat script1
   BEGIN{FS=","}
   {
   print $n
   n=1
   print $n
   }
   $ gawk -f script1 n=3 data1
   data13
   data11
   data21
   data21
   data31
   data31
   $ 
   ```

   这里一开始`n=3`，是由命令行设置的。

   但是像上面这种赋值方式有一个问题，就是无法在`BEGIN`部分使用变量。

   ```bash
   $ cat script1
   BEGIN{
   FS=","
   print n
   }
   {
   print $n
   }
   $ gawk -f script1 n=3 data1
   
   data13
   data23
   data33
   $ 
   ```

   解决方式是使用`-v`选项。`-v`命令行参数必须放在脚本代码之前。

   ```bash
   $ gawk -v n=3 -f script1 data1
   3
   data13
   data23
   data33
   $ 
   ```

### 处理数组

#### 定义数组变量

可以用标准赋值语句来定义数组变量。数组变量赋值的格式如下

```bash
var[index] = element
```

其中`var`是变量名，`index`是关联数组的索引值，`element`是数据元素值。下面是一些例子

```bash
$ gawk 'BEGIN{capital["Illinois"]="Springfield";print capital["Illinois"]}'
Springfield
$ gawk 'BEGIN{var[1]=34;var[2]=3;total=var[1]+var[2];print total}'
37
$ 
```

#### 遍历数组变量

如果要在`gawk`中遍历一个关联数组，可以用`for`语句的一种特殊形式

```bash
for (var in array)
{
	statements
}
```

要注意这里的`var`中存的是数组中的索引值。可以根据这个索引值得到数组中的数据

```bash
$ gawk 'BEGIN{
> var["a"]=1
> var["g"]=2
> var["m"]=3
> var["u"]=4
> for (test in var)
> {
> print test,"-",var[test]
> }
> }'
u - 4
m - 3
a - 1
g - 2
$ 
```

注意这里的索引值返回顺序不一定。

#### 删除数组变量

从关联数组中删除数组索引要使用特殊命令

```bash
delete array[index]
```

### 使用模式

#### 正则表达式

`gawk`可以使用正则表达式来选择程序脚本作用在数据流的哪些行上。在使用正则表达式时，正则表达式必须出现在它想要控制的程序脚本的左花括号前。

```bash
$ gawk 'BEGIN{FS=","} /11/{print $1}' data1
data11
$ 
```

注意这种方式下正则表达式也会匹配分隔符。

```bash
$ gawk 'BEGIN{FS=","} /,d/{print $1}' data1
data11
data21
data31
$ 
```

#### 匹配操作符

匹配操作符允许将正则表达式限定在记录中的特定数据字段。匹配操作符是`~`。可以指定匹配操作符、数据字段变量以及要匹配的正则表达式。

```bash
$ gawk -F, '$2 ~ /^data2/{print $0}' data1
data21,data22,data23,data24,data25
$ 
```

这个例子是在第二个字段匹配`^data2`。

同时也可以使用`!`修饰`~`代表不匹配。

```bash
$ gawk -F, '$2 !~ /^data2/{print $0}' data1
data11,data12,data13,data14,data15
data31,data32,data33,data34,data35
$ 
```

#### 数学表达式

除了正则表达式，也可以在匹配模式中使用数学表达式。

```bash
$ gawk -F: '$4 == 0{print $1}' /etc/passwd
root
$ gawk -F: '$4 != 0{print $1}' /etc/passwd
daemon
bin
sys
sync
$ 
```

这里的`$4 == 0`就是一个数学表达式，表示第四个字段等于0的行。这里的数学表达式包括各种形式：`== != <= >= < >`。

当然也可以对文本数据使用表达式，和正则表达式不同，这里的表达式必须完全匹配。

```bash
$ gawk -F, '$1 == "data11"{print $1}' data1
data11
$ gawk -F, '$1 == "data"{print $1}' data1
$ 
```

### 结构化命令

#### `if`语句

```bash
if (condition)
	statement1
```

使用方式和c语言相似。

```bash
$ cat data3
Jones 2143 78 84 77
Gondrol 2321 56 58 45
RinRao 2122 38 37
Edwin 2537 87 97 95
Dayan 2415 30 47
$ cat script2.gawk
{
total=$3+$4+$5
avg=total/3
if(avg>=90){
        grade="A"
}else if(avg>=80){
        grade="B"
}else if(avg>=70){
        grade="C"
}else if(avg>=0 && avg<70){
        grade="D"
}
print $0,"=>",grade
}
$ gawk -f script2.gawk data3
Jones 2143 78 84 77 => C
Gondrol 2321 56 58 45 => D
RinRao 2122 38 37 => D
Edwin 2537 87 97 95 => A
Dayan 2415 30 47 => D
$ 
```

另外`gawk`中也有`?`命令。

#### `while`语句

```bash
while (condition)
{
	statements
}
```

使用方式和C语言一样。

#### `do-while`语句

```bash
do{
	statements
} while (condition)
```

使用方式和C语言一样。

#### `for`语句

```bash
for (variable assignment; condition; iteration process)
```

使用方式和C语言一样。

### 格式化打印

`gawk`有类似c语言中的`printf`命令来进行格式化打印。`printf`的命令格式如下

```bash
printf "format string", var1, var2, ...
```

和c语言的`printf`一样，`format string`包含文本元素和格式化指定符。格式化指定符采用如下格式

```bash
%[modifier]control-letter
```

其中`control-letter`是一个单字符代码，用于指明显示什么类型的数据，而`modifier`则定义了可选的格式化特性。下面列出了控制字母

| 控制字母 |                描述                |
| :------: | :--------------------------------: |
|   `c`    |     将一个数作为ASCII字符显示      |
|   `d`    |              显示整数              |
|   `i`    |              显示整数              |
|   `e`    |           显示科学计数法           |
|   `f`    |             显示浮点数             |
|   `g`    | 显示科学计数法或浮点数（选择短的） |
|   `o`    |             显示8进制              |
|   `s`    |             显示字符串             |
|   `x`    |            显示十六进制            |
|   `X`    |        显示十六进制（大写）        |

还有3种修饰符可以进一步控制输出：

| 修饰符  |                             描述                             |
| :-----: | :----------------------------------------------------------: |
| `width` | 指定了输出字段最小宽度。是一个数字。如果输出短于这个值，输出右对齐，空格填充。 |
| `prec`  | 指定了浮点数种小数点后的位数。是一个数字。或者字符串显示的最大字符数。 |
|   `-`   |                 表明采用左对齐而不是右对齐。                 |

下面是一个例子

```bash
$ cat data5
130 120 135
160 113 140
145 170 215
$ gawk '{
total=0
for(i=1;i<=4;i++){
total+=$i
}
avg=total/3
printf "Avg: %06.1f\n", avg
}' data5
Avg: 0128.3
Avg: 0137.7
Avg: 0176.7
$ 
```

这里`%5.1f`代表输出一个浮点数，`width=6`并且用0填充，`prec=1`。

### 函数

#### 定义函数

```bash
function name([variables])
{
	statements
}
```

函数的使用方式基本和c语言完全一致

#### 使用自定义函数

和c语言调用函数的方式完全一致。

如果需要创建函数库，则需要先创建一个文件储存需要的函数，例如`functionlib`，需要使用这个函数库时，使用下面的方式调用

```bash
$ gawk -f functionlib -f script data
```

#### 内建函数

除了自定义函数，`gawk`内部有很多内建函数

1. 数学函数。

   |     函数     |         描述         |
   | :----------: | :------------------: |
   | `atan2(x,y)` |    `x/y`的反正切     |
   |   `cos(x)`   |                      |
   |   `exp(x)`   |                      |
   |   `int(x)`   |                      |
   |   `log(x)`   |      求自然对数      |
   |   `rand()`   | 生成0~1的随机浮点数  |
   |   `sin(x)`   |                      |
   |  `sqrt(x)`   |                      |
   |  `srand(x)`  | 为随机数计算指定种子 |

   除此之外，还有一些位运算相关的函数。

   |        函数         | 描述 |
   | :-----------------: | :--: |
   |    `and(v1,v2)`     | `&`  |
   |    `compl(val)`     | `~`  |
   | `lshift(val,count)` | `<<` |
   |     `or(v1,v2)`     | `\|`  |
   | `rshift(val,count)` | `>>` |
   |    `xor(v1,v2)`     | `^`  |

2. 字符串函数。

   |             函数             |                             描述                             |
   | :--------------------------: | :----------------------------------------------------------: |
   |       `asort(s [,d])`        | 将数组`s`按照元素值排序。索引值会替换成表示新的排序顺序的连续数字。如果指定了`d`排序后的数组会存在`d`中 |
   |       `asorti(s [,d])`       | 将数组`s`按照索引值排序。元素值会替换成表示新的排序顺序的连续数字，如果指定了`d`排序后的数组会存在`d`中 |
   |    `gensub(r, s, h [,t])`    | 在`$0`或目标字符串`t`中匹配`r`，如果`h`是`g`或者`G`则全部替换，如果是一个数字则替换第`h`次出现的地方 |
   |      `gsub(r, s [,t])`       |       在`$0`或目标字符串`t`中匹配`r`，并用`s`替换全部        |
   |        `index(s, t)`         |                 返回字符串`t`在`s`中的索引值                 |
   |        `length([s])`         |            求长度，如果没有指定`s`则返回`$0`长度             |
   |      `match(s, r [,a])`      | 返回`s`中正则表达式`r`出现的位置的索引。如果指定了数组`a`，它会储存`s`中匹配的那部分 |
   |      `split(s, a [,r])`      |        将`s`用`FS`或`r`分割到数组`a`中。返回字段总数         |
   | `sprintf(format, variavles)` |                      `printf`返回字符串                      |
   |       `sub(r, s [,t])`       |      在`$0`或目标字符串`t`中匹配`r`，并用`s`替换第一处       |
   |     `substr(s, i [,n])`      | 返回`s`从索引`i`开始（索引从1开始）`n`个的子字符串，如果没有`n`就到结尾 |
   |         `tolower(s)`         |                            转小写                            |
   |         `toupper(s)`         |                            转大写                            |

3. 时间函数。

   |              函数              |                          描述                          |
   | :----------------------------: | :----------------------------------------------------: |
   |       `mktime(datespec)`       | 将一个`YYYY MM DD HH MM SS[DST]`格式的日期转化为时间戳 |
   | `strftime(format[,timestamp])` |       将当前时间的时间戳或`timestamp`转化为日期        |
   |          `systime()`           |                    当前时间的时间戳                    |

   

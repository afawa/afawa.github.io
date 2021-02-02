---
title: 'sed,gawk基础与正则表达式'
date: 2021-01-19 22:12:17
tags: linux command line
categories: linux command line
toc: true
---
# `sed`基础，`gawk`基础，正则表达式

## `sed` 编辑器

### `sed`简单应用

`sed`编辑器被称为流编辑器，流编辑器会在编辑器处理数据之前基于预先提供的一组规则来编辑数据流。`sed`可以根据命令来处理数据流中的数据，这些命令要么从命令行中输入，要么存储在一个命令文本文件中。`sed`编辑器会执行下列操作。

1. 一次从输入中读取一行数据
2. 根据所提供的编辑器命令匹配数据
3. 按照命令修改流中的数据
4. 将新的数据输出到`STDOUT`

<!--more-->

`sed`命令的格式如下

```bash
sed options script file
```

根据选项可以修改`sed`命令的行为，

1. `-n`。不产生命令输出，使用`print`命令来完成输出。
2. `-e script`。在处理输入时，将script中指定的命令添加到已有的命令中。（命令行输入，多个`-e script`组合）
3. `-f file`。在处理输入时，将file中指定的命令添加到已有的命令中。（文件输入）
4. `-i`。之间编辑文件命令。

#### 在命令行定义编辑器命令

默认情况下，`sed`编辑器会将指定的命令应用到`STDIN`输入流上。可以直接通过管道输入`sed`编辑器进行处理。

```bash
$ echo "This is a test" | sed 's/test/big test/' 
This is a big test
$ 
```

在上面的示例中使用管道对`sed`编辑器输入文本，然后使用`s`命令对于文本进行替换，这里的`s/test/big test/`代表把test全部替换成big test。

`sed`编辑器不会修改文本文件的数据，它只会将修改后的数据发送到`STDOUT`中。

#### 在命令行使用多个编辑器命令

要在`sed`命令行上执行多个命令时，使用`-e`选项

```bash
$ cat data1.txt
The quick brown fox jumps over the lazy dog.
The quick brown fox jumps over the lazy dog.
The quick brown fox jumps over the lazy dog.
The quick brown fox jumps over the lazy dog.
$ sed -e 's/brown/green/; s/dog/cat/' data1.txt
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
$ sed -e 's/brown/green/' -e 's/dog/cat/' data1.txt
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
$ sed -e '
> s/brown/green/
> s/dog/cat/
> ' data1.txt
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
```

注意使用分号时在命令末尾和分号之间不能有空格。如果使用多行模式必须在封尾单引号所在行结束命令。

#### 从文件读取编辑器命令

如果有大量要处理的`sed`命令，可以使用`-f`选项从文件中读取。

```bash
$ cat script1.sed
s/brown/green/
s/dog/cat/
$ sed -f script1.sed data1.txt
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
The quick green fox jumps over the lazy cat.
```

### `sed`编辑器基础

#### 更多的替换选项

1. 替换标记。上面使用的默认形式`s/dog/cat/`这种形式默认只会替换每行的第一个出现的`dog`。事实上，要让替换命令能够替换一行中不同地方出现的文本必须使用替换标记。替换标记会在替换命令字符串之后设置。

   ```bash
   s/pattern/replacement/flags
   ```

   有四种可用的替换标记

   1. 数字，表明新文本将替换第几次模式匹配的地方。
   2. `g`，表明新文本会替换所有的匹配文本。
   3. `p`，表明原先行的内容要打印出来。
   4. `w file`，将替换的结果写到文件中。

   ```bash
   $ cat data4.txt
   This is a test of the test script
   This is the second test of the test script
   $ sed 's/test/trail/2' data4.txt
   This is a test of the trail script
   This is the second test of the trail script
   $ sed 's/test/trail/g' data4.txt
   This is a trail of the trail script
   This is the second trail of the trail script
   $ sed 's/test/trail/gw temp' data4.txt
   This is a trail of the trail script
   This is the second trail of the trail script
   $ cat temp
   This is a trail of the trail script
   This is the second trail of the trail script
   ```

   `p`替换标记会输出修改过的行，一般和`sed`的`-n`选项一起使用。注意`sed`编辑器的正常输出在`STDOUT`中，只有那些包含匹配模式的行才会被`w`替换选项保存。

2. 替换字符。替换文件中的路径名可能会非常麻烦。例如

   ```bash
   $ sed 's/\/bin\/bash/\/bin\/csh/' /etc/passwd
   ```

   要解决这个问题，`sed`编辑器允许选择其他字符来作为替换命令的字符串分隔符

   ```bash
   $ sed 's!/bin/bash!/bin/csh!' /etc/passwd
   ```

#### 使用地址

默认情况下，`sed`会处理所有行。如果只想命令作用于特定行或某些行，则必须使用行寻址。有两种行寻址方式。两种形式都使用相同的格式来指定地址。

```bash
[address]command

address {
	command1
	command2
	...
}
```

1. 以数字形式表示行区间。

   1. 用单个行号指定一行。

      ```bash
      $ sed '2s/dog/cat/' data1.txt
      The quick brown fox jumps over the lazy dog.
      The quick brown fox jumps over the lazy cat.
      The quick brown fox jumps over the lazy dog.
      The quick brown fox jumps over the lazy dog.
      $ 
      ```

   2. 使用行号区间，起始行号+逗号+结束行号。

      ```bash
      $ sed '1,3s/dog/cat/' data1.txt
      The quick brown fox jumps over the lazy cat.
      The quick brown fox jumps over the lazy cat.
      The quick brown fox jumps over the lazy cat.
      The quick brown fox jumps over the lazy dog.
      $ sed '2,$s/dog/cat/' data1.txt
      The quick brown fox jumps over the lazy dog.
      The quick brown fox jumps over the lazy cat.
      The quick brown fox jumps over the lazy cat.
      The quick brown fox jumps over the lazy cat.
      $ 
      ```

      注意这里可可用使用`$`来代表最后一行。

2. 用文本模式过滤。`sed`编辑器允许指定文本模式来过滤出命令要作用的行。格式如下：

   ```bash
   /pattern/command
   ```

   这里必须使用正斜线把`pattern`包裹起来。

   ```bash
   $ sed '/quick/{s/dog/cat/; s/fox/tiger/}' data1.txt
   The quick brown tiger jumps over the lazy cat.
   The quick brown tiger jumps over the lazy cat.
   The quick brown tiger jumps over the lazy cat.
   The quick brown tiger jumps over the lazy cat.
   ```

   这里的`pattern`都是采用正则表达式匹配的。

   同时这种方式也可用使用区间模式。

   ```bash
   /pattern1/,/pattern2/command
   ```

   这种情况下会先找到`pattern1`作为开始模式，找到后对于后面紧跟的所有行都执行命令`command`，直到遇到`pattern2`命令结束，特别需要小心的一点是可能有多个区间被匹配到，造成不可知的结果，尽量少用。

3. 命令组合。下面是一个组合命令

   ```bash
   $ sed '/quick/{s/dog/cat/; s/fox/tiger/};2,3{s/cat/dog/;s/jump/run/}' data1.txt
   The quick brown tiger jumps over the lazy cat.
   The quick brown tiger runs over the lazy dog.
   The quick brown tiger runs over the lazy dog.
   The quick brown tiger jumps over the lazy cat.
   ```

#### 删除行

删除命令`d`它常常和上面的行寻址配合起来使用。例如

```bash
$ cat data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
This is line number 4.
$ sed '3d' data6.txt
This is line number 1.
This is line number 2.
This is line number 4.
```

#### 插入和附加文本

`sed`有两个插入操作。

1. `i`命令会在指定行前增加一个新行。
2. `a`命令会在指定行后增加一个新行。

格式如下：

```bash
sed '[address]command\
new line'
```

下面是一个例子。

```bash
$ sed '3a\
> insert1
> 4i\
> intert2' data6.txt
This is line number 1.
This is line number 2.
This is line number 3.
insert1
intert2
This is line number 4.
$ 
```

从这个例子可以发现插入的语句是不会进入行计算里面的。（因为这里的`This is line number 4.`一直被当作第4行）

#### 修改行

`sed`中使用`c`命令来对行进行修改。格式和上面的插入相似，需要单独指定新行。

```bash
$ sed '3c\
> New Line' data6.txt
This is line number 1.
This is line number 2.
New Line
This is line number 4.
$ 
```

注意对于修改行命令我们也可以使用行区间的模式，但是`c`命令会把所有行替换为一行。

```bash
$ sed '2,$c\
> New Line' data6.txt
This is line number 1.
New Line
$
```

#### 转换命令

转换命令`y`是唯一可以处理单个字符的`sed`编辑器命令。格式如下

```bash
[address]y/inchars/outchars/
```

`y`会对`inchars`和`outchars`进行一一映射，注意`inchars`和`outchars`长度需要相等。下面是一个例子

```bash
$ sed 'y/123/789/' data6.txt
This is line number 7.
This is line number 8.
This is line number 9.
This is line number 4.
$ 
```

#### 打印

有三个命令能够用来打印数据流中的信息：

1. `p`命令用来打印文本行。`p`命令一般和`-n`选项组合，配合行寻址。例子如下

   ```bash
   $ sed -n '/3/{p;s/line/test/p}' data6.txt
   This is line number 3.
   This is test number 3.
   $ 
   ```

2. `=`命令来打印行号。等号命令会打印行在数据流中的当前行号。例子如下

   ```bash
   $ sed -n '3{=;p;s/line/test/p}' data6.txt
   3
   This is line number 3.
   This is test number 3.
   $ sed '=' data6.txt
   1
   This is line number 1.
   2
   This is line number 2.
   3
   This is line number 3.
   4
   This is line number 4.
   $ 
   ```

3. `l`命令用来列出行。`l`命令可以打印数据流中不可打印的`ASCII`字符。任何不可打印字符要么在其八进制前加一个反斜线，要么使用C分格命名法（例如`\t`）。

#### `sed`处理文件

1. 写入文件。使用`w`命令可以写入文件。格式如下

   ```bash
   [address]w filename
   ```

   将对应的行写入到文件中。

2. 读取文件。`r`命令允许将一个独立文件的数据插入到数据流中。格式如下

   ```bash
   [address]r filename
   ```

   这个命令如果不加地址会把`r`命令指定的文件内容插入到数据流的每一行后。例子如下

   ```bash
   $ sed '/1/r data1.txt' data6.txt
   This is line number 1.
   The quick brown fox jumps over the lazy dog.
   The quick brown fox jumps over the lazy dog.
   The quick brown fox jumps over the lazy dog.
   The quick brown fox jumps over the lazy dog.
   This is line number 2.
   This is line number 3.
   This is line number 4.
   $ 
   ```

## `gawk` 程序

虽然`sed`编辑器时非常方便自动修改文本文件的工具，但也有自身的限制。`gawk`能提供一个类编程环境来修改和重新组织文件中的数据。`gawk`提供了一种编程语言，可以做如下的事：

1. 定义变量来保存数据。
2. 使用算术和字符串操作来处理数据。
3. 使用结构化编程（例如`if-then`语句和循环）。
4. 通过提取数据文件中的数据元素，将其重新排序或格式化，生成格式化报告。

`gawk`程序的报告生成能力通常用来从大文件文本中提取数据元素，并将它们格式化成可读的报告。其中最完美的例子是格式化日志文件。

### `gawk`简单应用

#### `gawk`命令格式

`gawk`程序的基本格式如下：

```bash
gawk options program file
```

下面是`gawk`程序的一些可用选项

1. `-F fs`。指定行中划分数据字段的分段分隔符。
2. `-f file`。从指定文件中读取程序。
3. `-v var=value`。定义`gawk`程序中的一个变量及其默认值。
4. `-mf N`。指定要处理的数据文件中的最大字段数。
5. `-mr N`。指定数据文件中的最大数据行数。
6. `-W keyword`。指定`gawk`的兼容模式或警告等级。

`gawk`的强大之处在于程序脚本。可以写脚本来读取文本行的数据，然后处理并显示数据，创建任何类型的输出报告。

#### 从命令行读取程序脚本

`gawk`程序脚本用一对花括号来定义。必须将脚本命令放到两个花括号中。由于`gawk`命令行假定脚本是单个文本字符串，必须将脚本放到单引号中。下面是一个例子

```bash
$ gawk '{print "Hello World!"}'
```

上面的例子使用了`print`命令，它会打印输出到`STDOUT`中，但是因为上面的命令没有指定文件，它会从`STDIN`读取输入。在运行的过程中它会一直等待`STDIN`的输入文本。如果输入一行文本然后按下回车键，`gawk`就会运行指定的脚本，在这个例子中就是打印一行`Hello World!`。

#### 使用数据字段变量

`gawk`会自动给一行中的每个数据元素分配一个变量。默认情况下，`gawk`会将如下变量分配给它在文本行中发现的数据字段。

1. `$0`代表整个文本行
2. `$1`代表文本行中的第一个数据字段
3. `$n`代表文本行中的第n个数据字段

在文本行中，每个数据字段都是通过字段分隔符划分的。`gawk`在读取一行文本时会用预定的字段分隔符划分每个数据字段。默认的时任意的空白字符。下面是一个例子

```bash
$ cat data2.txt
One line of test text.
Tow line of test text.
Three line of test text.
$ gawk '{print $1}' data2.txt
One
Tow
Three
$ gawk -F: '{print $1}' /etc/passwd
root
daemon
bin
sys
sync
games
man
lp
mail
news
uucp
[...]
```

#### 在程序脚本中使用多个命令

和`sed`类似，可以使用加分号和多行两种方式。

```bash
$ echo "My name is Rich" | gawk '{$4="Christine"; print $0}'
My name is Christine
$ echo "My name is Rich" | gawk '{
> $4="Christine"
> print $0}'
My name is Christine
```

#### 从文件中读取程序

使用`-f`命令就可以指定文件。下面是一个例子

```bash
$ cat script3.gawk
{
test = "'s home directory is "
print $1 test $6
}
$ gawk -F: -f script3.gawk /etc/passwd
root's home directory is /root
daemon's home directory is /usr/sbin
bin's home directory is /bin
sys's home directory is /dev
sync's home directory is /bin
games's home directory is /usr/games
[...]
$ 
```

这里需要注意`gawk`程序在引用变量值时并未像shell脚本一样使用美元符。

#### 在处理数据前后运行脚本

可以使用`BEGIN`关键字和`END`关键字来分别定义在数据处理前后的行为。下面时一个例子

```bash
$ cat script4.gawk
BEGIN {
print "The latest list of users and shells"
print " UserID \t Shell"
print "------- \t ---------"
FS=":"
}

{
print $1 "   \t    " $7
}

END {
print "This concludes the listing"
}
$ gawk -f script4.gawk /etc/passwd
The latest list of users and shells
 UserID          Shell
-------          ---------
root        /bin/bash
daemon              /usr/sbin/nologin
bin         /usr/sbin/nologin
sys         /usr/sbin/nologin
sync        /bin/sync
games               /usr/sbin/nologin
$ 
```

## 正则表达式

### 正则表达式类型

在linux中，有两种流行的正则表达式引擎：

1. POSIX基础正则表达式（BRE）引擎
2. POSIX扩展正则表达式（ERE）引擎

`sed`编辑器符合BRE引擎，使用ERE引擎对应的符号需要使用转义，`gawk`程序用ERE引擎来处理它的正则表达式。

### 定义BRE模式

#### 纯文本

简单的文本匹配，不包含特殊字符。

#### 特殊字符

正则表达式识别的特殊字符包括`.*[]^${}\+?|()`。如果需要以纯文本模式匹配这些字符，需要使用`\`进行转义。例如

```bash
$ echo "The cost is \$4.00" | sed -n '/\$/p'
The cost is $4.00
$ 
```

最后，尽管`/`不是一个特殊字符，但是在`sed`和`gawk`中使用时仍然需要转义。

```bash
$ echo "3 / 2" | sed -n '/\//p'
3 / 2
$ 
```

#### 锚字符

1. 锁定在行首。`^`定义从数据流中文本行的行首开始的模式，会在每个由换行符决定的新数据行的行首检查模式。如果将`^`放到模式开头之外的其他位置，那它就跟普通字符一样，不再是特殊字符。

   ```bash
   $ echo "sdas ^" | sed -n '/^/p'
   sdas ^
   $ echo "sdas ^" | sed -n '/s ^/p'
   sdas ^
   $ echo "sdas ^" | sed -n '/^s/p'
   sdas ^
   $ echo "sdas ^" | sed -n '/^a/p'
   $ 
   ```

2. 锁定在行尾。`$`定义从数据流中文本行的行首开始的模式。和`^`完全类似。

3. 组合使用。如果`^`和`$`组合使用，表示匹配只包含中间内容的行，特别的`^$`可以用来匹配空行。

#### 点号字符

`.`用来匹配除了换行符之外的任意单个字符。它必须匹配一个字符。

```bash
$ echo "at" | sed -n '/.at/p'
$ echo " at" | sed -n '/.at/p'
 at
$ 
```

#### 字符组

使用方括号`[]`来定义一个字符组。方括号中包含所有希望出现在改字符组中的字符。

```bash
$ echo "Yes" | sed -n '/[Yy][Ee][Ss]/p'
Yes
$ 
```

#### 排除型字符组

在字符组的开头加上`^`可以反转字符组，去匹配没有在字符组中出现的字符。

```bash
$ echo "hat" | sed -n '/[^ch]at/p'
$ echo "at" | sed -n '/[^ch]at/p'
$ echo " at" | sed -n '/[^ch]at/p'
 at
$ 
```

#### 区间

可以使用单破折线符号`-`在字符组中表示字符。例如`[1-9][a-g][a-ch-m]`。

#### 特殊字符组

|     组      |      描述      |
| :---------: | :------------: |
| [[:alpha:]] |    a-z,A-Z     |
| [[:alnum:]] |  0-9,a-z,A-Z   |
| [[:blank:]] | 空格或者制表符 |
| [[:digit:]] |      0-9       |
| [[:lower:]] |      a-z       |
| [[:upper:]] |      A-Z       |
| [[:print:]] | 任意可打印字符 |
| [[:punct:]] |    标点符号    |
| [[:space:]] |  任意空白字符  |



#### 星号

在字符后面放置型号`*`表明该字符必须在匹配模式的文本中出现0次或者多次（注意这里和通配符`*`是不同的，不能混淆）

```bash
$ echo "ik" | sed -n '/ie*k/p'
ik
$ echo "ieeeeek" | sed -n '/ie*k/p'
ieeeeek
```

### 扩展正则表达式

注意这里属于ERE的范畴，也就是说`sed`需要转义，`gawk`支持。

#### 问号

`?`表明前面出现的字符可以出现0次或者1次，`sed`中需要使用`\?`

```bash
$ echo "bt" | gawk '/be?t/{print $0}'
bt
$ echo "bt" | sed -n '/be\?t/p'
bt
$ 
```

#### 加号

`+`表示前面的字符出现1次或者多次，`sed`中使用`\+`

```bash
$ echo "bt" | gawk '/be+t/{print $0}'
$ echo "bet" | gawk '/be+t/{print $0}'
bet
$ echo "bet" | sed -n '/be\+t/p'
bet
$ 
```

#### 使用花括号

花括号允许为可重复的正则表达式指定一个上限。有两种形式

1. `{m}`。表示准确出现m次。
2. `{m,n}`。表示至少出现m次，至多出现n次。
3. `{m,}`。至少出现m次。

注意`gawk`程序默认不识别。必须指定`--re-interval`。注意在`sed`编辑器中是支持花括号的，只是需要使用转义`\{m\}`

```bash
$ echo "bt" | gawk --re-interval '/be{1}t/{print $0}'
$ echo "bet" | gawk --re-interval '/be{1}t/{print $0}'
bet
$ 
```

#### 管道符号

管道符号`|`相当于逻辑OR，只要两边的模式有一个匹配即可。同样`sed`中使用`\|`

```bash
$ echo "I'm a dog" | gawk '/dog|cat/{print $0}'
I'm a dog
$ echo "I'm a cat" | gawk '/dog|cat/{print $0}'
I'm a cat
$ echo "I'm a bird" | gawk '/dog|cat/{print $0}'
$ echo "I'm a dog" | sed -n '/dog\|cat/p'
I'm a dog
$ echo "I'm a cat" | sed -n '/dog\|cat/p'
I'm a cat
$ 
```

#### 表达式分组

可以使用圆括号`()`进行分组。当使用分组时，该组合被当作一个字符，可以使用`?+`等特殊字符。注意在`sed`中也是支持圆括号的，需要使用转义`\(\)`在`sed`中，会保持`\(\)`匹配的字符，可以使用`\1`进行提取，如果有多个`\(\)`都匹配成功，则使用`\2 \3`进行提取。

```bash
$ echo "Saturday" | gawk '/Sat(urday)?/{print $0}'
Saturday
$ echo "Sat" | gawk '/Sat(urday)?/{print $0}'
Sat
$ echo "Sat" | sed -n '{/Sat\(urday\)\{0,1\}/p;}'
Sat
$ echo "Saturday" | sed -n '{/Sat\(urday\)\{0,1\}/p;}'
Saturday
$ echo "{1234567}"|sed 's/{\([0-9]*\)}/\1/'
1234567
$ echo "Yes" | sed  's!\(Y\|y\)\(E\|e\)\(S\|s\)!\3!'
s
$ 
```


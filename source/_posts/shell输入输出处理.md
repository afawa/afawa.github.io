---
title: shell输入输出处理
date: 2021-01-17 17:26:42
tags: shell programming
categories: programming language
toc: true
---
## shell 处理输入输出

### 处理用户输入

#### 命令行参数

1. 读取参数。

   1. bash shell会将一些称为位置参数的特殊变量分配给输入到命令行中的所有参数。位置参数是标准的数字：`$0`是程序名，`$1`是第一个参数，一直到`$9`。如果参数不止9个，需要使用`${10}`这种方式使用参数。
<!--more-->

   2. 如果参数值中包含空格，需要使用引号来（单引号或者双引号），将文本字符串作为参数传递时，引号并非数据的一部分，只是表明数据的起止位置。

      ``` bash
      $ ./test.sh "Rich Blum"
      ```

2. 读取脚本名。可以使用$0获取shell在命令行启动的脚本名。但是这里存在一个问题，如果使用另一个命令来运行shell脚本，命令会和脚本名混在一起，出现在$0参数中。并且如果使用脚本完整路径使用脚本时，会显示脚本的绝对路径。为了只使用脚本的名称，可以使用`basename`命令。

   ``` bash
   name=$(basename $0)
   ```

3. 测试参数。需要在使用之前检查是否传入了对应量的参数。下面是一个例子。

   ``` bash
   if [ -n "$1" ]
   then
   	echo $1
   else
   	echo "not define"
   fi
   ```

#### 特殊参数变量

1. 参数统计。`$#`变量中存储了脚本运行时携带的命令行参数的个数。如果想要之间使用最后一个参数，可以使用`${!#}`来引用。
2. 抓取所有数据。`$@`和`$*`可以用来对于参数进行遍历。
   1. `$*`变量会将命令行上提供的所有参数保存为一个单词。
   2. `$@`变量会将命令行上提供的所有参数当作同一个字符串中的多个独立的单词。（用于遍历十分方便）

#### 移动变量

1. ` shift` 命令可以用来操作命令行参数，默认情况下它会将每个参数向左移动一个位置，`$3 -> $2` 并且原来的`$1`直接消失，`$0`内依然是脚本名。当不知道有多少参数时，可以用它配合`while`进行遍历。例子如下

   ``` bash
   while [ -n "$1" ]
   do
   	echo $1
   	shift
   done
   ```

   同时也可以对于`shift`指定一个数字，代表移动的数量，`shift 2`就是移动两个变量。

#### 处理选项

1. 查找选项
   1. 处理简单选项。可以使用`case`语句配合`shift`进行选项提取。例如

      ``` bash
      while [ -n "$1" ]
      do
      	case "$1" in 
      		-a) echo "Found -a" ;;
      		-b) echo "Found -b" ;;
      		*) echo "$1 is not a option" ;;
      	esac
      	shift
      done
      ```
   
2. 分离参数和选项。如果在脚本中同时使用选项和参数，Linux中处理这个问题的标准方式是用特殊字符将两者分开。对Linux来说，这个特殊字符是`--`。shell会用`--`来表明选项列表的结束，之后的都会被当作命令行参数。实现时只需要在case语句中加入对于`--`的判断。
   
      ``` bash
      while [ -n "$1" ]
	   do
      	case "$1" in 
	   		-a) echo "Found -a" ;;
      			-b) echo "Found -b" ;;
      		--) shift
      			break ;;
      		*) echo "$1 is not a option" ;;
      	esac
      	shift
      done
      
      for para in $@
      do
      	echo "Parameter $para"
      done
      ```
      
   3. 处理带值的选项。例子如下
   
      ``` bash
      while [ -n "$1" ]
      do
      	case "$1" in
      		-a) echo "Found -a" ;;
      		-b) para="$2"
      			echo "Found -b, parameter=$para"
      			shift ;;
      		--) shift
      			break ;;
      		*) echo "$1 is not a option" ;;
      	esac
      	shift
      done
      for para in $@
      do
      	echo "Parameter $para"
      done
      ```
   
      这里`-b`选项需要一个额外的参数。
   
3. 使用`getopt`命令。上面的内容无法处理更加复杂的需求，例如我们想要把多个选项放到一个参数中

   ``` bash
   $ ./test.sh -ac
   ```

   这样的操作需要使用`getopt`选项

   1. `getopt`格式。`getopt optstring parameters`，其中`optstring`定义了有效的选项字母，还定义了哪些选项需要参数值。在`optstring`中列出每个需要参数值的字母，然后再每个需要参数值的选项字母后加一个冒号。`getopt`会根据`optstring`进行解析。例子如下

      ``` bash
      $ getopt ab:cd -a -b test1 -cd test2 test3
      -a -b test1 -c -d -- test2 test3
      $
      ```

      如果指定了一个不在`optstring`中的选项，默认情况下会产生一条错误信息，

      ``` bash
      $ getopt ab:cd -a -b test1 -cde test2 test3
      getopt: invalid option -- 'e'
      -a -b test1 -c -d -- test2 test3
      $
      ```

      可以在命令后加入`-q`选项

      ``` bash
      $ getopt -q ab:cd -a -b test1 -cde test2 test3
      -a -b test1 -c -d -- test2 test3
      $
      ```

   2. 在脚本中使用`getopt`。方法是使用`getopt`命令生成的格式化后的版本来替换已有的命令行选项和参数。需要使用`set`命令。`set`命令的选项之一是`--`，它会将命令行参数替换为`set`命令的命令行值。

      ``` bash
      #!/bin/bash
      set -- $(getopt -q ab:cd "$@")
      
      while [ -n "$1" ]
      do
      	case "$1" in
      		-a) echo "Found -a" ;;
      		-b) para="$2"
      			echo "Found -b, parameter=$para"
      			shift ;;
      		-c) echo "Found -c" ;;
      		-d) echo "Found -d" ;;
      		--) shift
      			break ;;
      		*) echo "$1 is not a option" ;;
      	esac
      	shift
      done
      for para in $@
      do
      	echo "Parameter $para"
      done
      ```

      但是`getopt`任然存在一个小问题：`getopt`并不擅长处理带引号和空格的场景，对于上述代码，下面的方法可以解决这个问题。

      ``` bash
      $ ./test.sh -a -b test1 -cd "test2 test3" test4
      Found -a
      Found -b, parameter=test1
      Found -c
      Found -d
      Parameter 'test2
      Parameter test3'
      parameter 'test4'
      ```

4. `getopts`命令。与`getopt`不同，前者将命令行上选项和参数处理后只生成一个输出，而`getopts`命令能够和已有的shell参数变量配合。每次调用它时，它一次处理命令行上检测的一个参数。处理完所有参数后，它会退出并返回一个大于0的退出码。可以和循环命令结合使用。命令格式如下：
   
   ```bash
   getopts optstring variable
   ```

   其中`optstring`和`getopt`命令类似，注意如果需要去掉错误信息的话，可以在`optstring`之前加一个冒号。`getopts`命令将当前参数保存在命令行中定义的variable中。`getopts`命令会用到两个环境变量，如果选项需要跟一个参数值，`OPTARG`环境变量就会保存这个值。`OPTIND`环境变量保存了参数列表中`getopts`正在处理的参数位置。

   ``` bash
   #!/bin/bash
   while getopts :ab:c opt
   do
      case "$opt" in
      a) echo "Found -a" ;;
      b) echo "Found -b, parameter $OPTARG" ;;
      c) echo "Found -c" ;;
      *) echo "Unknown option $opt" ;;
      esac
   done
   ```

   注意对于`$OPTARG`如果包含空格需要用引号包裹。

   ``` bash
   $ ./test.sh -ab "test1 test2" -c
   Found -a
   Found -b, parameter test1 test2
   Found -c
   ```

   初次之外，`getopts`将命令行上找到的所有未定义的选项统一输出为问号。

   ``` bash
   $ ./test.sh -ab "test1 test2" -c -d
   Found -a
   Found -b, parameter test1 test2
   Found -c
   Unknown option ?
   ```

   `getopts`只会处理选项参数，可以使用`shift`结合`OPTIND`环境变量来解析后续参数。

   ``` bash
   #!/bin/bash
   while getopts :ab:c opt
   do
      case "$opt" in
      a) echo "Found -a" ;;
      b) echo "Found -b, parameter $OPTARG" ;;
      c) echo "Found -c" ;;
      *) echo "Unknown option $opt" ;;
      esac
   done
   
   shift $[ $OPTIND - 1 ]
   
   for para in "$@"
   do
      echo "Parameter $para"
   done
   ```

   这样就可以处理了，

   ``` bash
   $ ./test.sh -ab "test1 test2" -c test3 test4
   Found -a
   Found -b, parameter test1 test2
   Found -c
   Parameter test3
   Parameter test4
   $ ./test.sh -ab test1 test2 -c test3 test4
   Found -a
   Found -b, parameter test1
   Parameter test2
   Parameter -c
   Parameter test3
   Parameter test4
   ```

   不过需要注意`getopts`解析时需要把选项放在参数之前，这点和`getopt`不一样，对于`getopt`或默认把参数整理出来放在最后。

   ``` bash
   $ getopt ab:c -ab test1 test2 -c test3 test4
      -a -b test1 -c -- test2 test3 test4
   ```


#### 读取用户输入

1. 基本的读取。`read`命令从标准输入（键盘）或另一个文件描述符中接受输入。在收到输入后`read`命令会将数据存在一个变量里。

   ``` bash
   read name
   echo "I'm $name"
   ```

   `read`命令包含了`-p`选项能够在命令行指定提示符

   ``` bash
   read -p "Who are you?" name
   echo "I'm $name"
   ```

   `read`可以指定多个变量，输入的每个数值都会分配给变量列表中的下一个变量，如果变量数量不够，剩下的数据就全部分配给最后一个变量。`read`也可以不指定变量，这样的话收到的任何数据都会在特殊环境变量`REPLY`中。

2. 超时，限定数目。

   1. `-t`选项指定了`read`命令等待输入的秒数，当计时器过期之后，`read`命令返回一个非零退出码。
   2. 也可以不对输入过程计时，而是让`read`命令来统计输入的字符数。使用`-n`选项加上一个数字代表接受多少个字符。

3. 隐藏输入。使用`-s`命令避免`read`命令中输入的数据出现在显示器上。

4. 从文件中读取。每次使用`read`都会读取文件的一行，直到文件末尾，使用方式就是 `cat + 管道 + while + read` 。（`done + 重定向`也是可以的）

   ``` bash
   count=1
   cat test.txt | while read line
   do
   	echo "$count  $line"
   	count=$[ $count + 1 ]
   done
   ```



### 呈现数据

#### 输入和输出

1. 标准文件描述符。文件描述符是一个非负整数，可以唯一标识会话中打开的文件。每个进程一次最多可以有九个文件描述符。bash shell保留了前三个文件描述符。

   1. 0    `STDIN`     标准输入。
   2. 1    `STDOUT`  标准输出。当命令生成错误信息时，shell创建了输出重定向文件，但错误信息显示在显示屏上。如果使用输出重定向，此时错误信息不会出现在重定向的文件中。
   3. 2    `STDERR`   标准错误。默认情况下，`STDERR`文件描述符会和`STDOUT`文件描述符指向同样的地方（尽管分配给他们的文件描述符不同）。也就是说在默认情况下，错误信息也会显示在显示器上。但是`STDERR`不会随着`STDOUT`的重定向而发生改变。

2. 重定向错误。

   1. 只重定向错误。可以选择只重定向错误信息，讲该文件描述符值放在重定向符号前（必须紧跟）

      ``` bash
      $ ls -al badfile 2> test
      ```

      这样的话只会把错误进行重定向，标准输出还是会出现在屏幕上。

   2. 重定向错误和数据。

      1. 如果想要将两者标准输出和错误输出分开重定向。使用如下形式

         ``` bash
         $ls -al goodfile badfile 1> test1 2> test2
         ```

      2. 如果想要将两者输出一起重定向，使用`&>`符号

         ``` bash
         $ls -al goodfile badfile &> test
         ```

         使用`&>`符号时，为了避免错误信息散落在输出文件中，相较于标准输出，bash shell自动赋予了错误信息更高的优先级，重定向后所有的错误信息会集中在开头。

#### 脚本中重定向输出

1. 临时重定向。如果有意在脚本中生成错误信息，可以将单独的一行输出重定向到`STDERR`。

   ``` bash
   echo "error message" >&2
   ```

   这行把`echo`命令的输出重定向到`STDERR`中。

2. 永久重定向。可以使用`exec`命令告诉shell在脚本执行期间重定向某个特定文件描述符。

   ``` bash
   exec 2>errorfile
   echo "normal message"
   exec 1>outputfile
   echo "in output file"
   echo "in error file" >&2
   ```

#### 脚本中重定向输入

1. 可以使用与脚本中重定向`STDOUT`和`STDERR`相同的方法来将`STDIN`从键盘重定向到其他位置。`exec`命令允许你将`STDIN`重定向到Linux系统上的文件。

   ``` bash
   exec 0< testfile
   ```

#### 创建自己的重定向

1. 创建输出文件描述符。可以使用`exec`命令来给输出分配文件描述符。和标准的文件描述符一样，一旦将另一个文件描述符分配给一个文件，这个重定向就会一直有效，直到重新分配。

   ``` bash
   exec 3>testfile
   echo "test message" >&3
   ```

2. 重定向文件描述符。你可以分配另外一个描述符给标准文件描述符，反之亦然。可以使用这个方法恢复以重定向的文件描述符。

   ``` bash
   exec 3>&1
   exec 1>testfile
   echo "In file"
   exec 1>&3
   ```

   上面这个例子先把3重定向到1的当前位置，然后把1重定向到testfile文件，现在`echo`命令的输出会出现在文件中，然后再把1重定向到3的位置，也就是恢复了原来的`STDOUT`。这是一种再脚本中临时重定向输出，然后恢复默认输出设置的常用方法。

3. 创建输入文件描述符。相似的也可以临时重定向`STDIN`然后恢复

   ``` bash
   exec 6<&0
   exec 0<testfile
   while read line
   do
   	echo $line
   done
   exec 0<&6
   ```

4. 创建读写描述符。由于是对同一个文件进行读写，shell会维护一个内部指针，指明在文件中的当前位置，任何读写都会从文件指针上次的位置开始。需要特别小心，可以看下面这个例子。

   ``` bash
   $ cat test.sh
   #!/bin/bash
   exec 3<> testfile
   read line <&3
   echo "Read: $line"
   echo "This is a test line" >&3
   $ cat testfile
   This is the first line.
   This is the second line.
   This is the third line.
   $ ./test.sh
   ReAD: This is the first line.
   $ cat testfile
   This is the first line.
   This is a test line
   ine.
   This is the third line.
   ```

   可以发现在对于testfile文件进行一次读取后，指针位于第二行的开头，这时写入的话会覆盖文件第二行的内容。

5. 关闭文件描述符。如果创建了新的输入或输出文件描述符，shell会在脚本退出是自动关闭他们。如果需要手动关闭，将它重定向到特殊符号`&-`。下面的这段代码允许会报错。

   ``` bash
   exec 3>testfile
   echo "test message" >&3
   exec 3>&-
   echo "bad message" >&3
   ```

   如果随后在脚本中打开了同一个输出文件，shell会用一个新文件替换已有文件。（需要用`>>`）

#### 列出打开的文件描述符

1. `lsof`命令会列出整个linux系统打开的所有文件描述符。该命令会产生大量输出，会显示当前系统上打开的每个文件的有关信息。这包括后台运行的所有进程以及登录到系统的任何用户。有大量的参数可以过滤输出，`-p`运行输出`PID`，`-d`指定需要显示的文件描述符编号，`-a`用于对于多个选项的输出求交。

#### 阻止命令输出

1. 如果想要丢弃某些输出。例如丢弃`STDERR`的输出，可以将其重定向到`/dev/null`文件。重定向到该位置的任何数据都会被丢掉。

   ``` bash
   ls -al badfile testfile 2> /dev/null
   ```

   这种方式可以丢弃报错信息。

#### 创建临时文件

linux系统有特殊的目录，专供临时文件使用。linux使用`/tmp`目录来存放不需要永久保存的文件。大多数linux发行版配置了系统在启动时自动删除`/tmp`目录的所有文件。系统上的任何用户都有权限在读写`/tmp`目录中的文件。

`mktemp`命令可以用来创建临时文件，可以在`/tmp`目录中创建一个唯一的临时文件。会将文件的读写权限分配给文件的属主，并将用户设置为文件的属主，其他人没法访问它。

1. 创建本地临时文件。默认情况下`mktemp`会在本地创建一个文件。要用`mktemp`命令在本地目录中创建一个临时文件，只需要指定一个文件名模板就可以了。模板可以包含任意文本文件名，在文件名末尾加上6个x就行了。`mktemp`命令会用6个字符码替换这6个x，保证文件名在当前目录是唯一的。`mktemp`命令的输出是文件名（相对路径），在脚本中使用这个命令时，可能要将文件名保存在变量中，这样就可以继续引用了。

   ``` bash
   tmpfile=$(mktemp test.xxxxxx)
   exec 3>$tmpfile
   echo "first line" >&3
   exec 3>&-
   echo "file contains:"
   cat $tmpfile
   rm -f $tmpfile 2> /dev/null
   ```

2. 在`/tmp`目录创建临时文件。`-t`选项会强制`mktemp`命令在系统的临时目录来创建该文件，此时`mktemp`返回临时文件的绝对路径。

3. 创建临时目录。`-d`选项告诉`mktemp`命令创建一个临时目录。可以在这个临时目录下继续创建临时文件。

   ``` bash
   tmpdir=$(mktemp -d dir.xxxxxx)
   cd $tmpdir
   tmpfile1=$(mktemp temp.xxxxxx)
   tmpfile2=$(mktemp temp.xxxxxx)
   ```

#### 记录消息

1. 如果想要输出同时出现在显示屏和日志文件上，可以使用`tee`命令。`tee`从`STDIN`读取的数据发向两端，一端是`STDOUT`，另一端是命令行指定的文件`tee file`。

   ``` bash
   date | tee testfile
   ```

   注意默认情况tee会覆盖输出文件，如果需要追加写入需要使用`-a`选项。

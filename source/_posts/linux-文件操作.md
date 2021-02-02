---
title: linux 文件操作
date: 2021-01-28 14:57:58
tags: linux programming
categories: linux
toc: true
---
# linux 文件操作

## linux 文件结构

在linux中，一切皆文件。对应文件操作有5个基本的函数`open`，`read`，`write`，`close`，`ioctl`。linux中任何事物都可以用一个文件来表示，或者通过特殊的文件提供。下面是一些会用到的特殊文件。

<!--more-->

### 目录

目录是用于保存其他文件的节点号和名字的文件。目录文件中的每个数据项都是指向某个文件节点的链接。

![](/images/linux-文件操作/image1.png)

### 文件和设备

unix和linux中比较重要的设备文件有3个。

1. `/dev/console`。这个设备代表的是系统控制台。错误信息和诊断信息通常会被发送到这个设备。每个unix设备都会有一个指定的终端或显示屏来接收控制台消息。
2. `/dev/tty`。如果一个进程有控制终端的话，那么特殊文件`/dev/tty`就是这个控制终端。如果是系统自动允许的进程和脚本就没有控制终端，所以它们不能打开`/dev/tty`。在能够使用该设备文件的情况下，`/dev/tty`允许程序直接向用户提供输出信息，而不管用户具体使用的是什么类型的伪终端或硬件终端。`/dev/console`设备只有一个，`/dev/tty`能访问不同的物理设备。
3. `dev/null`。所有写向这个设备的输出都会被丢弃，所有读这个设备的操作会立刻得到一个文件尾标志。

## 系统调用和设备驱动程序

只需要使用少量的函数就可以对文件和设备进行控制和访问。这些函数被称为系统调用。操作系统的核心部分，内核，是一组设备驱动程序。它们是一组对系统硬件进行控制的底层接口。为了向用户提供一个一致的接口，设备驱动程序封装了所有与硬件相关的特性。硬件特有的功能通常可以用`ioctl`系统调用来提供。`/dev`目录中的设备文件的用法是相同的，它们都可以被打开、读、写和关闭。下面是用于访问设备驱动程序的底层函数（系统调用）。

1. `open`：打卡文件或者设备
2. `read`：从打开的文件或设备写数据
3. `write`：向文件或者设备写数据
4. `close`：关闭文件或设备
5. `ioctl`：把控制信息传递给设备驱动程序。它踢馆一些与特定硬件有关的必要控制，用法随设备的不同而不同。

## 库函数

直接使用底层系统调用的问题是效率低下。因为系统调用会影响系统的性能（系统调用相比一般的函数调用开销更大），硬件也会限制对底层系统调用一次所能读写的数据块的大小。linux发行版提供了一系列更高层的接口（标准函数库）。

## 底层文件访问

### `write`系统调用

系统调用`write`函数的作用是把缓冲区`buf`的前`nbytes`个字节写入与文件描述符`fildes`关联的文件中。返回实际写入的字节数。如果文件描述符有错或者底层的驱动程序对数据块长度比较敏感，该返回值可能会小于`nbytes`。如果这个函数返回0，就表示没有写入任何数据；如果返回-1，就表示`write`在执行的过程中出现了错误，错误代表保存在全局变量`errno`里。

```c
#include <unistd.h>

size_t write(int fildes, const void *buf, size_t nbytes);
```

下面是一个简单的程序

<script src="https://gist.github.com/afawa/3056d8ba48490ad8e66ee470bc53b3fd.js"></script>

输出如下：

```bash
$ ./simple_write
Here is some data
$ 
```

需要注意的是，`write`可能会报告写入的字节比要求的少，这不一定是一个错误。在程序中需要检测`errno`已发现错误，然后再次调用`write`写入剩余的数据。

### `read`系统调用

系统调用`read`的作用是：从与文件描述符`fildes`相关联的文件里读入`nbytes`个字节的数据，并把它们放到数据区`buf`中。返回实际读入的字节数。

```c
#include <unistd.h>

size_t read(int fildes, void *buf, size_t nbytes);
```

下面的程序把标准输入的前128个字符复制到标准输出。如果输入少于128个字节，就把它们全体复制过去。

<script src="https://gist.github.com/afawa/b47c490b523f9f6874528b40d16a00e7.js"></script>

运行这个程序可以看到

```bash
$ echo "Hello World" | ./simple_read
Hello World
$ ./simple_read
read from stdin
read from stdin
$ 
```

### `open`系统调用

为了创建一个新的文件描述符，需要使用系统调用`open`

```c
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>

int open(const char *path, int oflags);
int open(const char *path, int oflags, mode_t mode);
```

如果`open`调用成功，会返回一个可以被读写和使用的文件描述符。这个文件描述符数唯一的，不会和任何其他进程共享。如果两个进程打开同一个文件，那么两个进程会分别维护一个指针来代表当前读（写）操作应该从哪里开始，所有如果同时对于一个文件进行写操作，会相互覆盖。

准备打开的文件或设备的名字作为`path`传递给函数，`oflags`参数用于指定打开文件所采取的动作。下面是一些`oflags`。如果使用多个`o_flags`使用`|`进行连接。

|    模式    |                     说明                     |
| :--------: | :------------------------------------------: |
| `O_RDONLY` |                     只读                     |
| `O_WRONLY` |                     只写                     |
|  `O_RDWR`  |                     读写                     |
| `O_APPEND` |             写入时追加在文件末尾             |
| `O_TRUNC`  |              丢弃文件已有的内容              |
| `O_CREAT`  | 如果需要，就按参数`mode`中的访问模式创建文件 |
|  `O_EXCL`  |  与`O_CREAT`一起使用，确保调用者创建出文件   |

`open`如果调用成功就返回一个文件描述符，如果调用失败就返回-1，新的文件描述符总是使用未用描述符的最小值。

任何一个运行中的程序能够同时打开的文件数时有限制的。这个限制通常是由`limits.h`头文件中的常量`OPEN_MAX`定义的，它的值随系统的不同而不同，但是POSIX要求至少16（同时受本地系统全局性限制的影响）。这个限制可以在运行时调整，所有`OPEN_MAX`不是一个常量。

#### 访问权限的初始值

当使用带有`O_CREAT`的`open`调用来创建文件时，必须使用有3个参数格式的`open`调用。第三个参数`mode`时几个标志按位或后得到的，这些标志再头文件`sys/stat.h`中定义：

1. `S_IRUSR`，`S_IWUSR`，`S_IXUSR`。属主读写执行。
2. `S_IRGRP`，`S_IWGRP`，`S_IXGRP`。属组读写执行。
3. `S_IROTH`，`S_IWOTH`，`S_IXOTH`。其他用户读写执行。

下面是一个例子

```c
open("myfile", O_CREAT, S_IRUSR|S_IXOTH);
```

它创建了一个名为`myfile`的文件，文件属主有读权限，其他用户有执行权限。

有几个因素会对文件的访问权限产生影响。

1. 指定的访问权限只有在创建文件时才会使用。
2. 用户掩码（`umask`命令查看），简单来说，`open`调用里面的`mode`值会与当前用户掩码的反值做`&`操作。

### `close`系统调用

使用`close`调用终止文件描述符与其对应文件之间的关联。文件描述符被释放并且能够被重新使用。`close`调用成功时返回0，出错时返回-1。

```c
#include <unistd.h>

int close(int fildes);
```

注意在编程时检查`close`的返回值。

### `ioctl`系统调用

`ioctl`提供了一个用于控制设备及其描述符行为和配置底层服务的接口。终端、文件描述符、套接字甚至磁带机都可以有为它们定义的`ioctl`。

```c
#include <unistd.h>

int ioctl(int fildes, int cmd, ...);
```

（不做讨论了）

### 使用系统调用进行文件复制

下面是一个简单的文件复制程序

<script src="https://gist.github.com/afawa/932760eab749cca7fdca7b3f0155bc27.js"></script>

这个程序每次读入一个字节，实现文件的复制，因为系统调用本身开销很大，所以可以用如下方式进行优化。

<script src="https://gist.github.com/afawa/54924b2fce731668fb807f6d8d4bbd82.js"></script>

这里需要注意一点，`#include <unistd.h>`行必须首先出现，因为它定义的与POSIX规范有关的标志可能会影响到其他头文件。

### 其他系统调用

#### `lseek`系统调用

`lseek`系统调用对文件描述符`fildes`的读写指针进行设置。读写指针可被设置为文件的绝对位置，也可以被设置为相对于当前位置或文件尾的某个相对位置。

```c
#include <unistd.h>
#include <sys/type.h>

off_t lseek(int fildes, off_t offset, int whence);
```

其中`whence`参数定义该偏移值的用法。`whence`可以取下列值之一。

1. `SEEK_SET`：`offset`参数指定的是一个绝对位置
2. `SEEK_CUR`：`offset`参数是相对于当前位置的一个位置
3. `SEEK_END`：`offset`是相对于文件尾的一个位置

`lseek`返回从文件头到文件指针被设置处的字节偏移值，失败时返回-1。参数`offset`的类型`off_t`与具体实现有关，定义在`sys/types.h`中。

#### `fstat`，`stat`，`lstat`系统调用

`fstat`系统调用返回与打开文件描述符相关的文件的状态信息，该信息将会写到一个buf结构体中。`stat`和`lstat`会通过文件名查到对应的状态信息。如果文件是一个符号链接，`lstat`返回这个符号链接本身的信息，`stat`返回链接指向的文件的信息。

```c
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>

int fstat(int fildes, struct stat *buf);
int stat(const char *path, struct stat *buf);
int lstat(const char *path, struct stat *buf);
```

`stat`结构的成员在不同类UNIX系统上会有所变化，但一般会包括下面的内容

|  stat成员  |                     说明                     |
| :--------: | :------------------------------------------: |
| `st_mode`  |            文件权限和文件类型信息            |
|  `st_ino`  |             与该文件关联的inode              |
|  `st_dev`  |                保存文件的设备                |
|  `st_uid`  |               文件属主的UID号                |
|  `st_gid`  |               文件属主的GID号                |
| `st_atime` |            文件上一次被访问的时间            |
| `st_ctime` | 文件的权限、属主、组或内容上一次被改变的时间 |
| `st_mtime` |         文件的内容上一次被修改的时间         |
| `st_nlink` |             该文件上硬链接的个数             |

#### `dup`和`dup2`系统调用

`dup`系统调用提供了一种复制文件描述符的方法，使得多个描述符描述同一个文件。`dup`复制文件描述符`fildes`，返回一个新的描述符。`dup2`则是把一个文件描述符复制为一个新的文件描述符，如果指定的新文件描述符已经被打开了，那么就先关闭之前的文件描述符，要注意在`dup2`中关闭和打开这个操作是原子的。

```c
#include <unistd.h>

int dup(int fildes);
int dup2(int fildes, int fildes2);
```

## 标准I/O库

在标准I/O库中，与底层文件描述符对应的是流，它被实现为指向结构`FILE`的指针。

在启动程序时，有三个文件流是自动打开的，`stdin`，`stdout`，`stderr`。

### fopen函数

`fopen`库函数类似于`open`系统调用。如果需要对应设备进行明确的控制，最后使用`open`系统调用，因为这可以避免用库函数带来的一些潜在问题，如输入/输出缓冲。

```c
#include <stdio.h>

FILE *fopen(const char *filename, const char *mode);
```

`mode`参数指定文件的打开方式，

1. "r"或者"rb"，只读。文件必须存在，否则打开失败。
2. "w"或者"wb"，写方式打开，把文件长度截断为0（或者创建空文件）
3. "a"或者"ab"，写方式打开，把新内容追加在文件尾（或者创建空文件）
4. "r+"或者"rb+"，以“读写”方式打开文件。既可以读取也可以写入，也就是随意更新文件。文件必须存在，否则打开失败。
5. "w+"或者"wb+"，以“写入/更新”方式打开文件，相当于w和r+叠加的效果。既可以读取也可以写入，也就是随意更新文件。如果文件不存在，那么创建一个新文件；如果文件存在，那么清空文件内容（相当于删除原文件，再创建一个新文件）。
6. "a+"或者"ab+"，以“追加/更新”方式打开文件，相当于a和r+叠加的效果。既可以读取也可以写入，也就是随意更新文件。如果文件不存在，那么创建一个新文件；如果文件存在，那么将写入的数据追加到文件的末尾（文件原有的内容保留）。

这里的"b"代表打开的是二进制文件。

`fopen`在成功时返回一个非空的`FILE *`指针，失败时返回`NULL`，`NULL`在`stdio.h`中被定义。

可用的文件流数量和文件描述符的数量一样都是有数量限制的。实际的限制是由头文件`stdio.h`中定义的`FOPEN_MAX`定义的，它的值至少是8。

### `fread`函数

`fread`函数用于从一个文件流里读取数据。数据从文件流`stream`读到`ptr`指向的数据缓冲区中，`size`参数指定一次读取的数据记录的长度，`nitems`指定要传输的记录的个数。返回值是成功读入的记录的数目。在`fread`中并没有区分发生错误和读完流，所以两种发生时返回值都是一个小于`nitems`的数，需要使用`feof`和`ferror`来判断是否有错误发生。

```c
#include <stdio.h>

size_t fread(void *ptr, size_t size, size_t nitems, FILE *stream);
```

### `fwrite`函数

与`fread`函数几乎完全一致

```c
#include <stdio.h>

size_t fwrite(void *ptr, size_t size, size_t nitems, FILE *stream);
```

### `fclose`函数

`fclose`关闭指定文件流`stream`，使所有尚未写出的数据都写出。

```c
#include <stdio.h>

int fclose(FILE *stream);
```

### `fflush`函数

`fflush`库函数的作用是把文件流里的尚未写出的数据立刻写出。

```c
#include <stdio.h>

int fflush(FILE *stream);
```

### `fseek`函数

`fseek`和`lseek`对应，其中的`offset`参数和`whence`参数完全一致，区别是`lseek`返回一个`off_t`数值，`fssek`返回一个整数：0表示成功，-1表示失败。

```c
#include <stdio.h>

int fseek(FILE *stream, long int offset, int whence);
```

### `fgetc`，`getc`，`getchar`函数

`fgetc`函数从流中读出下一个字符返回如果到文件尾或者遇到错误则返回`EOF`（-1）,需要使用`feof`和`ferror`进行区分。`getc`的作用和`fgetc`一致，但是`getc`有可能被作为一个宏实现。`getchar`的相当于`getc(stdin)`。

```c
#include <stdio.h>

int fgetc(FILE *stream);
int getc(FILE *stream);
int getchar();
```

下面是一个使用标志输入输出库实现的复制程序。

<script src="https://gist.github.com/afawa/87cd2e75a5849aa1901c78f50a850ab5.js"></script>

### `fputc`，`putc`，`putchar`函数

这三个函数都是写入一个字符，具体差距见上面

```c
#include <stdio.h>

int fputc(int c, FILE *stream);
int putc(int c, FILE *stream);
int putchar(int c);
```

### `fgets`，`gets`函数

这两个函数从流中读取字符串

```c
#include <stdio.h>

char *fgets(char *s, int n, FILE *stream);
char *gets(char *s);
```

`fgets`函数把读到的数据写入`s`中，直到遇到下面出现的其中一种情况：

1. 遇到换行符
2. 已经读取了n-1个字符
3. 到达文件尾

它会把换行符也写入到`s`中，同时`fgets`会加上一个表示结尾的空字节`\0`表示结束所以一次调用只能最多读取n-1个字符。调用成功时，`fgets`返回一个指向`s`的指针。如果遇到`EOF`或者错误则会返回`NULL`空指针。

`gets`函数类似`fgets`，但是只会读`stdin`，并且只会遇到换行符停止，并且在写入`s`时会抛弃最后的换行符，并且`gets`很有可能导致缓冲区溢出（少用）

### 格式化输入输出

#### `printf`，`fprintf`，`sprintf`函数

函数原型

```c
#include <stdio.h>

int printf(const char *format, ...);
int sprintf(char *s, const char *format, ...);
int fprintf(FILE *stream, const char *format, ...);
```

`printf`函数输出到`stdout`，`fprintf`函数输出到指定的文件流，`sprintf`函数把自己的输出和一个结尾空字符写入字符串中。其他的`printf`系列函数可以在`man 3 printf`中查看。

普通字符在输出时不发生变化。转换控制符让`printf`取出传递过来的其他参数并对它们的格式进行编排。转换控制符总是以`%`开头。下面是一些常用的转换控制符。

1. `%d`，`%i`。十进制输出整数
2. `%o`。八进制输出
3. `%x`。十六进制输出
4. `%c`。输出一个字符
5. `%s`。输出一个字符串
6. `%f`。输出一个单精度浮点数
7. `%e`。科学计数法输出一个双精度浮点数
8. `%g`。以通用格式输出一个双精度浮点数

`printf`函数返回一个整数表示其输出的字符的个数，但是`sprintf`函数并没有把最后的`\0`计算在内。

#### `scanf`，`fscanf`，`sscanf`函数

函数原型

```c
#include <stdio.h>

int scanf(const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
int sscanf(const char *s, const char *format, ...);
```

`scanf`系列函数和`printf`相反，是读取数据到变量中，这些变量的类型必须正确，并且它们必须精确匹配格式字符串，否则内存就会被破坏。（编译器并不会报错）

使用`%[]`控制符读取由一个字符串集合中的字符构成的字符串。格式字符串`%[A-Z]`将读取一个由大写字母构成的字符串。如果想要读取一个带空格的字符串，并且在换行符处停止，可以使用`%[^\n]`来进行读取。

`scanf`函数的返回值是它成功读取的数据项的个数，如果在读取第一个数据项时失败了，返回0，如果达到了输入的结尾，就会返回`EOF`。如果文件流发生读错误，流错误标志就会被设置并且错误变量`errno`将被设置以指明错误类型。

### 其他流函数

1. `fgetpos`：获取文件流的当前（读写）位置
2. `fsetpos`：设置文件流的当前（读写）位置
3. `ftell`：返回文件流当前（读写）位置的偏移值
4. `rewind`：重置文件流内的读写位置
5. `freopen`：重新使用一个文件流
6. `setvbuf`：设置文件流的缓冲机制
7. `remove`：相当于`unlink`函数，如果`path`参数是一个目录的话，其作用就相当于`rmdir`函数

### 文件流错误

为了表明错误，许多库函数会返回一个超出范围的值，例如控制针和`EOF`。此时，错误由外部变量`errno`指出。

```c
#include <error.h>

extern int errno;
```

注意，许多函数都可能改变`errno`的值，它的值只有在函数调用失败之后才有意义。在函数使用失败后必须立刻对其进行检查。

也可以通过检查文件流的状态来确定是否发生了错误，或者是否到达了文件尾。

```c
#include <stdio.h>

int ferror(FILE *stream);
int feof(FILE *stream);
void clearerr(FILE *stream);
```

`ferror`函数测试一个文件流的错误标识，如果该标识被设置就返回一个非0值，否则返回0。

`feof`函数测试一个文件流的文件尾标识，如果该标识被设置就返回一个非0值，否则返回0。

`clearerr`函数的作用是清除由`stream`指向文件流的文件尾标识和错误标识。可以通过使用这个函数从文件流的错误中恢复。

### 文件流和文件描述符

每个文件流都和一个底层文件描述符相关联。可以把底层的输入输出操作和高层的文件流操作混合使用，一般来说这不是一个明智的做法，因为数据缓冲的后果难以预料。

```c
#include <stdio.h>

int fileno(FILE *stream);
FILE *fdopen(int fildes, const char *mode);
```

`fileno`函数可以确定文件流使用的文件描述符，如果失败就返回-1，如果需要对一个已经打开的文件流进行底层访问，这个函数很有用。`fdopen`在一个已经打开的文件描述符上创建一个新的文件流，事实上这个函数的作用是提供一个缓冲区。

## 文件和目录的维护

### `chmod`系统调用

可以通过`chmod`系统调用来改变文件或目录的访问权限。

```c
#include <sys/stat.h>

int chmod(const char *path, mode_t mode);
```

`path`参数指定的文件被修改为具有`mode`参数给出的访问权限。

### `chown`系统调用

超级用户可以使用`chown`系统调用来改变文件的属主

```c
#include <sys/type.h>
#include <unistd.h>

int chown(const char *path, uid_t owner, gid_t group);
```

### `unlink`，`link`，`syslink`系统调用

```c
#include <unistd.h>

int unlink(const char *path);
int link(const char *path1, const char *path2);
int syslink(const char *path1, const char *path2);
```

`unlink`系统调用删除一个文件的目录项并减少它的（硬）链接数。在成功时返回0，失败时返回-1。如果一个文件的链接数减少到0，并且没有进程打开它，这个文件就会被删除。`link`系统调用将创建一个指向已有文件`path1`的新链接，使用`syslink`创建符号链接。

### `mkdir`和`rmdir`系统调用

类比shell中的`mkdir`和`rmdir`程序

```c
#include <sys/types.h>
#include <sys.stat.h>

int mkdir(const char *path, mode_t mode);
```

```c
#include <unistd.h>

int rmdir(const char *path);
```

### `chdir`系统调用和`getcwd`函数

`chdir`系统调用可以用来切换目录（相当于`cd`）

```c
#include <unistd.h>

int chdir(const char *path);
```

`getcwd`函数用来确定当前的工作目录

```c
#include <unistd.h>

char *getcwd(char *buf, size_t size);
```

`getcwd`函数把当前目录的名字写到给定的缓冲区`buf`里。如果目录名的长度超出了参数`size`给出的缓冲区长度，就会返回`NULL`，如果成功，返回指针`buf`。

## 扫描目录

之前提到目录本身是一种特殊的文件，与目录操作相关的函数在`dirent.h`头文件中声明。

### `opendir`函数

```c
#include <sys/types.h>
#include <dirent.h>

DIR *opendir(const char *name);
```

`opendir`函数打开一个目录并且建立一个目录流。如果成功返回一个指向`DIR`结构的指针，如果失败返回一个空指针。注意，目录流使用一个底层的文件描述符来访问目录本身，如果打开文件过多，`opendir`可能失败。

### `readdir`函数

`readdir`函数返回一个指针，该指针指向的结构里保存着目录流`dirp`中下一个目录项的有关资料。后续的`readdir`调用将会返回后续的目录项。如果发生错误或者到达目录尾，则返回`NULL`。

```c
#include <sys/types.h>
#include <dirent.h>

struct dirent *readdir(DIR *dirp);
```

`dirent`结构中包含的目录项内容包括以下部分：

1. `ino_t d_ino`：文件的`inode`节点号
2. `char d_name[]`：文件的名字

### `telldir`，`seekdir`函数

`telldir`函数的返回值记录一个目录流的当前位置，`seekdir`函数的作用是设置目录流的目录项指针。

```c
#include <sys/types.h>
#include <dirent.h>

long int telldir(DIR *dirp);
void seekdir(DIR *dirp, long int inc);
```

### `closedir`函数

`closedir`函数关闭一个目录流并释放资源。在执行成功时返回0，发生错误时返回-1。

```c
#include <sys/types.h>
#include <dirent.h>

int closedir(DIR *dirp);
```

下面是一个目录扫描程序

<script src="https://gist.github.com/afawa/db8b061b19cda5dde96e307754813b23.js"></script>

## 错误处理

错误代码的取值和含义都在头文件`error.h`里。有两个非常有用的函数可以用来报告出现的错误。

### `strerror`函数

`strerror`函数把错误代码映射成一个字符串，该字符串对发生的错误类型进行说明。

```c
#include <string.h>

char *strerror(int errnum);
```

### `perror`函数

`perror`函数把`errno`变量中报告的错误映射到一个字符串，并输出到`stderr`中。该字符串前加上`s`中给出的信息，再加上一个冒号和一个空格。

```c
#include <stdio.h>

void perror(const char *s);
```

## `/proc`文件系统

Linux提供了一个特殊的文件系统`procfs`，它通常以`/proc`目录的形式呈现。该目录中包含了许多特殊文件用来对驱动程序和内核信息进行更高层的访问。

1. `/proc/cpuinfo`给出cpu的详细信息
2. `/proc/meminfor`给出内存使用情况
3. `/proc/version`给出内核版本情况
4. `/proc/net/sockstat`给出网络套接字的使用情况
5. `/proc/sys/fs/file-max`给出运行的程序同时能打开的文件总数。可以对其进行修改


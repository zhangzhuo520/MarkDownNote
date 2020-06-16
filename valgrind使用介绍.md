#valgrind使用介绍
---------
#介绍
## valgrind概述
Valgrind是一个构建动态分析工具的测试框架。它由许多工具组成，能够通过调试、分析或相似功能改进程序。Valgrind的体系结构是模块化的，所以新工具可以很容易被创建，而不会对已存在工具有任何干扰。
许多有用的工具已经作为标准工具提供。
1、Memcheck是内存检测工具，帮助你改进程序，特别是对使用C C++编写的程序更准确。
2、Cachegrind是缓存和分支预测的工具，帮助你改进程序实质运行更快
3、Callgrind是一个图形化的缓存探测工具，与Cachegrind功能有一些重叠，但是Callgrind能够收集一些信息是Cachegrind没有的
4、Helgrind是线程错误检测工具，帮助改进多线程程序更准确。
5、DRD也是一个线程错误检测工具，DRD与Helgrind相似，但是使用不同的分析技术，所以可能会发现不同的问题。
6、Massif是堆探测工具，帮助改进程序使用更少内存。
7、DHAT是另一个堆检测工具，帮助了解代码段的生命周期，利用率，效率规划。
8、SGcheck是一个实验性的工具，能够检测栈和全局数组溢出错误。它的功能是对Memcheck的补充，SGcheck能够找到Memcheck无法找到的问题，反之亦然。
9. BBV is an experimental SimPoint basic block vector generator. It is useful to people doing computer architecture research and development.
There are also a couple of minor tools that aren’t useful to most users: Lackey is an example tool that illustrates some instrumentation basics; and Nulgrind is the minimal Valgrind tool that does no analysis or instrumentation, and is only useful for testing purposes.
Valgrind is closely tied to details of the CPU and operating system, and to a lesser extent, the compiler and basic C libraries. Nonetheless, it supports a number of widely-used platforms, listed in full at http://www.valgrind.org/.
Valgrind is built via the standard Unix ./configure, make, make install process; full details are given in the README ﬁle in the distribution.Valgrind is licensed under the The GNU General Public License, version 2. The valgrind/*.h headers that you may wish to include in your code (eg. valgrind.h, memcheck.h, helgrind.h, etc.) are distributed under a BSD-style license, so you may include them in your code without worrying about license conﬂicts. Some of the PThreads test cases, pth_*.c, are taken from "Pthreads Programming" by Bradford Nichols, Dick Buttlar & Jacqueline Proulx Farrell, ISBN 1-56592-115-1, published by O’Reilly & Associates, Inc.
If you contribute code to Valgrind, please ensure your contributions are licensed as "GPLv2, or (at your option) any later version." This is so as to allow the possibility of easily upgrading the license to GPLv3 in future. If you want to modify code in the VEX subdirectory, please also see the ﬁle VEX/HACKING.README in the distribution.


##上手文档

###简介

Valgrind系列工具是提供一系列可以使你的程序运行更快更稳定调试、监测工具组。这里面最常用的便是memcheck工具，它能够检测出C和C++程序中的多种致命的内存问题。

下面的使用说明主要是教你快速上手memcheck工具，如何用它来检测你的程序中的内存问题。若想了解memcheck或者其他工具完整的说明文档，请参照用户手册（User Manual）。

###你的程序应当作的准备
1、编译程序的时候需要添加-g选项，这样memcheck的报错信息便可以通过调试信息包括具体出错代码的行数。
2、如果你在调试程序时不介意程序的运行速度，那么建议你选择-O0的编译优化选项。如果选择-O1编译优化级别，虽然，memcheck大多数情况下依然是正常工作，报错信息也基本准确，但是错误代码行数有可时会不准确。
3、不推荐使用-O2以上的编译优化级别，memcheck会误报未初始化变量错误。

使用memcheck检测你的程序
若你的程序运行命令如下
myprog arg1 arg21
那么使用如下的格式便可以使用memcheck检查你的程序中的内存问题

```
valgrind --leak-check=yes myprog arg1 arg21
```
memcheck是valgrind的默认工具，”–leak-check”选项用于检测内存泄漏。

使用valgrind之后，你的程序运行相较于普通时候会慢20～30倍，并且会占用更多的内存。作为补偿，memcheck工具会详细地打印出它所监测到的内存问题和内存泄漏。

###memcheck输出含义

下面的C语言示例程序存在一个内存错误和内存泄漏。


```
#include <stdlib.h>

void f(void)
{
    int* x = malloc(10 * sizeof(int));
    x[10] = 0;        // problem 1: heap block overrun

}                    // problem 2: memory leak -- x not freed

int main(void)
{
    f();
    return 0;
}

1234567891011121314
```
大多数的错误信息会像下面这样（下面的错误信息示例是上面示例代码的错误1）


```
==19182== Invalid write of size 4
==19182==    at 0x804838F: f (example.c:6)
==19182==    by 0x80483AB: main (example.c:11)
==19182==  Address 0x1BA45050 is 0 bytes after a block of size 40 alloc'd
==19182==    at 0x1B8FF5CD: malloc (vg_replace_malloc.c:130)
==19182==    by 0x8048385: f (example.c:5)
==19182==    by 0x80483AB: main (example.c:11)1234567
```

注意：

1、每条错误消息都有详细的错误描述信息，记得仔细阅读

2、19182是进程ID

3、第一行（Invalid write…）是说明这是哪种错误。这里的意思是说这里有违法写操作，会导致程序出问题

4、第一行下面的则是追踪栈（第二行到第三行），它会告诉你错误发生在哪里。有的时候，程序追踪栈是会特别大，或者是难以理解，特别是你使用c++ STL库的时候。建议你自下而上地阅读函数追踪栈。如果由于函数追踪栈特别大，导致打印出来的函数追踪栈并不完全，则建议你使用’–num-callers’选项提升打印栈大小。

5、代码地址（例如：0x804838F）通常来说并不重要，但是有时候通过查看代码地址查出某些怪异的bug，例如代码地址若为0x00000000则显然是有问题的。

6、有些错误消息会有额外的信息（第4～第7行）来描述错误相关的内存地址信息。这条信息是说写的内存位置超过了malloc申请的地址空间，该错误是在程序example.c的第五行。

修复错误最好按照memcheck报告的错误顺序来进行修复，根据经验往往很多后面的错误是由前面的错误导致的。如果不按照这么做的话，通常会带来许多不必要的麻烦。

内存泄漏的消息示例如下


```
  ==19182== 40 bytes in 1 blocks are definitely lost in loss record 1 of 1
  ==19182==    at 0x1B8FF5CD: malloc (vg_replace_malloc.c:130)
  ==19182==    by 0x8048385: f (a.c:5)
  ==19182==    by 0x80483AB: main (a.c:11)1234
```

追踪栈将会告诉你泄漏的内存是在哪里申请的。不幸的是，memcheck并不能告诉你因为什么操作导致内存泄漏。（请无视’vg_replace_malloc.c’，这是一个代码实现细节）

###内存泄漏分很多种，两种最常见的内存泄漏是：

1、’definitely lost’直接泄漏：你的程序确实发生了内存泄漏，修复它！

2、’probably lost’可能泄漏：你的程序有可能要发生内存泄漏，除非你做一些搞笑的事情（比如：把指向一块内存地址头的指针，而且指向这块内存地址头的指针只有一个，然后你把该指针指向该内存的内部的位置，这样便会发生内存泄漏，因为通常来说你把指针指向别处，那么程序很难指导申请的该处内存的头部是哪里，除非人为规定，但是依然很容易出现各种错误）

memcheck也会报告使用未初始化变量错误，通常报错名称为’Conditional jump or move depend on uninitialised value(s)’。通常很难判断这些错误的根因是什么，这时候你可以使用’–track-origins=yes’选项获得额外的信息来判断原因。虽然该选项会使得memcheck运行得更慢，但是实际上所获得的额外信息大大节省了你在寻找和考虑这个问题的原因是什么的时间。

如果你不懂memcheck所报出的错误，请查询memcheck用户手册，那里很详细。



###注意事项

memcheck也并不完美；它偶尔也会出现误报，不过有抑制这种误报的方法（具体抑制方法请参看《用户手册》）。然而，memcheck在99%的情况下都是正确的，所以你最好不要因为你认为有可能memcheck报得错误是错的就不管该错误信息。毕竟，你并不会忽略编译器报的警告错误，对吧！memcheck的抑制机制也对它报出系统库函数中的错误有用，因为你不能修改系统库函数，所以只能避开它。memcheck的默认设置已经过滤掉很多误报，但是并不代表你就不会遇到问题。

memcheck并不能检测出你程序中的每一个内存错误。例如，它不能检测出来越界读操作、不能检测出静态申请的内存或静态栈上进行写操作。但是它肯定能够检测出很多能让你程序崩溃的错误（例如，segmentation fault段错误）

尽量让你的程序’clean’这样memcheck便能更准确的报出错误。一旦你能够做到这样，那么你将会更容易检测出新加入的代码中是哪里出了问题导致memcheck报出新的错误。从过去几年使用memcheck工具的经验中，大型代码通过memcheck检查、清理，也是能够做到memcheck零报错。比如，KDE的很大一部分，OpenOffice.org和Firefox使用memcheck检查过后，是没有或者基本没有memcheck报出的错误的。


##启动方式: valgrind [valgrind-options] your-prog [your-prog-options]

--gen-suppressions=yes 当Valgrind检测到系统库函数错误时，屏蔽此类错误
-v ，--verbose获得详细信息
必须调用Valgrind执行真正的可执行文件，而非shell或Perl的脚本文件
Valgrind输出三种方式：
（1）默认输出到2（stderr），可通过--log-fd=9指定
（2）指定输出到文件--log-file=filename
（3）指定输出到网络端口--log-socket=192.168.0.1:12345
-q,--quiet 安静执行，只打印错误信息
--trace-children=<yes|no> [default: no] 跟踪exec调用执行的子进程
--xml=<yes|no> [default: no] 用xml格式输出日志
--error-limit=<yes|no> [default: yes]  默认yes，在总量达到10,000,000，或者1,000个不同的错误，Valgrind停止报告错误。设置为no，则不限制。

###Memcheck内存检测工具

Memcheck能够检测的几类问题：
获得非法内存
使用未定义的值
错误的释放堆内存
使用memcpy或相关函数，源和目的指针重叠
内存泄露

###错误信息解释
######非法的读写错误
```
Invalid read of size 4
at 0x40F6BBCC: (within /usr/lib/libpng.so.2.1.0.9)
by 0x40F6B804: (within /usr/lib/libpng.so.2.1.0.9)
by 0x40B07FF4: read_png_image(QImageIO *) (kernel/qpngio.cpp:326)
by 0x40AC751B: QImageIO::read() (kernel/qimage.cpp:3621)
Address 0xBFFFF0E0 is not stack’d, malloc’d or free’d
```


###使用未初始化的值
```
Conditional jump or move depends on uninitialised value(s)
at 0x402DFA94: _IO_vfprintf (_itoa.h:49)
by 0x402E8476: _IO_printf (printf.c:36)
by 0x8048472: main (tests/manuel1.c:8)
```
--track-origins=yes 查看详细未初始化信息

###在系统调用中，使用未初始化的或不可寻址值

```
#include <stdlib.h>
#include <unistd.h>
int main( void )
{
char* arr = malloc(10);
int* arr2 = malloc(sizeof(int));
write( 1 /* stdout */, arr, 10 );
exit(arr2[0]);
}
```

###错误信息

```
Syscall param write(buf) points to uninitialised byte(s)
at 0x25A48723: __write_nocancel (in /lib/tls/libc-2.3.3.so)
by 0x259AFAD3: __libc_start_main (in /lib/tls/libc-2.3.3.so)
by 0x8048348: (within /auto/homes/njn25/grind/head4/a.out)
Address 0x25AB8028 is 0 bytes inside a block of size 10 alloc’d
at 0x259852B0: malloc (vg_replace_malloc.c:130)
by 0x80483F1: main (a.c:5)
Syscall param exit(error_code) contains uninitialised byte(s)
at 0x25A21B44: __GI__exit (in /lib/tls/libc-2.3.3.so)
by 0x8048426: main (a.c:8)
```
###非法释放

```
Invalid free()
at 0x4004FFDF: free (vg_clientmalloc.c:577)
by 0x80484C7: main (tests/doublefree.c:10)
Address 0x3807F7B4 is 0 bytes inside a block of size 177 free’d
at 0x4004FFDF: free (vg_clientmalloc.c:577)
by 0x80484C7: main (tests/doublefree.c:10)
```
###堆空间不恰当释放
使用new[]分配空间，free释放

```
Mismatched free() / delete / delete []
at 0x40043249: free (vg_clientfuncs.c:171)
by 0x4102BB4E: QGArray::~QGArray(void) (tools/qgarray.cpp:149)
by 0x4C261C41: PptDoc::~PptDoc(void) (include/qmemarray.h:60)
by 0x4C261F0E: PptXml::~PptXml(void) (pptxml.cc:44)
Address 0x4BB292A8 is 0 bytes inside a block of size 64 alloc’d
at 0x4004318C: operator new[](unsigned int) (vg_clientfuncs.c:152)
by 0x4C21BC15: KLaola::readSBStream(int) const (klaola.cc:314)
by 0x4C21C155: KLaola::stream(KLaola::OLENode const *) (klaola.cc:416)
by 0x4C21788F: OLEFilter::convert(QCString const &) (olefilter.cc:272)
```

###源地址和目的地址有重叠
使用memcpy,strcpy, strncpy, strcat, strncat这些函数可能出现

```
==27492== Source and destination overlap in memcpy(0xbffff294, 0xbffff280, 21)

==27492== at 0x40026CDC: memcpy (mc_replace_strmem.c:71)

==27492== by 0x804865A: main (overlap.c:40)
```

4.2.7内存泄露检测
使用--leak-check 
9种可能出现的情况

```
Pointer chain          AAA Category         BBB Category

-------------          ------------         ------------

(1) RRR ------------> BBB                    DR

(2) RRR ---> AAA ---> BBB DR                 IR

(3) RRR               BBB              DL

(4) RRR AAA ---> BBB DL IL

(5) RRR ------?-----> BBB                  (y)DR, (n)DL

(6) RRR ---> AAA -?-> BBB DR               (y)IR, (n)DL

(7) RRR -?-> AAA ---> BBB (y)DR, (n)DL     (y)IR, (n)IL

(8) RRR -?-> AAA -?-> BBB (y)DR, (n)DL     (y,y)IR, (n,y)IL, (_,n)DL

(9) RRR AAA -?-> BBB DL                    (y)IL, (n)DL

Pointer chain legend:

- RRR: a root set node or DR block

- AAA, BBB: heap blocks

- --->: a start-pointer

- -?->: an interior-pointer

Category legend:

- DR: Directly reachable

- IR: Indirectly reachable

- DL: Directly lost

- IL: Indirectly lost

- (y)XY: it’s XY if the interior-pointer is a real pointer

- (n)XY: it’s XY if the interior-pointer is not a real pointer

- (_)XY: it’s XY in either case
"Still reachable". This covers cases 1 and 2
"Deﬁnitely lost". This covers case 3  

"Indirectly lost". This covers cases 4 and 9
"Possibly lost". This covers cases 5--8 不需要考虑
```
###命令行选项
--leak-check=<no|summary|yes|full> [default: summary]       设置为full或yes将给出详细内存泄露检测
--show-possibly-lost=<yes|no> [default: yes]    设置为disabled将不显示 "possibly lost"结果
--track-origins=<yes|no> [default: no]       跟踪未初始化值产生根源


```
常见的valgrind命令
valgrind --tool=memcheck --leak-check=yes --show-reachable=yes ./bin/PanGenGUI > log.txt 2>&1

valgrind --tool=memcheck --leak-check=full --log-file=mem.log ./bin/PanGenGUI
```
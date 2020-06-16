###与符号表分离程序或动态库, 如何用GDB调试
-----------------
#Debugging Information in Separate Files
GDB支持将程序调试信息放在独立的文件里，与可执行程序分离，其可以自动查找和自动加载调试信息。
由于调试信息比较大，甚至比可执行程序还要大，通常将可执行程序的调试信息以单独文件的形式发布，需要调试时可以再安装这些文件。

##GDB支持两种设置单独调试信息文件的方式：
第一种，可执行程序含调试链接，该链接指定单独的调试信息文件名。单独调试文件名通常是executable.debug，executable是相应的可执行程序名，
不带路径（比如：ls.debug是/usr/bin/ls的调试信息文件）。
此外，调试链接为调试文件设置了CRC32的校验和，GDB用此校验和来确保可执行文件和调试文件是同一个版本的。
第二种，可执行文件含版本ID号和唯一的bit串，而相应的调试信息文件也包含该bit串。该方式只在某些系统上支持，特别是那些在二进制文件里使用ELF格式和GNU Binutils的系统。更多关于此功能的细节，可以参见–build-id命令行选项的介绍，在GNU连接器“命令行选项”节中。
虽然版本ID号没有直接指出调试信息文件名，但是可以从版本ID号里计算出来，参见下面。

##GDB也提供两种不同的方式查找调试信息文件：
第一种，对于调试链接方式，GDB在可执行文件的目录里查找对应名字的文件，接着在此目录下的子目录.debug下查找，最后在全局调试目录下的一个子目录里查找，此子目录的名字和可执行文件的绝对文件名的先导目录名相同。
第二种，对于版本ID方式，GDB在全局调试目录下的.build-id子目录下查找名为nn/nnnnnnnn.debug的文件，这里nn是版本ID字符串的头两个16进制字符，nnnnnnnn是余下的字符。真正的版本ID字符串是32个，或更多的16进制字符，不是10个。
举个例子，假设要调试/usr/bin/ls,此程序有个调试链接指定了调试文件ls.debug，且其版本ID是是16进制的abcdef1234。如果全局调试目录是/usr/lib/debug，那么GDB按顺序查找下列调试信息文件：
```
/usr/lib/debug/.build-id/ab/cdef1234.debug
/usr/bin/ls.debug
/usr/bin/.debug/ls.debug
/usr/lib/debug/usr/bin/ls.debug
```
同时可以设置全局调试信息目录的名称，并查看当前GDB所使用的名称。
```
set debug-file-directory directory // 将directory设置为GDB搜索单独调试信息文件的目录。
show debug-file-directory          // 显示搜索单独调试信息文件的目录。
```
Gdb 调试与符号表分离的二进制程序
通常当没有在程序spec中明确指定不进行strip时，缺省打rpm包都会把二进制程序或动态库的符号表等debuginfo信息与执行程序分离，生成一个debuginfo的 rpm包。比如localagent打rpm包时，会生成如下3个rpm 包：
```
local_agent-0.7.1-rc_1.x86_64.rpm
local_agent-debuginfo-0.7.1-rc_1.x86_64.rpm
local_agent-devel-0.7.1-rc_1.x86_64.rpm
将local_agent-debuginfo-0.7.1-rc_1.x86_64.rpm解压，可以看到相应的debug info文件。

[tmp]$ rpm2cpio local_agent-debuginfo-0.7.1-rc_1.x86_64.rpm |cpio –idv
./usr/local/bin/.debug/local_agent_client.debug
./usr/local/bin/.debug/local_agent_server.debug
./usr/local/lib64/.debug/liblocal_agent.so.debug
./usr/src/debug/local_agent-0.7.1
./usr/src/debug/local_agent-0.7.1/build
./usr/src/debug/local_agent-0.7.1/build/release64
./usr/src/debug/local_agent-0.7.1/build/release64/local_agent
./usr/src/debug/local_agent-0.7.1/build/release64/local_agent/ClientParamParser.cpp
.......
./usr/src/debug/local_agent-0.7.1/build/release64/local_agent/local_agent.pb.cc
.......
```
对于符号表与二进程序分离的程序，该如何调试呢？  
```
[local]$ gdb bin/local_agent_server
GNU gdb Fedora (6.8-37.el5)
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu"...
(no debugging symbols found)
(gdb) b main
Breakpoint 1 at 0x401a80
(gdb) r
Starting program: /home/admin/tmp/usr/local/bin/local_agent_server
[Thread debugging using libthread_db enabled]
[New Thread 0x2ad3e6935860 (LWP 27869)]
Breakpoint 1, 0x0000000000401a80 in main ()
(gdb) bt
#0 0x0000000000401a80 in main ()
(gdb)
.......
```
可以看出，由于gdb没有符号表，所以显示的都是二进制地址。

方法一：gdb 启动时通过 –s 指定
当gdb启动时，通过–s指定符号表文件来解决。如下：
```
[ local]$ gdb -e bin/local_agent_server -s debug/local_agent_server.debug
GNU gdb Fedora (6.8-37.el5)
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu"...
(gdb) b main
Breakpoint 1 at 0x401a80: file build/release64/local_agent/LocalAgentMain.cpp, line 34.
(gdb) bt
No stack.
(gdb) r
Starting program: /home/admin/tmp/usr/local/bin/local_agent_server
[Thread debugging using libthread_db enabled]
[New Thread 0x2af3399a7860 (LWP 27862)]
Breakpoint 1, main (argc=1, argv=0x7fff5728f0b8) at build/release64/local_agent/LocalAgentMain.cpp:34
34 build/release64/local_agent/LocalAgentMain.cpp: No such file or directory.
in build/release64/local_agent/LocalAgentMain.cpp
(gdb) quit
```
如果是符号表与二进程序分离的程序进行所产生的core，可以通过以下方式调试：
gdb -c core.1234 -e bin/local_agent_server -s debug/local_agent_server.debug
其中：
-c 指定core文件
-e 指定binary，用线上的binary即可
-s 指定符号表，也就是我们新生成的符号表

方法二：将.debug 目录放到可执行程序所在目录
除了通过–s指定debuginfo文件外，还可以将.debug目录复制到可执行程序所在目录或者将local_agent_server.debug复制到可执行程序所在的目录。
```
[admin@s002182.cm6 local]$ pwd /home/admin/tmp/usr/local
[admin@s002182.cm6 local]$ cp -r debug/ bin/.debug
[admin@s002182.cm6 local]$ ls -lha bin/
total 44K
drwxr-xr-x 3 admin admin 4.0K Mar 21 18:41 .
drwxr-xr-x 6 admin admin 4.0K Mar 21 17:47 ..
drwxr-xr-x 3 admin admin 4.0K Mar 21 18:49 .debug
-rwxr-xr-x 1 admin admin 14K Mar 21 17:44 local_agent_client
-rwxr-xr-x 1 admin admin 14K Mar 21 17:44 local_agent_server
```
此时，可以看到符号表了。

```
[local]$ gdb bin/local_agent_server
GNU gdb Fedora (6.8-37.el5)
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu"...
(gdb) b main
Breakpoint 1 at 0x401a80: file build/release64/local_agent/LocalAgentMain.cpp, line 34.
(gdb) q
```
set debug-file-directory 方式没有生效
尝试用set debug-file-directory方式来设置debug路径，目前看还没有生效，需要再确定一下。
```
[admin@s002182.cm6 local]$ gdb bin/local_agent_server

GNU gdb Fedora (6.8-37.el5)
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type "show copying
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu"...
(no debugging symbols found)
(gdb) b main
Breakpoint 1 at 0x401a80
(gdb) show debug-file-directory
The directory where separate debug symbols are searched for is "/usr/lib/debug".
(gdb) set debug-file-directory /usr/lib/debug:/home/admin/tmp/usr/local/debug
(gdb) show debug-file-directory
The directory where separate debug symbols are searched for is "/usr/lib/debug:/home/admin/tmp/usr/local/debug
```
gdb 调试与符号表分离的动态库
当二进制程序所依赖的动态库是符号表等调试信息与动态库二进制分离的时，该如何让 gdb 加载动态库的符号表呢？

可执行程序运行时，依赖的动态库是通过 rpm 包安装的情况
比较简单，只要将debuginfo的rpm包将通过rpm安装即可，因为.debug目录缺省会安装在动态库所在的目录下。
下面的库的rpm包解压后，可以清楚的看到这一点：
```
[admin@s002182.cm6 tmp]$ rpm2cpio anet-1.3.2-rc_2.x86_64.rpm |cpio -idv
./usr/local/lib64/libanet.so
./usr/local/lib64/libanet.so.3
./usr/local/lib64/libanet.so.3.0.0
563 blocks

[admin@s002182.cm6 tmp]$ rpm2cpio anet-debuginfo-1.3.2-rc_2.x86_64.rpm |cpio -idv
./usr/local/lib64/.debug/libanet.so.3.0.0.debug
./usr/src/debug/anet-1.3.2
./usr/src/debug/anet-1.3.2/anet
./usr/src/debug/anet-1.3.2/anet/addrspec.h
./usr/src/debug/anet-1.3.2/anet/adminclient.h
```
可执行程序运行时，依赖的动态库是通过rpm2cipo解压到某个目录的情况
如果可执行程序运行时，依赖的动态库是用户自已通过rpm2cpio解压后复制到某个用户自定义的目录的，那么，拿到该动态库的debuginfo rpm后，
相应的用 rpm2cpio 解压，并把.debug目录复制到相应的动态库所在的目录。
比如以下示例，sap_server依赖libarpc.so.1，但其debuginfo信息分离在单独的debuginfo rpm包中的，所以导致gdb core时,看不到libarpc.so.1的符号表。
```
gdb bin/sap_server_d core.17131
(gdb) bt
#0 0x0000003959230265 in raise (sig=<value optimized out>) at ../nptl/sysdeps/unix/sysv/linux/raise.c:64
#1 0x0000003959231d10 in abort () at abort.c:88
#2 0x000000395d6bec44 in __gnu_cxx::__verbose_terminate_handler () at ../../../../libstdc++-v3/libsupc++/vterminate.cc:97
#3 0x000000395d6bcdb6 in __cxxabiv1::__terminate (handler=<value optimized out>) at ../../../../libstdc++-v3/libsupc++/eh_terminate.cc:4
#4 0x000000395d6bcde3 in std::terminate () at ../../../../libstdc++-v3/libsupc++/eh_terminate.cc:53
#5 0x000000395d6bd2ef in __cxa_pure_virtual () at ../../../../libstdc++-v3/libsupc++/pure.cc:5
#6 0x00002b8729bfc933 in arpc::ClientPacketHandler::handlePacket () from /home/admin/sap/lib/libarpc.so.1
#7 0x00002b872a0ddcd8 in anet::Connection::handlePacket () from /home/admin/sap/lib/libanet.so.3
#8 0x00002b872a0e75fb in anet::TCPConnection::readData () from /home/admin/sap/lib/libanet.so.3
#9 0x00002b872a0e624b in anet::TCPComponent::handleReadEvent () from /home/admin/sap/lib/libanet.so.3
#10 0x00002b872a0e8c0b in anet::Transport::eventIteration () from /home/admin/sap/lib/libanet.so.3
#11 0x00002b872a0e8cf2 in anet::Transport::eventLoop () from /home/admin/sap/lib/libanet.so.3
#12 0x00002b872a0e8d47 in anet::Transport::run () from /home/admin/sap/lib/libanet.so.3
#13 0x00002b872a0ea02d in anet::Thread::hook () from /home/admin/sap/lib/libanet.so.3
#14 0x0000003959e064a7 in start_thread (arg=<value optimized out>) at pthread_create.c:297
#15 0x00000039592d3c2d in clone () from /lib64/libc.so.6
(gdb) thread 2
```
要让gdb调试core时，能显示动态库的符号表，可以用如下方法：
a). 先解压debuginfo  rpm包
rpm2cpio arpc-debuginfo-0.14.1-rc_1.x86_64.rpm |cipo -idv
b). 将 ./usr/local/lib64/.debug复制到 libarpc.so.1所在目录：
/home/admin/sap/lib/  cp -r  ./usr/local/lib64/.debug  /home/admin/sap/lib/
```
[admin@s007238.cm6 sap]$ gdb bin/sap_server_d core.17131
GNU gdb Fedora (6.8-37.el5)
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law. Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu"...
Reading symbols from /home/admin/sap/lib/libhb_node.so...done.
Loaded symbols for /home/admin/sap/lib/libhb_node.so
Reading symbols from /home/admin/sap/lib/libcm_basic.so...done.
Loaded symbols for /home/admin/sap/lib/libcm_basic.so
Reading symbols from /home/admin/sap/lib/libarpc.so.1...Reading symbols from /home/admin/sap/lib/.debug/libarpc.so.1.0.0.debug...done.
... ...
#0 0x0000003959230265 in raise (sig=<value optimized out>) at ../nptl/sysdeps/unix/sysv/linux/raise.c:64
64 return INLINE_SYSCALL (tgkill, 3, pid, selftid, sig);
(gdb) bt
#0 0x0000003959230265 in raise (sig=<value optimized out>) at ../nptl/sysdeps/unix/sysv/linux/raise.c:64
#1 0x0000003959231d10 in abort () at abort.c:88
#2 0x000000395d6bec44 in __gnu_cxx::__verbose_terminate_handler () at ../../../../libstdc++-v3/libsupc++/vterminate.cc:97
#3 0x000000395d6bcdb6 in __cxxabiv1::__terminate (handler=<value optimized out>) at ../../../../libstdc++-v3/libsupc++/eh_terminate.cc:4#4 0x000000395d6bcde3 in std::terminate () at ../../../../libstdc++-v3/libsupc++/eh_terminate.cc:53
#5 0x000000395d6bd2ef in __cxa_pure_virtual () at ../../../../libstdc++-v3/libsupc++/pure.cc:55
#6 0x00002b8729bfc933 in arpc::ClientPacketHandler::handlePacket (this=0x2aac6df6ee38, packet=0x2aac6df6fc30, args=<value optimized out>) at build/release64/arpc/ClientPacketHandler.cpp:35
#7 0x00002b872a0ddcd8 in anet::Connection::handlePacket () from /home/admin/sap/lib/libanet.so.3
#8 0x00002b872a0e75fb in anet::TCPConnection::readData () from /home/admin/sap/lib/libanet.so.3
#9 0x00002b872a0e624b in anet::TCPComponent::handleReadEvent () from /home/admin/sap/lib/libanet.so.3
#10 0x00002b872a0e8c0b in anet::Transport::eventIteration () from /home/admin/sap/lib/libanet.so.3
#11 0x00002b872a0e8cf2 in anet::Transport::eventLoop () from /home/admin/sap/lib/libanet.so.3
#12 0x00002b872a0e8d47 in anet::Transport::run () from /home/admin/sap/lib/libanet.so.3
#13 0x00002b872a0ea02d in anet::Thread::hook () from /home/admin/sap/lib/libanet.so.3
#14 0x0000003959e064a7 in start_thread (arg=<value optimized out>) at pthread_create.c:297
#15 0x00000039592d3c2d in clone () from /lib64/libc.so.6
Current language: auto; currently c
(gdb)
```
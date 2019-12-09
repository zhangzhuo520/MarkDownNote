#CMake的基本入门知识
---
##自定义变量
主要有隐式定义和显式定义两种。 
隐式定义的一个例子是PROJECT指令，它会隐式的定义< projectname >_BINARY_DIR和< projectname >_SOURCE_DIR两个变量；显式定义使用SET指令构建自定义变量，比如:SET(HELLO_SRCmain.c)就可以通过${HELLO_SRC}来引用这个自定义变量了。

##变量引用方式
使用${}进行变量的引用；在IF等语句中,是直接使用变量名而不通过${}取值。

##常用变量
###CMAKE_BINARY_DIR 
###PROJECT_BINARY_DIR 
###<projectname >_BINARY_DIR 
这三个变量指代的内容是一致的，如果是in-source编译，指得就是工程顶层目录；如果是out-of-source编译，指的是工程编译发生的目录。PROJECT_BINARY_DIR跟其它指令稍有区别，目前可以认为它们是一致的。

###CMAKE_SOURCE_DIR 
###PROJECT_SOURCE_DIR 
###<projectname >_SOURCE_DIR 
这三个变量指代的内容是一致的，不论采用何种编译方式，都是工程顶层目录。也就是在in-source编译时,他跟CMAKE_BINARY_DIR等变量一致。PROJECT_SOURCE_DIR跟其它指令稍有区别,目前可以认为它们是一致的。 
（out-of-source build与in-source build相对，指是否在CMakeLists.txt所在目录进行编译。）

###CMAKE_CURRENT_SOURCE_DIR 
当前处理的CMakeLists.txt所在的路径，比如上面我们提到的src子目录。

###CMAKE_CURRRENT_BINARY_DIR 
如果是in-source编译，它跟CMAKE_CURRENT_SOURCE_DIR一致；如果是out-of-source编译，指的是target编译目录。使用ADD_SUBDIRECTORY(src bin)可以更改这个变量的值。使用SET(EXECUTABLE_OUTPUT_PATH <新路径>)并不会对这个变量造成影响,它仅仅修改了最终目标文件存放的路径。

###CMAKE_CURRENT_LIST_FILE 
输出调用这个变量的CMakeLists.txt的完整路径

###CMAKE_CURRENT_LIST_LINE 
输出这个变量所在的行

###CMAKE_MODULE_PATH 
这个变量用来定义自己的cmake模块所在的路径。如果工程比较复杂，有可能会自己编写一些cmake模块，这些cmake模块是随工程发布的，为了让cmake在处理CMakeLists.txt时找到这些模块，你需要通过SET指令将cmake模块路径设置一下。比如SET(CMAKE_MODULE_PATH,${PROJECT_SOURCE_DIR}/cmake) 
这时候就可以通过INCLUDE指令来调用自己的模块了。

###EXECUTABLE_OUTPUT_PATH 
新定义最终结果的存放目录

###LIBRARY_OUTPUT_PATH 
新定义最终结果的存放目录

###PROJECT_NAME 
返回通过PROJECT指令定义的项目名称。

###cmake调用环境变量的方式
使用$ENV{NAME}指令就可以调用系统的环境变量了。比如MESSAGE(STATUS "HOME dir: $ENV{HOME}")设置环境变量的方式是SET(ENV{变量名} 值)。

###CMAKE_INCLUDE_CURRENT_DIR 
自动添加CMAKE_CURRENT_BINARY_DIR和CMAKE_CURRENT_SOURCE_DIR到当前处理的CMakeLists.txt，相当于在每个CMakeLists.txt加入：INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_SOURCE_DIR})

###CMAKE_INCLUDE_DIRECTORIES_PROJECT_BEFORE 
将工程提供的头文件目录始终置于系统头文件目录的前面,当定义的头文件确实跟系统发生冲突时可以提供一些帮助。

###CMAKE_INCLUDE_PATH和CMAKE_LIBRARY_PATH

##系统信息
CMAKE_MAJOR_VERSION，CMAKE主版本号，比如2.4.6中的2
CMAKE_MINOR_VERSION，CMAKE次版本号，比如2.4.6中的4
CMAKE_PATCH_VERSION，CMAKE补丁等级，比如2.4.6中的6
CMAKE_SYSTEM，系统名称，比如Linux-2.6.22
CMAKE_SYSTEM_NAME，不包含版本的系统名，比如Linux
CMAKE_SYSTEM_VERSION，系统版本，比如2.6.22
CMAKE_SYSTEM_PROCESSOR，处理器名称，比如i686
UNIX，在所有的类Unix平台为TRUE，包括OSX和cygwin
WIN32，在所有的Win32平台为TRUE，包括cygwin
主要的开关选项
CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS 
用来控制IF ELSE语句的书写方式。

###BUILD_SHARED_LIBS 
这个开关用来控制默认的库编译方式。如果不进行设置，使用ADD_LIBRARY并没有指定库类型的情况下，默认编译生成的库都是静态库；如果SET(BUILD_SHARED_LIBSON)后,默认生成的为动态库。

###CMAKE_C_FLAGS 
设置C编译选项，也可以通过指令ADD_DEFINITIONS()添加。

###MAKE_CXX_FLAGS 
设置C++编译选项，也可以通过指令ADD_DEFINITIONS()添加。

##cMake常用指令
这里引入更多的cmake指令,为了编写的方便,将按照cmakeman page 的顺序介绍各种指令，不再推荐使用的指令将不再介绍。

###基本指令
###PROJECT(HELLO) 
指定项目名称，生成的VC项目的名称，使用${HELLO_SOURCE_DIR}表示项目根目录。

###INCLUDE_DIRECTORIES 
指定头文件的搜索路径，相当于指定gcc的-I参数 
INCLUDE_DIRECTORIES(${HELLO_SOURCE_DIR}/Hello) #增加Hello为include目录

###TARGET_LINK_LIBRARIES 
添加链接库，相同于指定-l参数 
TARGET_LINK_LIBRARIES(demoHello) #将可执行文件与Hello连接成最终文件demo

###LINK_DIRECTORIES 
动态链接库或静态链接库的搜索路径，相当于gcc的-L参数 
LINK_DIRECTORIES(${HELLO_BINARY_DIR}/Hello)#增加Hello为link目录

###ADD_DEFINITIONS 
向C/C++编译器添加-D定义，比如： 
ADD_DEFINITIONS(-DENABLE_DEBUG-DABC) 
参数之间用空格分割。如果代码中定义了:
```
#ifdef ENABLE_DEBUG

#endif
```
这个代码块就会生效。如果要添加其他的编译器开关,可以通过CMAKE_C_FLAGS变量和CMAKE_CXX_FLAGS变量设置。

###ADD_DEPENDENCIES* 
定义target依赖的其它target，确保在编译本target之前，其它的target已经被构建。ADD_DEPENDENCIES(target-name depend-target1 depend-target2 ...)

###ADD_EXECUTABLE 
ADD_EXECUTABLE(helloDemo demo.cxx demo_b.cxx) 
指定编译，好像也可以添加.o文件，将cxx编译成可执行文件

###ADD_LIBRARY 
ADD_LIBRARY(Hellohello.cxx) #将hello.cxx编译成静态库如libHello.a

###ADD_SUBDIRECTORY 
ADD_SUBDIRECTORY(Hello) #包含子目录

###ADD_TEST 
ENABLE_TESTING 
ENABLE_TESTING指令用来控制Makefile是否构建test目标，涉及工程所有目录。语法很简单，没有任何参数，ENABLE_TESTING()一般放在工程的主CMakeLists.txt中。 
ADD_TEST指令的语法是:ADD_TEST(testnameExename arg1 arg2 …) 
testname是自定义的test名称，Exename可以是构建的目标文件也可以是外部脚本等等，后面连接传递给可执行文件的参数。

如果没有在同一个CMakeLists.txt中打开ENABLE_TESTING()指令，任何ADD_TEST都是无效的。比如前面的Helloworld例子,可以在工程主CMakeLists.txt中添加
```
ADD_TEST(mytest ${PROJECT_BINARY_DIR}/bin/main)
ENABLE_TESTING
```
生成Makefile后，就可以运行make test来执行测试了。

###AUX_SOURCE_DIRECTORY 
基本语法是:AUX_SOURCE_DIRECTORY(dir VARIABLE)，作用是发现一个目录下所有的源代码文件并将列表存储在一个变量中，这个指令临时被用来自动构建源文件列表，因为目前cmake还不能自动发现新添加的源文件。比如：
```
AUX_SOURCE_DIRECTORY(. SRC_LIST)
ADD_EXECUTABLE(main ${SRC_LIST})
```
可以通过后面提到的FOR EACH指令来处理这个LIST。

###CMAKE_MINIMUM_REQUIRED 
语法为CMAKE_MINIMUM_REQUIRED(VERSION versionNumber [FATAL_ERROR])， 
比如:CMAKE_MINIMUM_REQUIRED(VERSION 2.5 FATAL_ERROR) 
如果cmake版本小与2.5，则出现严重错误，整个过程中止。

###EXEC_PROGRAM 
在CMakeLists.txt处理过程中执行命令，并不会在生成的Makefile中执行。具体语法为:
```
EXEC_PROGRAM(Executable [directory in which to run] [ARGS <arguments to executable>] [OUTPUT_VARIABLE <var>] [RETURN_VALUE <var>])
```
用于在指定的目录运行某个程序，通过ARGS添加参数，如果要获取输出和返回值，可通过OUTPUT_VARIABLE和RETURN_VALUE分别定义两个变量。 
这个指令可以帮助在CMakeLists.txt处理过程中支持任何命令，比如根据系统情况去修改代码文件等等。举个简单的例子，我们要在src目录执行ls命令，并把结果和返回值存下来，可以直接在src/CMakeLists.txt中添加：
```
EXEC_PROGRAM(ls ARGS "*.c" OUTPUT_VARIABLE LS_OUTPUT RETURN_VALUE LS_RVALUE)
IF(not LS_RVALUE)
    MESSAGE(STATUS "ls result: " ${LS_OUTPUT})
ENDIF(not LS_RVALUE)
```
在cmake生成Makefile过程中，就会执行ls命令，如果返回0，则说明成功执行，那么就输出ls *.c的结果。关于IF语句，后面的控制指令会提到。

###FILE指令 
文件操作指令，基本语法为:
```
FILE(WRITEfilename "message to write"... )
FILE(APPENDfilename "message to write"... )
FILE(READfilename variable)
FILE(GLOBvariable [RELATIVE path] [globbing expression_r_rs]...)
FILE(GLOB_RECURSEvariable [RELATIVE path] [globbing expression_r_rs]...)
FILE(REMOVE[directory]...)
FILE(REMOVE_RECURSE[directory]...)
FILE(MAKE_DIRECTORY[directory]...)
FILE(RELATIVE_PATHvariable directory file)
FILE(TO_CMAKE_PATHpath result)
FILE(TO_NATIVE_PATHpath result)
```

###INCLUDE指令 
用来载入CMakeLists.txt文件，也用于载入预定义的cmake模块。
```
INCLUDE(file1[OPTIONAL])
INCLUDE(module[OPTIONAL])
```
OPTIONAL参数的作用是文件不存在也不会产生错误，可以指定载入一个文件，如果定义的是一个模块，那么将在CMAKE_MODULE_PATH中搜索这个模块并载入，载入的内容将在处理到INCLUDE语句是直接执行。

###INSTALL指令

###FIND_指令 
FIND_系列指令主要包含一下指令:
```
FIND_FILE(<VAR>name1 path1 path2 …)    VAR变量代表找到的文件全路径,包含文件名
FIND_LIBRARY(<VAR>name1 path1 path2 …)    VAR变量表示找到的库全路径,包含库文件名
FIND_PATH(<VAR>name1 path1 path2 …)   VAR变量代表包含这个文件的路径
FIND_PROGRAM(<VAR>name1 path1 path2 …)   VAR变量代表包含这个程序的全路径
FIND_PACKAGE(<name>[major.minor] [QUIET] [NO_MODULE] [[REQUIRED|COMPONENTS][componets...]])   用来调用预定义在CMAKE_MODULE_PATH下的Find<name>.cmake模块，也可以自己定义Find<name>模块，通过SET(CMAKE_MODULE_PATH dir)将其放入工程的某个目录中供工程使用，后面会详细介绍FIND_PACKAGE的使用方法和Find模块的编写。
```

###FIND_LIBRARY示例:
```
FIND_LIBRARY(libXX11 /usr/lib)
IF(NOT libX)
    MESSAGE(FATAL_ERROR "libX not found")
ENDIF(NOT libX)
```

###控制指令
IF指令，基本语法为:
```
IF(expression_r_r)
    #THEN section.
    COMMAND1(ARGS…)
    COMMAND2(ARGS…)
    …
ELSE(expression_r_r)
    #ELSE section.
    COMMAND1(ARGS…)
    COMMAND2(ARGS…)
    …
ENDIF(expression_r_r)
```

另外一个指令是ELSEIF，总体把握一个原则，凡是出现IF的地方一定要有对应的ENDIF，出现ELSEIF的地方，ENDIF是可选的。表达式的使用方法如下:
```
IF(var)  如果变量不是：空, 0, N, NO, OFF, FALSE, NOTFOUND 或 <var>_NOTFOUND时，表达式为真。
IF(NOT var)， 与上述条件相反。
IF(var1AND var2)， 当两个变量都为真是为真。
IF(var1OR var2)， 当两个变量其中一个为真时为真。
IF(COMMANDcmd)， 当给定的cmd确实是命令并可以调用是为真。
IF(EXISTS dir) or IF(EXISTS file)， 当目录名或者文件名存在时为真。
IF(file1IS_NEWER_THAN file2)， 当file1比file2新，或者file1/file2其中有一个不存在时为真文件名请使用完整路径。
IF(IS_DIRECTORY dirname),  当dirname是目录时为真。
IF(variableMATCHES regex)
IF(string MATCHES regex) 当给定的变量或者字符串能够匹配正则表达式regex时为真。比如:
IF("hello" MATCHES "hello")
    MESSAGE("true")
ENDIF("hello" MATCHES "hello")

IF(variable LESS number)
IF(string LESS number)
IF(variable GREATER number)
IF(string GREATER number)
IF(variable EQUAL number)
IF(string EQUAL number)

数字比较表达式

IF(variable STRLESS string)
IF(string STRLESS string)
IF(variable STRGREATER string)
IF(string STRGREATER string)
IF(variable STREQUAL string)
IF(string STREQUAL string)
```
按照字母序的排列进行比较。

IF(DEFINED variable)，如果变量被定义，为真。 
一个小例子,用来判断平台差异:
```
IF(WIN32)
    MESSAGE(STATUS“This is windows.”) #作一些Windows相关的操作
ELSE(WIN32)
    MESSAGE(STATUS“This is not windows”) #作一些非Windows相关的操作
ENDIF(WIN32)
```
上述代码用来控制在不同的平台进行不同的控制,但是阅读起来却并不是那么舒服, ELSE(WIN32)之类的语句很容易引起歧义。 
这就用到了我们在 常用变量 一节提到的CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS开关。可以SET(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTSON)，这时候就可以写成:

```
IF(WIN32)

ELSE()

ENDIF()
```

如果配合ELSEIF使用,可能的写法是这样:
```
IF(WIN32)
    #dosomething related to WIN32
ELSEIF(UNIX)
    #dosomething related to UNIX
ELSEIF(APPLE)
    #dosomething related to APPLE
ENDIF(WIN32)
```

###WHILE指令
WHILE指令的语法是:
```
WHILE(condition)
    COMMAND1(ARGS…)
    COMMAND2(ARGS…)
    …
ENDWHILE(condition)
```
其真假判断条件可以参考IF指令。

###FOREACH指令
FOREACH指令的使用方法有三种形式: 
(1)列表
```
FOREACH(loop_vararg1 arg2 …)
    COMMAND1(ARGS…)
    COMMAND2(ARGS…)
    …
ENDFOREACH(loop_var)
```

像我们前面使用的AUX_SOURCE_DIRECTORY的例子
```
AUX_SOURCE_DIRECTORY(.SRC_LIST)
FOREACH(F ${SRC_LIST})
    MESSAGE(${F})
ENDFOREACH(F)
```

(2)范围

```
FOREACH(loop_var RANGE total)

ENDFOREACH(loop_var)
```

从0到total以1为步进，举例如下:

```
FOREACH(VARRANGE 10)
    MESSAGE(${VAR})
ENDFOREACH(VAR)
```
最终得到的输出是:

0
1
2
3
4
5
6
7
8
9
10

(3)范围和步进
```
FOREACH(loop_var RANGE start stop [step])

ENDFOREACH(loop_var)
```
从start开始到stop结束，以step为步进。举例如下:
```
FOREACH(A RANGE 5 15 3)
    MESSAGE(${A})
ENDFOREACH(A)
```

最终得到的结果是:

5
8
11
14

这个指令需要注意的是，直到遇到ENDFOREACH指令，整个语句块才会得到真正的执行。

复杂的例子:模块的使用和自定义模块
这里将着重介绍系统预定义的Find模块的使用以及自己编写Find模块，系统中提供了其他各种模块，一般情况需要使用INCLUDE指令显式的调用，FIND_PACKAGE指令是一个特例，可以直接调用预定义的模块。

其实纯粹依靠cmake本身提供的基本指令来管理工程是一件非常复杂的事情，所以cmake设计成了可扩展的架构，可以通过编写一些通用的模块来扩展cmake。

首先介绍一下cmake提供的FindCURL模块的使用，然后基于前面的libhello共享库，编写一个FindHello.cmake模块。 
（一）使用FindCURL模块 
在/backup/cmake目录建立t5目录,用于存放CURL的例子。 
建立src目录,并建立src/main.c,内容如下:
```
#include<curl/curl.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

FILE*fp;

intwrite_data(void *ptr, size_t size, size_t nmemb, void *stream)

{
    int written = fwrite(ptr, size, nmemb, (FILE *)fp);
    return written;
}

int main()
{
    const char *path = “/tmp/curl-test”;
    const char *mode = “w”;
    fp = fopen(path,mode);
    curl_global_init(CURL_GLOBAL_ALL);
    CURL coderes;
    CURL *curl = curl_easy_init();

    curl_easy_setopt(curl, CURLOPT_URL, “http://www.linux-ren.org”);
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_data);
    curl_easy_setopt(curl, CURLOPT_VERBOSE, 1);
    res = curl_easy_perform(curl);
    curl_easy_cleanup(curl);
}
```

这段代码的作用是通过curl取回www.linux-ren.org的首页并写入/tmp/curl-test文件中

建立主工程文件CmakeLists.txt，如下：
```
PROJECT(CURLTEST)
ADD_SUBDIRECTORY(src)
```

建立src/CmakeLists.txt
```
ADD_EXECUTABLE(curltest main.c)
```

现在自然是没办法编译的，我们需要添加curl的头文件路径和库文件。

方法1: 
直接通过INCLUDE_DIRECTORIES和TARGET_LINK_LIBRARIES指令添加:

我们可以直接在src/CMakeLists.txt中添加:
```
INCLUDE_DIRECTORIES(/usr/include)
TARGET_LINK_LIBRARIES(curltestcurl)
```

然后建立build目录进行外部构建即可。 
现在要探讨的是使用cmake提供的FindCURL模块。

方法2: 
使用FindCURL模块。向src/CMakeLists.txt中添加:
```
FIND_PACKAGE(CURL)
IF(CURL_FOUND)
    INCLUDE_DIRECTORIES(${CURL_INCLUDE_DIR})
    TARGET_LINK_LIBRARIES(curltest${CURL_LIBRARY})
ELSE(CURL_FOUND)
    MESSAGE(FATAL_ERROR”CURL library not found”)
ENDIF(CURL_FOUND)
```

对于系统预定义的Find<name>.cmake模块,使用方法一般如上例所示，每一个模块都会定义以下几个变量:
```
<name>_FOUND
<name>_INCLUDE_DIR or <name>_INCLUDES
<name>_LIBRARY or <name>_LIBRARIES
```

可以通过<name>_FOUND来判断模块是否被找到，如果没有找到，按照工程的需要关闭某些特性、给出提醒或者中止编译，上面的例子就是报出致命错误并终止构建。 
如果<name>_FOUND为真，则将<name>_INCLUDE_DIR加入INCLUDE_DIRECTORIES，将<name>_LIBRARY加入TARGET_LINK_LIBRARIES中。

我们再来看一个复杂的例子,通过<name>_FOUND来控制工程特性:
```
SET(mySourcesviewer.c)
SET(optionalSources)
SET(optionalLibs)

FIND_PACKAGE(JPEG)

IF(JPEG_FOUND)
    SET(optionalSources${optionalSources} jpegview.c)
    INCLUDE_DIRECTORIES(${JPEG_INCLUDE_DIR} )
    SET(optionalLibs${optionalLibs} ${JPEG_LIBRARIES} )
    ADD_DEFINITIONS(-DENABLE_JPEG_SUPPORT)
ENDIF(JPEG_FOUND)

IF(PNG_FOUND)
    SET(optionalSources${optionalSources} pngview.c)
    INCLUDE_DIRECTORIES(${PNG_INCLUDE_DIR} )
    SET(optionalLibs${optionalLibs} ${PNG_LIBRARIES} )
    ADD_DEFINITIONS(-DENABLE_PNG_SUPPORT)
ENDIF(PNG_FOUND)

ADD_EXECUTABLE(viewer${mySources} ${optionalSources}
TARGET_LINK_LIBRARIES(viewer${optionalLibs}
```

通过判断系统是否提供了JPEG库来决定程序是否支持JPEG功能。

（二）编写属于自己的FindHello模块 
接下来在t6示例中演示如何自定义FindHELLO模块并使用这个模块构建工程。在/backup/cmake/中建立t6目录，并在其中建立cmake目录用于存放我们自己定义的FindHELLO.cmake模块，同时建立src目录，用于存放我们的源文件。

1.定义cmake/FindHELLO.cmake模块
```
FIND_PATH(HELLO_INCLUDE_DIR hello.h /usr/include/hello /usr/local/include/hello)

FIND_LIBRARY(HELLO_LIBRARY NAMES hello PATH /usr/lib /usr/local/lib)

IF(HELLO_INCLUDE_DIR AND HELLO_LIBRARY)
    SET(HELLO_FOUNDTRUE)
ENDIF(HELLO_INCLUDE_DIR AND HELLO_LIBRARY)

IF(HELLO_FOUND)
    IF(NOT HELLO_FIND_QUIETLY)
        MESSAGE(STATUS"Found Hello: ${HELLO_LIBRARY}")
    ENDIF(NOT HELLO_FIND_QUIETLY)
ELSE(HELLO_FOUND)
    IF(HELLO_FIND_REQUIRED)
        MESSAGE(FATAL_ERROR"Could not find hello library")
    ENDIF(HELLO_FIND_REQUIRED)
ENDIF(HELLO_FOUND)
```

针对上面的模块让我们再来回顾一下FIND_PACKAGE指令:
```
FIND_PACKAGE(<name>[major.minor] [QUIET] [NO_MODULE] [[REQUIRED|COMPONENTS][componets...]])
```
前面的CURL例子中我们使用了最简单的FIND_PACKAGE指令，其实它可以使用多种参数：

QUIET参数，对应与我们编写的FindHELLO中的HELLO_FIND_QUIETLY，如果不指定这个参数,就会执行:
```
MESSAGE(STATUS"Found Hello: ${HELLO_LIBRARY}")
```

REQUIRED参数，其含义是指这个共享库是否是工程必须的，如果使用了这个参数，说明这个链接库是必备库,如果找不到这个链接库,则工程不能编译。对应于FindHELLO.cmake模块中的HELLO_FIND_REQUIRED变量。 
同样,我们在上面的模块中定义了HELLO_FOUND,HELLO_INCLUDE_DIR, HELLO_LIBRARY变量供开发者在FIND_PACKAGE指令中使用。

下面建立src/main.c,内容为:
```
#include<hello.h>
int main()
{
    HelloFunc();
    return 0;
}
```

建立src/CMakeLists.txt文件,内容如下:
```
FIND_PACKAGE(HELLO)
IF(HELLO_FOUND)
    ADD_EXECUTABLE(hellomain.c)
    INCLUDE_DIRECTORIES(${HELLO_INCLUDE_DIR})
    TARGET_LINK_LIBRARIES(hello${HELLO_LIBRARY})
ENDIF(HELLO_FOUND)
```

为了能够让工程找到 FindHELLO.cmake 模块(存放在工程中的cmake目录)我们在主工程文件 CMakeLists.txt 中加入:
```
SET(CMAKE_MODULE_PATH${PROJECT_SOURCE_DIR}/cmake)
```

（三）使用自定义的FindHELLO模块构建工程 
仍然采用外部编译的方式,建立build目录,进入目录运行:
```
cmake ..
```
我们可以从输出中看到:
```
FoundHello: /usr/lib/libhello.so
```
如果我们把上面的FIND_PACKAGE(HELLO)修改为FIND_PACKAGE(HELLO QUIET),

不会看到上面的输出。接下来就可以使用make命令构建工程,运行:
```
./src/hello
```
可以得到输出
```
HelloWorld
```
说明工程成功构建。

（四）如果没有找到hellolibrary呢? 
我们可以尝试将/usr/lib/libhello.x移动到/tmp目录,这样按照FindHELLO模块的定义,找不到hellolibrary了,我们再来看一下构建结果:
```
cmake ..
```
仍然可以成功进行构建,但是这时候是没有办法编译的。

修改FIND_PACKAGE(HELLO)为FIND_PACKAGE(HELLO REQUIRED)，将hellolibrary定义为工程必须的共享库。 
这时候再次运行
```
cmake ..
```
我们得到如下输出:
```
CMakeError: Could not find hello library.
```
因为找不到libhello.x，所以,整个Makefile生成过程被出错中止。

一些问题
1.怎样区分debug、release版本 
建立debug/release两目录，分别在其中执行cmake -D CMAKE_BUILD_TYPE=Debug（或Release），需要编译不同版本时进入不同目录执行make即可：

Debug版会使用参数-g；
Release版使用-O3–DNDEBUG

另一种设置方法——例如DEBUG版设置编译参数DDEBUG
```
IF(DEBUG_mode)
    add_definitions(-DDEBUG)
ENDIF()
```
在执行cmake时增加参数即可，例如cmake -D DEBUG_mode=ON

2.怎样设置条件编译 
例如debug版设置编译选项DEBUG，并且更改不应改变CMakelist.txt 
使用option command，eg：
```
option(DEBUG_mode"ON for debug or OFF for release" ON)
IF(DEBUG_mode)
    add_definitions(-DDEBUG)
ENDIF()
```

使其生效的方法：首先cmake生成makefile，然后make edit_cache编辑编译选项；Linux下会打开一个文本框，可以更改，改完后再make生成目标文件——emacs不支持make edit_cache；

局限：这种方法不能直接设置生成的makefile，而是必须使用命令在make前设置参数；对于debug、release版本，相当于需要两个目录，分别先cmake一次，然后分别makeedit_cache一次；

期望的效果：在执行cmake时直接通过参数指定一个开关项，生成相应的makefile。
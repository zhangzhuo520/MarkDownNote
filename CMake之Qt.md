#CMake构建QT工程
----
##包含QT的库
FIND_PACKAGE(Qt4 REQUIRED)
INCLUDE(${QT_USE_FILE})

##生成QT的执行文件的语句
ADD_EXECUTABLE(test1 ${SOURCE}) //生成test1可执行文件
TARGET_LINK_LIBRARIES(test1 ${QT_QTCORE_LIBRARY})（可执行文件test1链接相关的Qt库）
**注意：**这两语句的生成文件（test1）名称需要相同

##下面是一个完整的例子
```CMake
project(test) // 工程名称
cmake_minimum_required(VERSION 2.8.12.2) //限定的CMake的最低版本号
FIND_PACKAGE(Qt4 REQUIRED)  //查找QT相关的库
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})//指定文件的搜索路径（g++ -l）
INCLUDE(${QT_USE_FILE})
ADD_DEFINITIONS(${QT_DEFINITIONS}) //相当g++ -D选选项（添加QT中的宏）
set(SOURCE mainwindow.cpp main.cpp)

#if 1
set(CMAKE_AUTOMOC ON) //自动生成Moc文件，这个对情况某些可能没有效果。
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON) 
#else
QT4_WRAP_CPP(MOCS mainwindow.cc) // 含有object宏的类
QT4_ADD_RESOURCES(QRC ${main.qrc})//资源文件
QT4_WRAP_UI(UI ${mianwindow.ui}) // ui文件
#end


ADD_EXECUTABLE(test1 ${SOURCE}) //生成可执行文件
TARGET_LINK_LIBRARIES(test1 ${QT_QTCORE_LIBRARY} ${QT_QTGUI_LIBRARY} ${QT_LIBRARIES}) //可执行文件链接到库
```
**注意**：CMake2.8.12只支持CMAKE_AUTOMOC，不支持CMAKE_AUTOUIC，CMAKE_AUTORCC。
**注意**：宏判断上面的两种添加特殊文件的方式任选一种即可（第一种优点是比较方便，不需要每个文件都有去添加，缺点是对版本有要求，以及对于一些特殊的情况不支持， 第二种比较稳定，但是很麻烦）

##使用到QT中的模块
```
SET(QT_USE_QTOPENGL TRUE)
```
常用的模块有：
```
                                     QT_USE_QTNETWORK
                                     QT_USE_QTOPENGL
                                     QT_USE_QTSQL
                                     QT_USE_QTXML
                                     QT_USE_QTSVG
                                     QT_USE_QTTEST
                                     QT_USE_QTDBUS
                                     QT_USE_QTSCRIPT
                                     QT_USE_QTWEBKIT
                                     QT_USE_QTXMLPATTERNS
                                     QT_USE_PHONON

```


##Qt插件开发
&emsp;&emsp;添加SHARED关键字，再将生成的动态库放到默认的文件下面
ADD_LIBRARY(EchoPlugin SHARED ${SOURCES})
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/plugins)
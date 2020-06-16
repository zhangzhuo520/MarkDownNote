#从qt编程看内存分区
---------------------
##网络上流形两大版本内存分区，分别为：

1. 五大内存分区：堆、栈、全局/静态存储区、自由存储区和常量存储区。
2. 五大内存分区：堆、栈、全局/静态存储区、字符串常量区和代码区。

且不论以上两种分区孰是孰非，孰优孰劣，我认为具体的内存分区和编译器有很大关系，我想不同编译器对内存的划分都不尽相同，但也大同小异。
综合对比，查阅相关资料，提出自己对C/C++程序的内存分区的认识。可划分为四大内存分区：堆、栈、静态存储区和代码区。

###堆区：
由程序猿手动申请，手动释放，若不手动释放，程序结束后由系统回收，生命周期是整个程序运行期间。使用malloc或者new进行堆的申请，堆的总大小为机器的虚拟内存的大小。
说明：new操作符本质上是使用了malloc进行内存的申请，new和malloc的区别如下：
（1）malloc是C语言中的函数，而new是C++中的操作符。
（2）malloc申请之后返回的类型是void*，而new返回的指针带有类型。
（3）malloc只负责内存的分配而不会调用类的构造函数，而new不仅会分配内存，而且会自动调用类的构造函数。

###栈区：
由系统进行内存的管理。主要存放函数的参数以及局部变量。在函数完成执行，系统自行释放栈区内存，不需要用户管理。整个程序的栈区的大小可以在编译器中由用户自行设定，VS中默认的栈区大小为1M，可通过VS手动更改栈的大小。64bits的Linux默认栈大小为10MB，可通过ulimit -s临时修改。

##静态存储区：
静态存储区内的变量在程序编译阶段已经分配好内存空间并初始化。这块内存在程序的整个运行期间都存在，它主要存放静态变量、全局变量和常量。
注意：
（1）这里不区分初始化和未初始化的数据区，是因为静态存储区内的变量若不显示初始化，则编译器会自动以默认的方式进行初始化，即静态存储区内不存在未初始化的变量。
（2）静态存储区内的常量分为常变量和字符串常量，一经初始化，不可修改。静态存储内的常变量是全局变量，与局部常变量不同，区别在于局部常变量存放于栈，实际可间接通过指针或者引用进行修改，而全局常变量存放于静态常量区则不可以间接修改。
（3）字符串常量存储在静态存储区的常量区，字符串常量的名称即为它本身，属于常变量。
（4）数据区的具体划分，有利于我们对于变量类型的理解。不同类型的变量存放的区域不同。后面将以实例代码说明这四种数据区中具体对应的变量。

###代码区：
存放程序体的二进制代码。比如我们写的函数，都是在代码区的。

示例代码：
```
int a = 0;//静态全局变量区
char *p1; //编译器默认初始化为NULL
void main()
{
    int b; //栈
    char s[] = "abc";//栈
    char *p2 = "123456";//123456在字符串常量区，p2在栈上
    static int c =0; //c在静态变量区，0为文字常量，在代码区
    const int d=0; //栈
    static const int d;//静态常量区
    p1 = (char *)malloc(10);//分配得来得10字节在堆区。
    strcpy(p1, "123456"); //123456放在字符串常量区，编译器可能会将它与p2所指向的"123456"优化成一个地方
}
```
数据区包括：堆，栈，静态存储区。
静态存储区包括：常量区（静态常量区），全局区（全局变量区）和静态变量区（静态区）。
常量区包括：字符串常量区和常变量区。
代码区：存放程序编译后的二进制代码，不可寻址区。

可以说，C/C++内存分区其实只有两个，即代码区和数据区。

###qt中的堆栈：


```
mainwindow.h
#ifndef MAINWINDOW_H
#define MAINWINDOW_H
#include <QtGui>
class MainWindow:public QWidget//从QWidget继承
{
Q_OBJECT
public:
MainWindow();//构造函数
};
#endif

mainwindow.cpp
#include "mainwindow.h"
MainWindow::MainWindow()
{
 QLabel *label;//定义QLabel指针                       stack中
 QPushButton button("OK");                          stack中
//QPushButton *button=new QPushButton("OK");注意和上一句区别
 button.show();//按钮显示出来
 QHBoxLayout *layout;//定义QHBoxLayout指针            stack中
 label=new QLabel(tr("Hello Qt"),this);//实例化QLabel heap中
 layout=new QHBoxLayout(this);//实例化QHBoxLayout     heap中
 layout->addWidget(label);//给Layout添加Widget
 layout->addWidget (&button);
 this->setLayout(layout);//设置Layout为layout
}

main.cpp
#include <QtGui>
#include "mainwindow.h"
int main(int argc,char *argv[])
{
QApplication *app;//应用程序类指针                      stack中
app=new QApplication(argc,argv); //实例化应用程序类      heap中
MainWindow winmain;//实例化MainWindow                 stack中
winmain.show();//弹出窗口
return app->exec();//返回
} 
```
当使用QPushButton button("OK")，编译运行该程序后发现并没有这个Button。构造时写了Button但运行时没有。但将这句话改为QPushButton *button=new QPushButton("OK");编译运行就通过了。问题就这样产生了。如果是QPushButton button("OK");出来的，在构造之后由于在栈区，系统会自动释放，所以运行时没有显示出来，而用QPushButton *button=new QPushButton("OK");出来的，由于在堆区，程序员没有释放，系统在程序退出后才会释放，所以就会显示在界面上了。从这个很小的例子上可以看出堆与栈的区别。

懒惰不会让你一下子跌到 但会在不知不觉中减少你的收获； 勤奋也不会让你一夜成功 但会在不知不觉中积累你的成果 越努力，越幸运。
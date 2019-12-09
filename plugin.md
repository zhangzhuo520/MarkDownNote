# QT的插件编写
&emsp;&emsp;随着应用程序开发的规模越来越大。功能模块数量越来越多，我们对程序的管理也没有那么的方便。这时候可以通过插件化的开发来进行软件模块化的开发。插件开发的好处多多。不仅方便管理，而且时候多人同时开发不同的模块。然后通过插件管理把插件加载到主程序中。极其方便。
## QT插件简介
&emsp;&emsp;Qt的插件分为高等级的API插件，和低等级的API插件。
&emsp;&emsp;**Higher-Lever API**表示的是QT框架中自带的一些插件，我们可以继承他的这些插件，然后实现我们的功能。例如:     QAccessibleBridgePlugin，QFontEnginePlugin，,QImagePlugin, QStylePlugin等；  
#### myplugin.h
```
#include <QStylePlugin>
class MyStylePlugin : public QStylePlugin
{
public:
    QStringList keys() const;
    QStyle *create(const QString &key);
};
```
#### myplugin.cc
```
#include "mystyleplugin.h"
#include <QProxyStyle>
QStringList MyStylePlugin::keys() const
{
    return QStringList() << "MyStyle";
}

QStyle *MyStylePlugin::create(const QString &key)
{
    if (key.toLower() == "mystyle")
        return new QProxyStyle;
    return 0;
}

Q_EXPORT_PLUGIN2(pnp_mystyleplugin, MyStylePlugin)
```
#### main.cc
    QApplication::setStyle(QStyleFactory::create("MyStyle"));

&emsp;&emsp;**Lower-Level API**表示我们自己创建的一些插件，这些插件可以是我们自己设计的任意功能。不局限于QT框架的插件类；
&emsp;&emsp;&emsp;&emsp;**使用插件**的实现步骤如下：
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;1.定义好一组只有纯虚函数的类，用于插件通信。
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;2.使用Q_DECLARE_INTERFACE告诉元对象系统这个插件接口类。
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;3.使用QPluginLoader加载插件。
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;4.使用qobject_cast()测试插件是否能使用。
&emsp;&emsp;&emsp;&emsp;**编写插件**的实现步骤如下：
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;1.定义一个类，这个类继承QObject和纯虚函数的插件通信类。
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;2.使用Q_INTERFACE（）告诉元对象系统这个插件接口类。
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;3.使用Q_EXPORT_PLUGIN2()导出插件。
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;4.使用合适的.pro文件构建插件。

代码如下：
interface.h
```
 class FilterInterface
 {
 public:
     virtual ~FilterInterface() {}

     virtual QStringList filters() const = 0;
     virtual QImage filterImage(const QString &filter, const QImage &image,
                                QWidget *parent) = 0;
 };
```
plugin.cc
```
#include <QObject>
#include <QStringList>
#include <QImage>

#include <plugandpaint/interfaces.h>

 class ExtraFiltersPlugin : public QObject, public FilterInterface
 {
     Q_OBJECT
     Q_INTERFACES(FilterInterface)

 public:
     QStringList filters() const;
     QImage filterImage(const QString &filter, const QImage &image,
                        QWidget *parent);
 };
```						
&emsp;&emsp;上面说的插件是将功能模板编译成一个动态库，动态库是单独发布的，并且在运行的时候进行检测和加载。还有一种插件是静态插件，静态就相当于静态库了，必须跟着程序一起发布。必须和程序一起构建和编译。优点是部署不容易出错。例如QT中的这几种插件：qtaccessiblecompatwidgets，qgif，qjpeg等。
&emsp;&emsp;使用静态插件必须要使用Q_IMPORT_PLUGIN将插件添加系统中去。创建静态插件的步骤如下：
&emsp;&emsp;1.在pro文件中添加CONFIG += static
&emsp;&emsp;2.在应用中使用Q_IMPORT_PLUGIN（）
&emsp;&emsp;3.在.pro文件中将LIBS和插件链接起来.

代码如下:
cc:

```
 #include <QApplication>
 #include <QtPlugin>

 Q_IMPORT_PLUGIN(qjpeg)
 Q_IMPORT_PLUGIN(qgif)
 Q_IMPORT_PLUGIN(qkrcodecs)

 int main(int argc, char *argv[])
 {
     QApplication app(argc, argv);
     ...
     return app.exec();
 }
```
pro:

```
 QTPLUGIN     += qjpeg \
                 qgif \
                 qkrcodecs
```

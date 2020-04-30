##Qt之QObject的理解：

Qt的机制会把一个类的私有变量单独封装一下取名为(className)Private,因此我们分析QObject源码的时候也会从两个地方分析。
```
定义符号的意思：
||
|| 表示继承关系                --------表示有一个（对象中的成员） 
||
                                    
                                                
                                                 QMeteObject(signal/slot), all kinds of flags,
QObjectDate ------------------------------------>QObjet*（this）,QObject*(parent), QOjectList 
      继承           
      ||
      ||
      ||
      ||
      ||                有一个                                               
 QObjectPrivate  <-------------------------  QObject ---------------> Q_OBJECT ------> QMeteObject...等一些信号与槽的相关的文件
        |
        |
        |
        |
        |
        V
struct connection     |
struct connectionList |for signal and slots
struct sender         |
objectName
extradata  (for user set property data)
QThreadData (for Qthread and object relationship)
QList<Object*> EventFilters(for eventfilters) 
QAtomicPointer<QtSharedPointer::ExternalRefCountData> sharedRefcount;(for epress object was delete)
           
```
完全分析下来QObject3个方面得功能（信号与槽的部分， object关系处理， object与线程）

###object与线程
主要分析QThreadData
```
QThreadData-----------------> thread（线程）,threadid(线程id), looplevel（事件list的count）, 
      |                       QAbstractEventDispatcher(事件派发)，QStack<QEventLoop *> eventLoops（事件循环列表），
      |                       QPostEventList postEventList（按照优先级插入事件的list）,其他的flag，变量等。
      |                       FlaggedDebugSignatures（提供了每个线程变为QThreadDate的存储方式）
      |
      |
      |
      V
QAbstractEventDispatcher （封装各种事件，事件调度）
根据不同的平台Qt分了不同的事件调度器

QEventDispatcherGlib          使用glib事件循环，有助于和Gtk的集成
QEventDispatcherUNIX          默认的glib不可用时，就用这个
QEventDispatcherWin32         Qt 创建一个带回调函数的隐藏窗口来处理事件。

在这几个类里面可以看到
Timer  SockerNotifer  他们提供注册、反注册功能。并将系统底层对应事件转换成Qt事件。


```
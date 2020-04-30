####RTTI
----
    应用运行的时可以实时获取对象类型
 C++中使用dynamic_cast和typeid进行运行时类型识别
 Qt提供另外两种运行时的识别方法 qobject_cast 和 QObject::inherits()。
 看名字就可以知道，这两个方法都是基于QObject的，也就是元对象系统。 
```
if (QLabel *label = qobject_cast<QLabel *>(obj))
{
    label->setText(tr("Ping"));
}
else if (QPushButton *button = qobject_cast<QPushButton *>(obj))
{
    button->setText(tr("Pong!"));
}
```
####据说，qobject_cast的速度比dynamic_cast的速度快很多。
####QObject::inherits(const char *className)的速度相对慢一些，所以尽可能使用qobject_cast。


####CONFIG += no_keywords
####它告诉QT不要定义moc关键字signals,slots和emit，因为这些名字可能将被用于第三方库，例如Boost。你只需简单的用QT宏将他们替换为 Q_SIGNALS, Q_SLOTS，和 Q_EMIT，就可以继续使用信号和槽了。


####Qt的鲜为人知的小功能
1. Q_GADGET：不需要从QObject继承就可以使用Qt的Meta Object功能
2. Q_ENUMS：对枚举对象进行反射（reflection）
3. Q_SIGNAL：不需要signals关键字就可以将一个函数指定为singal函数
4. Q_SLOT：不需要slots关键字就可以将一个函数指定为slot函数
5. Q_FOREACH：Qt提供的关键字foreach在配置文件里面定义了no_keywords的时候就失效了，而Q_FOREACH宏无论何时都有效
6. Q_FOREVER：原因同Q_FOREACH，用法和关键字forever一样
7. qChecksum：计算基于CRC-16-CCITT算法的CRC-16校验码
8. qCompress/qUncompress：基于zlib的算法对字节缓存区进行压缩/解压，可以自定义压缩比
9. qDeleteAll：对容器或者迭代器中的每个对象进行delete操作
10. qRound/qRound64：对浮点数取整，四舍五入
11. qFuzzyCompare：线程安全的浮点数模糊比较，处理了浮点精度误差的问题
12. qVersion：获取当前正在运行的Qt的版本号
13. sender：在slot函数中调用可以获取发送该signal的对象。仅用于来自同一线程的Qt::DirectConnection连接的 signal。由于这种做法破坏了面向对象的原则，慎用！
14. QSysInfo：获取当前运行的系统信息，包括版本号等，Mac/Symbian/Windows均适用
15. QtEndian：处理不同体系架构下字节存储次序不同的问题
16. QObject::findChildren<>：找出隐藏的子对象
17. qobject_cast：不需要编译器RTTI的支持就可以实现C++标准库dynamic_cast的功能
18. QObject::deleteLater：在事件处理循环中调用，则在事件处理循环结束时删除自己；在事件处理循环外调用，则在下一个事件处理循环开始时删除自己
19. qPrintable：类似QString::toLocal8Bit()::constData()，但是更好记
20. qDebug()：类似Windows下的TRACE或者OutputDebugString
21. connect(SomeObj, SIGNAL(foo()), SIGNAL(bar())：通过一个signal自动触发另一个signal。

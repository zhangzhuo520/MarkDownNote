#Qt多线程与界面通信的方式
------------------------
1.通过信号与槽
线程执行完直接发送信号，通知主界面刷新。

2.通过postEvent发送事件，刷新信息。
postEvent(QObject *, QEvent *)

3.通过QMetaObject::invokeMothe（）调用。
QMetaObject::invokeMethod(thread, "quit", Qt::QueuedConnection);
注意，要调用的类型必须是信号、槽，以及Qt元对象系统能识别的类型， 如果不是信号和槽，可以使用qRegisterMetaType（）来注册数据类型。此外，使用Q_INVOKABLE来声明函数，也可以正确调用。
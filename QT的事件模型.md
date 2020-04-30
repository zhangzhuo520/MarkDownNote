#关于QT事件机制的学习总结
----
&emsp;&emsp;Qt的核心在于QT的事件驱动机制。而整个事件的机制无非三个部分，事件的产生，事件的处理，事件的分发。（只有继承QOBject的类才能接收和使用事件。

##事件的产生
&emsp;&emsp;事件分为**操作系统事件**和**应用内部事件**，例如鼠标，键盘。。等属于由操作系统产生，外部输入事件，应用内部产生的事件有resize，paint等， 还有一种是自己子类化QEvent定义的事件。也属于内部事件。事件最开始的定义就是一堆纯数据信息，不会有其他的逻辑操作。（类比于短信，信息等）。
![](https://i.imgur.com/bJtNbUH.png)

	
####操作系统事件
 &emsp;&emsp操作系统产生的事件 keyPressEvent，keyReleaseEvent，mousePressEvent，mouseReleaseEvent，事件重操作系统传过来之后，QT将其封装成Qevent，发送到事件队列中。

####应用程序产生的事件
- sendEvent
  &emsp;&emsp;同步事件，事件产生后会立即被派发和处理，QWidget::repaint()就是这种类型的事件。同步事件存储再栈上，使用完了马上会被销毁。


- postEvent
  &emsp;&emsp;异步事件，事件产生后会放入事件队列中，依次等待处理。例如绘图update()，会new出来一个paintEvent事件,通过PostEvent放到事件队列中，等待处理。异步事件只支持分配在堆上的事件对象，事件被处理后由Qt平台销毁。

##事件的处理
&emsp;&emsp;当系统事件和应用程序产生的post事件进入事件队列后（sendEvent直接就派发给了相关控件对象），QT会先循环处理完所有的应用post事件，然后在处理系统事件，最后在处理系统产生的post事件，postEvent相对于sendEvent事件有一个缓冲的过程，比如说一秒钟调用10个update()，产生10个postEvent事件，那么事件队列中检测到有10个相同事件，只会处理一次，而sendEvent就会处理10次。
**（先处理post事件，在处理系统事件，最后处理系统事件产生的post事件）**

##事件的派发
####窗口事件的传递
下面是事件生成到发送给操作系统，到回到QT，到到达子控件的一个详细点的过程（windows平台为例）
- QApplication::exec()
	进入事件循环

- QCoreApplication::exec() 
	进入QCoreApplication循环

- QEventLoop::exec(ProcessEventsFlags ) 
	进入QeventLoop事件循环  
 
- QEventLoop::processEvents(ProcessEventsFlags ) 
    处理事件

- QEventDispatcherWin32::processEvents(QEventLoop::ProcessEventsFlags) 
	事件处理将事件打包发送为消息，发送给操作系统 

- QT_WIN_CALLBACK QtWndProc(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam)
	操作系统将事件发回给QT平台

- bool QETWidget::translateMouseEvent(const MSG &msg)   
	将操作系统打包的事件解包、翻译为QApplication可识别的事件

- bool QApplicationPrivate::sendMouseEvent(...)   
- inline bool QCoreApplication::sendSpontaneousEvent(QObject *receiver, QEvent *event)   
	根据事件类型发送事件

- bool QCoreApplication::notifyInternal(QObject *receiver, QEvent *event)
	线程内部处理事件

- bool QApplication::notify(QObject *receiver, QEvent *e)  
	QApplication事件的派发 

- bool QApplicationPrivate::notify_helper(QObject *receiver, QEvent * e) 
    线程内事件的处理
 
- bool QWidget::event(QEvent *event)
	相关控件事件的处理

- QMousePressEvent， QPaintEvent
	最后到达事件的相关函数

####非窗口事件的传递

简单一点的可以看下面的过程

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;  ![](https://i.imgur.com/WCaMY9J.png)
**因此，我们想要截获处理事件有一下4个地方。**
	
&emsp;&emsp;1.子类化QApplication，重写notify函数
&emsp;&emsp;2.使用事件过滤器installEventFilter（），重写EventFilter（）函数
&emsp;&emsp;3.重写QT控件的Event函数，获取控件的事件。
&emsp;&emsp;4.在控件的函数事件（KeyPressEvent(QKeyEvent事件)）中处理事件。

###关于事件的传递
&emsp;&emsp;对于某些控件的的事件, 如果在整个事件的派发过程结束后还没有被处理, 那么这个事件将会向上转发给它的父widget，直到最顶层窗口。
&emsp;&emsp;比如：事件可能最先发送给QCheckBox, 如果QCheckBox没有处理, 那么由QGroupBox接着处理；
如果QGroupBox仍然没有处理, 再送到QDialog, 因为QDialog已经是最顶层widget, 所以如果QDialog再不处理, QEvent将停止转发。

&emsp;&emsp;如果得到事件的对象，调用了accept()，则事件停止继续传播；如果调用了ignore()，事件向上一级继续传播。

&emsp;&emsp;Qt对自定义事件处理函数的默认返回值是accept()，但默认的事件处理函数是ingore()。因此，如果要继续向上传播，调用QWidget的默认处理函数即可。到此为止的话，不必显式调用accept()。但在event处理函数里，返回true表示accept，返回false表示向上级传播。在closeEvent是个特殊情形，accept表示quit，ignore表示取消，所以最好在closeEvent显式调用accept和ignore。
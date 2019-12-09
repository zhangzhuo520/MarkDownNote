#QPainter的使用：
------------

###painter绘制小部件只能在paintEvent里面绘制
Warning: When the paintdevice is a widget, QPainter can only be used inside a paintEvent() function or in a function called by paintEvent().

这才明白，QPainter 默认只能在 paintEvent 里面调用，在别处绘制就会遇到警告。另外，在构造函数中设置属性：

this->setAttribute(Qt::WA_PaintOutsidePaintEvent);　

是可以在paintEvent()之外绘制；但不幸的是，该属性只支持X11，对于 Windows、Mac OS X、Embedded Linux 均不予支持。

###用QImage做paintDevice遇到的问题

背景： 在画布上用鼠标press 画一个点。点数据保存到GUI上的Table里面。画布自适应大小。

产生问题：鼠标press的时候，画不上去，提示：
```
QPainter::begin: A paint device can only be painted by one painter at a time.
QPainter::setRenderHint: Painter must be active to set rendering hints
QPainter::setBrush: Painter not active
QPainter::begin: A paint device can only be painted by one painter at a time.
QPainter::setRenderHint: Painter must be active to set rendering hints
```
等等之类的错误

后来经过分析之后。发现代码中有两次使用QPainter painter(QImage *); painter.drawpoint();
产生的原因：
(1)QPainter painter(QImage *); painter.drawpoint();两个地方同时执行，导致QImage也就是paintDevice同时有两个Painter冲突。
(2)为什么会同时执行：因为两段代码都是比较耗时的操作。painter前一个painter来不及释放后一个painter就执行了；

解决方案：
1. 换一种写法，主动释放。 QPainter paiter; painter.begin(); painter.end();
2. 同步两个函数。或者加锁。让他们不同时执行。

##总结：
 1.painter绘制小部件只能在paintEvent里面使用。如果要在别处设置属性(Qt::WA_PaintOutsidePaintEvent）
 2.painter使用非widget做画图设备的时候。最好使用 painter.begin() painter.end();


#QPainter的使用：
------------

###1.painter绘制小部件只能在paintEvent里面绘制
Warning: When the paintdevice is a widget, QPainter can only be used inside a paintEvent() function or in a function called by paintEvent().

这才明白，QPainter 默认只能在 paintEvent 里面调用，在别处绘制就会遇到警告。另外，在构造函数中设置属性：

this->setAttribute(Qt::WA_PaintOutsidePaintEvent);　

是可以在paintEvent()之外绘制；但不幸的是，该属性只支持X11，对于 Windows、Mac OS X、Embedded Linux 均不予支持。

###2.用QImage做paintDevice遇到的问题

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

##关于QPainter的双缓冲的使用

两张image（image, temp_image）,一个flage bool is_draw, 两个点start_point, end_point
```
mousePressEvent(QMouseEvent *e)
{
   is_draw = true;
   start_point = e.pos();
}

mouseReleaseEvent(QMouseEvent *e)
{
   temp_image = image;
   is_draw = false;
}


mouseMoveEvent(QMouseEvent *e)
{
    end_pos = e.pos();
}

void paintEvent(QPaintEvent *e)
{
    if(!is_drawing) return;（1.如果不画，直接返回，不用每次都画, 2.防止反走样不生效（坑））
    QPainter paint;
    image = temp_image；(每次画的时候重空白的图上面获取，不会产生重影，双缓冲的核心)
    paint.begin(&image);
    paint.setRenderhinit(QPainter::Antialiasing, true);
    paint.drawline(start_point, end_point);
}

```

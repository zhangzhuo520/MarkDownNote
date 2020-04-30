## stack滑动窗口界面
&emsp;窗口滑动界面相对于普通的stackWidget窗口管理多了一些动画效果，可以让界面看起来更加酷炫，如图；
![](https://i.imgur.com/aiUBDKU.gif)


###滑动窗口的原理
&emsp滑动stackwidget的本质就是采用动画效果修饰widget的pos属性，同时移动前后的两个widget的位置，等动画完成之后，最后隐藏前一个widget。整个滑动的效果就达到啦。
核心代码如下:
```
void SlidingStackedWidget::slideInWgt(QWidget * newwidget,SlideDirection  direction) {
    
	if (m_active) {  //控制下是否需要滑动
        return;
	}

	else m_active=true;

	enum SlideDirection directionhint;
    int now=currentIndex();
	int next=indexOf(newwidget);
	if (now==next) {
		m_active=false;
		return;
	}
	else if (now<next){
		directionhint=m_vertical ? TopToBottom : RightToLeft;
	}
	else {
		directionhint=m_vertical ? BottomToTop : LeftToRight;
	}
	if (direction == Automatic) {
		direction=directionhint;
	}
    
    //获得frame的size 
    int offsetx=frameRect().width();
    int offsety=frameRect().height();
    
//resize新的widget
	widget(next)->setGeometry ( 0,  0, offsetx, offsety );
   
//根据移动的方向模式，计算出偏移量
	if (direction == BottomToTop)  {
		offsetx=0;
		offsety=-offsety;
	}
	else if (direction == TopToBottom) {
		offsetx=0;
		//offsety=offsety;
	}
	else if (direction == RightToLeft) {
		offsetx=-offsetx;
		offsety=0;
	}
	else if (direction == LeftToRight) {
		//offsetx=offsetx;
		offsety=0;
	}
	QPoint pnext=widget(next)->pos();
	QPoint pnow=widget(now)->pos();
	m_pnow=pnow;

//移动，显示新的widget
	widget(next)->move(pnext.x()-offsetx,pnext.y()-offsety);
	widget(next)->show();
	widget(next)->raise();

//初始化动画
	QPropertyAnimation *animnow = new QPropertyAnimation(widget(now), "pos");
	animnow->setDuration(m_speed);
	animnow->setEasingCurve(m_animationtype);
	animnow->setStartValue(QPoint(pnow.x(), pnow.y()));
	animnow->setEndValue(QPoint(offsetx+pnow.x(), offsety+pnow.y()));

	QPropertyAnimation *animnext = new QPropertyAnimation(widget(next), "pos");
	animnext->setDuration(m_speed);
	animnext->setEasingCurve(m_animationtype);
	animnext->setStartValue(QPoint(-offsetx+pnext.x(), offsety+pnext.y()));
	animnext->setEndValue(QPoint(pnext.x(), pnext.y()));

	QParallelAnimationGroup *animgroup = new QParallelAnimationGroup;

	animgroup->addAnimation(animnow);
	animgroup->addAnimation(animnext);

	QObject::connect(animgroup, SIGNAL(finished()),this,SLOT(animationDoneSlot()));
	m_next=next;
	m_now=now;
	m_active=true;
	animgroup->start();
}
void SlidingStackedWidget::animationDoneSlot(void) {
    setCurrentIndex(m_next);
//移动隐藏之前的widget
	widget(m_now)->hide();
	widget(m_now)->move(m_pnow);
	m_active=false;
	emit animationFinished();
}
```
以上就是滑动窗口界面的核心内容了。
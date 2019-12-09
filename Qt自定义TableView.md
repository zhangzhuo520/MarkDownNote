#Qt自定义之TabView
---------------------------------------
&emsp;&emsp;Qt的QTabView提供了Tab的操作非常方便，可以自定义，美化Tab。但是最近遇到这样一个需求，当TabView为空的时候需要设置固定大小的图片在Widget上。由于源码没有获得stackWidget的接口。所以自定义了一个TabView。

###QTabView的结构
&emsp;&emsp;QtabView由一个QTabBar和一个QStackWidget组成，如下
   ![](https://i.imgur.com/wUAeahI.png)
因此我们创建一个QWidget，添加一个QVboxLayout，上面添加一个QTabBar，下面添加一个QStackWidget即可。然后用相关的槽函数把tabbar和stackWidget关联起来。

####源码如下
TabBar.h
```
class ViewBar : public QTabBar
{
    Q_OBJECT
public:
    ViewBar(QWidget* parent = 0);
    ~ViewBar();

signals:

protected:

};
```
 TabBar.cc
```
ViewBar::ViewBar(QWidget *parent):
    QTabBar(parent)
{
    setExpanding(FALSE);
}

ViewBar::~ViewBar()
{
}
```
ViewStack.h
```
class ViewStack : public QStackedWidget
{
public:
    explicit ViewStack(QWidget *parent = 0);
    ~ViewStack();

protected:
    virtual void paintEvent(QPaintEvent *e)
    {
        QStackedWidget::paintEvent(e);
    }
};
```
ViewStack.cc
```
ViewStack::ViewStack(QWidget *parent):
    QStackedWidget(parent)
{
}

ViewStack::~ViewStack()
{
}
```
tabView.h
```
class TabView;
class ViewBar;
class ViewStack;

class TabViewPrivate : public QObject
{
    Q_DECLARE_PUBLIC(TabView)
    Q_OBJECT
public:
    explicit TabViewPrivate(TabView *patent = 0);
    ~TabViewPrivate();

    void init();
    TabView *q_ptr;
    ViewBar *m_view_bar;
    ViewStack *m_view_stack;
};

class TabView : public QWidget
{
    Q_DECLARE_PRIVATE(TabView)
    Q_OBJECT
public:
    explicit TabView(QWidget *parent = 0);
    ~TabView();
    void addTab(QWidget*, QString);
    void addTab(QWidget*, QIcon, QString);
    void setTabsClosable(bool);
    QWidget* currentWidget();
    void removeTab(int);
    int currentIndex();
    int count();
    void setCurrentIndex(int);
    void setTabToolTip(int, QString);
    void setTabText(int, QString);
    void clear();
    QTabBar *tab_bar();

signals:
    void tabCloseRequested(int);
    void currentChanged(int);

private slots:
    void slot_tab_clost(int);
    void slot_tab_change(int);

protected:
    void paintEvent(QPaintEvent *);

private:
    TabViewPrivate *d_ptr;
};
```

TabView.cc

```
#include "tabview.h"
#include "viewbar.h"
#include "viewstack.h"
#include "tabview_p.h"
#include <QVBoxLayout>
#include <QTabBar>
#include <QDebug>
#include <QPalette>

TabViewPrivate::TabViewPrivate(TabView *parent):
    q_ptr(parent)
{

}

TabViewPrivate::~TabViewPrivate()
{
}

void TabViewPrivate::init()
{
    Q_Q(TabView);
    m_view_bar = new ViewBar(q);
    m_view_stack = new ViewStack(q);
    QVBoxLayout * vlayout = new QVBoxLayout;
    vlayout->setMargin(0);
    vlayout->setSpacing(0);
    vlayout->addWidget(m_view_bar);
    vlayout->addWidget(m_view_stack);
    q->setLayout(vlayout);

    connect(m_view_bar, SIGNAL(currentChanged(int)), m_view_stack, SLOT(setCurrentIndex(int)));
    connect(m_view_bar, SIGNAL(currentChanged(int)), q, SLOT(slot_tab_change(int)));
    connect(m_view_bar, SIGNAL(tabCloseRequested(int)), q, SLOT(slot_tab_clost(int)));
}


TabView::TabView(QWidget *parent):
    QWidget(parent),
    d_ptr(new TabViewPrivate(this))
{
    Q_D(TabView);
    d->init();
}

TabView::~TabView()
{
}

void TabView::addTab(QWidget*widget, QString tab_name)
{
    Q_D(TabView);
    if(widget)
    {
        d->m_view_stack->addWidget(widget);
        d->m_view_bar->addTab(tab_name);
    }
}

void TabView::addTab(QWidget *widget, QIcon icon, QString tab_name)
{
    Q_D(TabView);
    if(widget)
    {
         //must add stack first, then add tab, beacuse addtab will be emit signal currentTabChange, and change stack, however, stack is empty, so ,core dump
        d->m_view_stack->addWidget(widget);
        d->m_view_bar->addTab(icon, tab_name);
    }
}

void TabView::setTabsClosable(bool enabel)
{
    Q_D(TabView);
    d->m_view_bar->setTabsClosable(enabel);
}

QWidget *TabView::currentWidget()
{
    Q_D(TabView);
    return d->m_view_stack->currentWidget();
}

void TabView::removeTab(int index)
{
    Q_D(TabView);
    if(QWidget *w = d->m_view_stack->widget(index))
    {
        d->m_view_stack->removeWidget(w);
        d->m_view_bar->removeTab(index);
    }
}

int TabView::currentIndex()
{
    Q_D(TabView);
    return d->m_view_bar->currentIndex();
}

int TabView::count()
{
    Q_D(TabView);
    return d->m_view_bar->count();
}

void TabView::setCurrentIndex(int index)
{
    Q_D(TabView);
    if(index < 0 || index > d->m_view_bar->count())
    {
        qDebug() << "index out of range! ";
    }

    d->m_view_bar->setCurrentIndex(index);
    d->m_view_stack->setCurrentIndex(index);
}

void TabView::setTabToolTip(int index, QString tip)
{
    Q_D(TabView);
    if(index < 0 || index > d->m_view_bar->count())
    {
        qDebug() << "index out of range! ";
    }
    d->m_view_bar->setTabToolTip(index, tip);
}

void TabView::setTabText(int index, QString filename)
{
    Q_D(TabView);
    if(index < 0 || index > d->m_view_bar->count())
    {
        qDebug() << "index out of range! ";
    }
    d->m_view_bar->setTabText(index, filename);
}

void TabView::clear()
{
    Q_D(TabView);
    while(d->m_view_bar->count())
    {
        removeTab(0);
    }
}

QTabBar *TabView::tab_bar()
{
    Q_D(TabView);
    return dynamic_cast<QTabBar *> (d->m_view_bar);
}

void TabView::paintEvent(QPaintEvent *)
{
}

void TabView::slot_tab_clost(int index)
{
    emit tabCloseRequested(index);
}

void TabView::slot_tab_change(int index)
{
    emit currentChanged(index);
}

```
以上就是自定义的TabView。总之一句话，原生不行，上去就是自己造。
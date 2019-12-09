# 关于QT的自定义布局
  自定义的布局形状如下

![](https://i.imgur.com/6OLPhkl.gif)
  ----------------------------------------
## 简介
&emsp;&emsp;以前觉得自定义布局很难，但是亲手写了一下发现也很简单。就是继承子类化Layout。实现几个虚函数，然后再setGeometry()这个函数中计算各个子item的位置从而实现布局效果就好了。

## 实现
&emsp;&emsp; 通过继承QLayout这个类来实现自己的layout，QT框架中自己实现了这几个布局类可以使用：QBoxLayout, QGridLayout, QFormLayout, QStackedLayout。以及BorderLayout和flowLayout（文档详细说明）

&emsp;&emsp;如果我们子类化QLayout必须实现 addItem(), sizeHint(), itemAt(), setGeometry(), takeAt()这几个函数。还应该实现mininumSize（）这个函数，来确保布局的最小大小不为0。确定布局的函数主要是setGeometry()，通过计算每个子Item的位置来做成你想要的样子。

## 代码如下：
border.h

```
#include <QLayout>
#include <QList>
#include <QRect>

class Flayout : public QLayout
{
public:
    enum direction{
        TOP = 0,
        BOTTOM,
        LEFT,
        RIGHT,
        CENTER
    };

    typedef struct fitem{
        fitem(QLayoutItem *item, direction direct):
            layout_item(item),
            dir(direct){}
        QLayoutItem *layout_item;
        direction dir;
    }Fitem;

    explicit Flayout(QWidget *parent = 0);
    ~Flayout();

    void addWidget(QWidget *, direction);
    void add(QLayoutItem *, direction);

protected:
    virtual void addItem(QLayoutItem *);
    void setGeometry(const QRect &);
    QSize sizeHint() const;
    QLayoutItem * itemAt(int) const;
    QLayoutItem *takeAt(int index);
    int count() const;
    QSize minimumSize() const;

private:
    QList<Fitem *>m_item_list;
};
```
border.cc
```
Flayout::Flayout(QWidget *parent):
    QLayout(parent)
{
    setSpacing(0);
}

Flayout::~Flayout()
{
    QLayoutItem *l;
    while ((l = takeAt(0)))
        delete l;
}

void Flayout::addItem(QLayoutItem * item)
{
    add(item, TOP);
}

void Flayout::setGeometry(const QRect & rect)
{
    int left_height = 0;
    int left_width = 0;
    QLayout::setGeometry(rect);

    for (int i = 0; i < m_item_list.size(); ++i)
    {
        Fitem *wrapper = m_item_list.at(i);
        QLayoutItem *item = wrapper->layout_item;
        direction direct = wrapper->dir;

        if (direct == LEFT)
        {
            item->setGeometry(QRect(rect.x(), rect.y(), rect.width() / 4, rect.height() / 4 * 3));

            left_height += item->geometry().height();
            left_width  += item->geometry().width();
        }
    }

    for (int i = 0; i < m_item_list.size(); ++i)
    {
        Fitem *wrapper = m_item_list.at(i);
        QLayoutItem *item = wrapper->layout_item;
        direction position = wrapper->dir;

        if (position == TOP)
        {
            item->setGeometry(QRect(rect.x() + left_width, rect.y(),
                                    rect.width() / 4 * 3, rect.height() / 4));
        }
        else if (position == BOTTOM)
        {
            item->setGeometry(QRect(rect.x(), left_height + rect.y(),
                                    rect.width() / 4 * 3, rect.height() / 4));
        }
        else if(position == RIGHT)
        {
            item->setGeometry(QRect(rect.x() + rect.width() -  left_width ,  rect.y() + rect.height() - left_height,
                                   rect.width() / 4, rect.height() / 4 * 3));
        }
        else if(position == CENTER)
        {
            item->setGeometry(QRect(rect.x() + rect.width() / 4,  rect.y() + rect.height() / 4,
                                   rect.width() / 2, rect.height() / 2));
        }
    }
}

QSize Flayout::sizeHint() const
{
    return QSize(300, 300);
}

QLayoutItem *Flayout::itemAt(int index) const
{
    Fitem *item = m_item_list.value(index);
    if (item)
    {
        return item->layout_item;
    }
    else
    {
        return 0;
    }
}

QLayoutItem *Flayout::takeAt(int index)
{
    if (index >= 0 && index < m_item_list.size()) {
        Fitem *layoutStruct = m_item_list.takeAt(index);
        return layoutStruct->layout_item;
    }
    return 0;
}

int Flayout::count() const
{
    return m_item_list.count();
}

QSize Flayout::minimumSize() const
{
    return QSize(30,30);
}

void Flayout::addWidget(QWidget * widget, direction dir)
{
    add(new QWidgetItem(widget), dir);
}

void Flayout::add(QLayoutItem * item, direction dir)
{
    m_item_list.append(new Fitem(item, dir));
}

```
使用：
widget.cc：
```
    Flayout *pLayout = new Flayout(this);
    pLayout->addWidget(createWidget("Black"), Flayout::CENTER);
    pLayout->addWidget(createWidget("White"), Flayout::TOP);
    pLayout->addWidget(createWidget("Red"), Flayout::BOTTOM);
    pLayout->addWidget(createWidget("Green"), Flayout::LEFT);
    pLayout->addWidget(createWidget("Yellow") , Flayout::RIGHT);
    setLayout(pLayout);
```

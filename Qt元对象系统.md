#一、元对象
---
元对象（meta object）意思是描述另一个对象结构的对象，比如获得一个对象有多少成员函数，有哪些属性。在Qt中，我们将要用到的是QMetaObject这个类。

元对象系统基于以下3点：

###以QObject作为基类
类声明的私有区域中，Q_Object宏指令使我们能够使用元对象的特性，比如动态属性、信号、槽等
元对象编译器（Meta-Object Compiler  moc）为QObject子类生成具有元对象特性的代码
我们可以通过QObject类的一个成员函数获得该类的元对象：

 QMetaObject *QObject::metaObject() const  

通过这个元对象，进而可以获取一个QObject对象的更多信息：

 QMetaObject::className() 返回运行时类的名称（不需要C++中的运行时类型识别机制RTTI）

 QMetaObject::methodCount() 返回类中方法的个数

以上只是元对象的简单介绍，记住元对象系统的3点特性。之所以要介绍元对象，因为Qt中很多用法是基于元对象的，如果不支持元对象，比如没有继承自QObject，那么很多东西将无法使用，下面对此作进一步介绍。

 

#二、类型识别
众所周知，C++中使用dynamic_cast和typeid这两个运算符进行运行时类型识别（RTII），但是Qt提供另外两种运行时类型识别方法：

 qobject_cast 和 QObject::inherits() 

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
据说，qobject_cast的速度比dynamic_cast的速度快很多。

QObject::inherits(const char *className)的速度相对慢一些，所以尽可能使用qobject_cast。

 

#三、Qt中的属性
1.自定义属性
我们可能已经接触到很多Qt中的属性了，比如qreal类型的opacity属性表示“透明度”，QRect类型的geometry表示“几何位置和大小”，QPoint类型的pos属性代表“位置”。

所谓属性，也就是类中的一个数据成员，我们可以获取（get）和设置（set）。

除了Qt中一些类已经具备的属性，我们还可以自定义属性，也就是定义一种访问数据成员的方式。

在一个继承自QObject的类中使用 Q_PROPERTY 宏指令，比如：
 ```
Q_PROPERTY(bool focus READ hasFocus)
Q_PROPERTY(bool enabled READ isEnabled WRITE setEnabled)
Q_PROPERTY(QColor color MEMBER m_color NOTIFY colorChanged)
 ```
不要被这个用法搞晕了，其实很简单，刚开始指定属性类型和名称，然后READ表示获取属性值的方法，一般这两点是必须的。其他都是可选的，比如WRITE表示设置属性值得方法，MEMBER表示这个属性在类中数据成员的名称，NOTIFY表示属性改变发出的信号。

2.属性的类型
由上可知，属性的类型可以是bool、QString、QRect等等，我们可以通过 QVariant::Type 的枚举值获得所有可用于属性的类型。

可以查到，它不支持枚举类型，但可以通过 Q_ENUM 来设置：
 ```
enum Priority { High, Low, VeryHigh, VeryLow };
Q_ENUM(Priority)
 
Q_PROPERTY(Priority priority READ priority WRITE setPriority NOTIFY priorityChanged)
 ```
当然，自定义的类型也是不支持的，需要通过 Q_DECLARE_METATYPE 注册元类型：
 ```
struct MyStruct
{
    int i;
    ...
};
Q_DECLARE_METATYPE(MyStruct)
 ```
3.属性的读与写
我们可以直接使用get和set方法来读写属性，也可以通过QObject与QMetaObject来间接地读写属性。

首先是设置属性值

比如类QAbstractButton有一个“down”的属性，表示按钮是否被按下，它有一个成员函数 QAbstractButton::setDown() 来改变属性值，同时，我们也可以通过 QObject::setProperty() 对其进行设置：
 ```
QPushButton *button = new QPushButton;
QObject *object = button;
 
button->setDown(true);
object->setProperty("down", true);
 ```
值得注意的是，setProperty()这个函数不但可以改变属性值，也可以在运行时动态地为对象添加属性。

接下来是读取属性值

如果有get函数，可以直接调用它，当然也可以通过 QObject::property() 来获取属性，它的返回值是 QVariant 类型的，通过 canConvert() 进行判断，然后将其转换为所需的类型。
 ```
QObject *object = ...
const QMetaObject *metaobject = object->metaObject();
int count = metaobject->propertyCount();
for (int i=0; i<count; ++i) {
    QMetaProperty metaproperty = metaobject->property(i);
    const char *name = metaproperty.name();
    QVariant value = object->property(name);
    ...
}
  ```

#四、自定义属性有什么用
也许你要问，说了这么多废话，不就是读写成员变量吗？好吧，平时确实不会用太多，但是如果做插件开发、qml等，就会经常用到了。不过，在通常的界面开发中，属性也是有用处的，下面举两个例子。

1.改变样式
样式表设置中有一个属性选择器，比如 QPushButton[flat="false"] 意思是当按钮属性flat为false时的样式。

举个栗子，我们有个QWidget类，名字叫PropertyTest，界面中有一个按钮叫pushButton
 ```
#pushButton{border:4px solid blue;}
PropertyTest[borderColor="red"] #pushButton{border:4px solid red;}
PropertyTest[borderColor="green"] #pushButton{border:4px solid green;}
PropertyTest[borderColor="blue"] #pushButton{border:4px solid blue;}
```
按钮默认样式是blue蓝色，通过改变类PropertyTest的属性borderColor值改变按钮的颜色。

在代码中，首先定义属性

Q_PROPERTY(QString borderColor READ getBorderColor WRITE setBorderColor)
使用一个成员变量保存属性的值，并通过set和get函数分别设置和获得该值。
 ```
private:
    QString m_strBorderColor;
private:
    void setBorderColor(const QString &strBorderColor){ m_strBorderColor = strBorderColor; }
    QString getBorderColor(){ return m_strBorderColor; }
 ```
单击按钮pushButton改变属性值，从而改变按钮pushButton的样式。
 ```
void PropertyTest::changeBorderColor()
{
    if (m_iTest % 3 == 0)
    {
        setBorderColor("red");
    }
    else if (m_iTest % 3 == 1)
    {
        setBorderColor("green");
    }
    else
    {
        setBorderColor("blue");
    }
    style()->unpolish(ui.pushButton_3);
    style()->polish(ui.pushButton_3);
    update();
    m_iTest++;
}
 ```
最后要注意的是，上面代码中的unpolish和polish部分。

在Qt文档中有个提醒，在使用属性选择器时，如果之前控件有其它样式，那么需要重写设置一下，“旧的不去，新的不来”，通过unpolish和polish抹去旧的样式，涂上新的样式。
2.动画中使用自定义属性
如果我们想通过动画使一个按钮逐渐变透明，思路会是这样：按钮QPushButton继承自QWidget，在QWidget中有个函数setWindowOpacity，所以只需使用动画类QPropertyAnimation，属性那个参数设置为windowOpacity。

然而，实际中，按钮透明度不会有任何改变，继续查看文档才知道——只有调用setWindowFlags函数，将窗口属性设置为Qt::Window，windowOpacity这个属性才能生效。但是这样做pushbutton就不是正常的widget了。

因此，有必要寻求其它方法，在QWidget中有一个函数setGraphicsEffect(QGraphicsEffect *)，其中QGraphicsEffect有一个派生类QGraphicsOpacityEffect，可以通过它来设置QWidget的透明度。
 ```
m_pOpacityEffect = new QGraphicsOpacityEffect(this);
m_pOpacityEffect->setOpacity(1);
this->setGraphicsEffect(m_pOpacityEffect);
Q_PROPERTY(qreal buttonOpacity READ buttonOpacity WRITE setBtnOpacity)
 ```
上面的写法可能不太好，因为qreal精度与机器有关，最好用double或float。

定义属性时，在函数setBtnOpacity中改变QGraphicsOpacityEffect对象，来调整透明度。

好了，现在我们将动画属性名称设置为buttonOpacity—— QPropertyAnimation::setPropertyName("buttonOpacity") ，就能通过动画改变按钮的透明度了。

 

五、invokeMethod()
Qt中的信号槽机制是以元对象为基础的，通过名称以类型安全的方式来间接调用槽函数。

当调用槽函数时，实际是由invokeMethod()完成的。

比如显示一个窗口，一般是通过show()函数来完成，不过我们还能这样做：

MyWidget w;
QMetaObject::invokeMethod(&w, "show");
上面讲了如何将成员变量注册进元对象系统，那么对于成员函数，该怎么做呢？

在声明一个类的成员函数时，通过使用 Q_INVOKABLE 宏进行注册，可以使它们能够被元对象系统调用。
 ```
class Window : public QWidget
{
    Q_OBJECT
public:
    Window();
    void normalMethod();
    Q_INVOKABLE void invokableMethod();
};
 ```
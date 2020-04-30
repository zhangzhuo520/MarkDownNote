#使用Q_DECLARE_METATYPE 和 qRegisterMetaType
-------------------------

Q_DECLARE_METATYPE 是为了让QVariant能识别到你定义的类型，方便转换

qRegisterMetaType 是为了信号与槽能传递你自定义的类型（主要多线程需要注册，单线程不注册也好像能传递）

事例如下：
  ```
Q_DECLARE_METATYPE：

Q_DECLARE_METATYPE(render::LayoutScene *)
typedef QMap<render::LayoutScene *, render::GeometryView> MapRender;
Q_DECLARE_METATYPE(MapRender)

qRegisterMetaType
qRegisterMetaType<render::GeometryView>("render::GeometryView");
qRegisterMetaType<render::LayoutScene *>("render::LayoutScene");

```


1. 使用Q_DECLARE_METATYPE最好放在命名空间外面，不然会出现下面错误
    ```
 error: specialization of ‘template<class T> struct QMetaTypeId’ in different namespace
    ```

2.使用qRegisterMetaType要在信号与槽连接前使用。须要再类下面先定义Q_DECLARE_METATYPE，再进行注册。
#QT自定义QWidget类的插件
---
&emsp;&emsp;之前使用了Qt的LowerAPI自定义的插件，发现只是实现某个模块的功能，并不能实现相关小控件的插件化。于是还有研究关于QT小控件的插件化。
###QDesigner的集成插件
&emsp;&emsp;1.利用QtCreator的插件框架，继承QDesignerCustomWidgetInterface，实现QwidgetInterface。把编译生成的库文件，头文件，源文件分别放到QT对应的Lib，include目录下面即可。

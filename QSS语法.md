# 关于QSS语法介绍
----------------------------
###&emsp;&emsp;QSS作为QT美化的主要手段在写软件中必不可少，虽然用了这么久，但是对QSS对其中的一些使用方式还是不熟悉，所以下面来总结写QSS的知识点。
##### &emsp; &emsp;QSS的语法如下：
语法类别 | 实例 | 解释
-|-|-
通配选择器 | * | 匹配所有的控件
类型选择器 | QPushButton | 匹配所有QPushButton和其子类的实例
属性选择器 |QPushButton[flat="false"] |匹配所有flat属性是false的QPushButton实例，注意该属性可以是自定义的属性，不一定非要是类本身具有的属性
类选择器 | .QPushButton | 匹配所有QPushButton的实例，但是并不匹配其子类。这是与CSS中的类选择器不一样的地方，注意前面有一个点号
ID选择器 | QPushButton#myButton | 匹配所有id为myButton的控件实例，这里的id实际上就是objectName指定的值
后代选择器 | QDialog QPushButton | 所有QDialog容器中包含的QPushButton，不管是直接的还是间接的
子选择器 | QDialog > QPushButton | 所有QDialog容器下面的QPushButton，其中要求QPushButton的直接父容器是QDialog


        


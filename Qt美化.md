#QT布局相关知识
---------
&emsp;&emsp;最近对整个项目做了全方位的美化，记录下美化相关的布局类的知识。
###Layout代码美化知识：
 1.Margin：Layout外面四周的空间（通常是11px）
 2.setContentsMargins Layout外面四周的空间
以上两个只设置一个就好，同时设置会有冲突
 3.setspacing:布局内空间相邻的空间。

- 关于QT设置不同的style导致tooltip中text不能显示的问题，通过sytlesheet统一重新绘制一遍即可
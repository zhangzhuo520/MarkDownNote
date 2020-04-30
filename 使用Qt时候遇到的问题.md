1.用插件添加Dockwidget时候，调用Dockwidget->raise()函数失效，Dockwidget不会active？
解决：如果不生效需要先设置setVisibel（true）;再设置raise（）；

2.使用自定义的Treeview model删除节点的时候crash， 原因是删除的节点的parent指针地址发送了变动，当时知道为什么会发生变动。很神奇
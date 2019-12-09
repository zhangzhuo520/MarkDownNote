#Centos7 编译QtCreator2.5.2
---------
1.到QtCreator2.5.2源码目录， mkdir Build
2.cd Bulid
3.make-qt4 ../qtcreator.pro
4.make -j90

遇到的问题:
```
make[3]: Leaving directory `/home/zhuozhang/work/qt-creator-2.5.2-src/build/src/plugins/cvs'
.moc/release-shared/moc_helpviewer.cpp:91:8: error: ‘QWebView’ has not been declared
     { &QWebView::staticMetaObject, qt_meta_stringdata_Help__Internal__HelpViewer,
```
直接把helpViewer.h中关于#if !defined(QT_NO_WEBKIT)的部分删除掉就好
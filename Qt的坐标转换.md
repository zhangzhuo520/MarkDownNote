#关于Qt不同控件坐标转换的问题
-----------------------------
Qt每个Widget都有自己的坐标系
mapToGlobal （自己的坐标转换为应用的全局坐标）
mapFromGlobal（应用的全局坐标转换为自己的坐标）

mapToParent（自己的坐标转换为父亲的坐标）
mapFromParent（父亲的坐标转换乐自己的坐标）

使用mapToGlobel转换的时候要注意层级关系。
例如widget的parent是frame
使用的时候
widget->mapToGlobal(mapToParent(pos))
要先转换成parent的坐标，在转行为全局坐标。


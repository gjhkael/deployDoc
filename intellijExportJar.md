# Intellij导出可执行jar包
intellij在继承maven和sbt等智能化管理项目方面确实做的很好，但是有时候我们需要将单个带有main方法的类单独导出成可执行jar的时候会出现一些问题。这里主要将一下如何在Intellij导出可执行jar并如何执行它。

###导出jar包
- 1 File->Project Structure ->Artifacts ->点击+号->jar->From moduls
- 2 在Main Class中将main函数加入进行，利用IDE扫描所有的main中选择
- 3 在Directory for META-INF/MANIFESF.MF中选择相应的文件（尤其是多模块项目，不能选错）
- 4 点击ok->然后在右边的选项中选择Build on make。
- 5 回到IDE，在菜单中点击Build->Make Project，然后到相应的项目中找到jar包

以上步骤在下图体现

![图](https://github.com/gjhkael/deployDoc/blob/master/image/intellij.png)

###执行jar包
如果直接使用java -jar xxx.jar会出现：Error: Invalid or corrupt jarfile错误
一种简单的方法就是使用：java -cp xxx.jar xxx.xxx.xxx(主函数路径)。


[AngularJS]:http://angularjs.org
[Gulp]:http://gulpjs.com

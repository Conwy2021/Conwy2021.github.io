# hugo写作备忘






#### 常用命令

在blog目录下启动cmd。

执行hugo命令 编译 。

>  使用hugo 后会自动将改动的文章或者配置，更新到public 目录下 ，实际上传的也是public目录下的文件



执行hugo server -D  本地启动  -D 是可以看到草稿文件的。

执行hugo server 是看不到草稿文件的。

启动后 http://localhost:1313/  本地访问博客。

草稿文件命令 在文章开头中draft: true

如图

![](/hugo使用备忘.assets/image-20221208225938771.png)

创建文章命令  hugo new posts/第二篇测试博客.md （默认是草稿模式）

#### 图片设置

正常做法： 

1. 写文章时，在该文章目录下新建文件夹存放照片，引用时采用相对路径。 

2. 完成之后，在引用路径前加个 /，比如原来引用方式 ``![](imgs/pic_name.png)`` ，需要修改为 ``![](/imgs/pic_name.png)`` 。

3.  之后将该图片文件夹移动到 static 目录下即可。 

   注意：如果该文件夹名包含空格可能会不能被显示，支持中文，但是不支持含空格。
   

#### 其他

文章目录为 ``\blog\content``

#### 更新至public目录

cmd中 使用 hugo 命令



 









---
layout:     post
title:      Git上传到Github仓库相关操作命令与步骤
subtitle:   通过Git上传代码到Github仓库
date:       2019-08-21
author:     JC
header-img: img/github.jpg
catalog: false
tags:
    - Git
    - Github
---
### 第一步：cd进入你放项目文件的地址

### 第二步：输入git init

是在当前项目的目录中生成本地的git管理（会发现在当前目录下多了一个.git文件夹）

### 第三步：输入git add .     

别忘了最后的点

这个是将项目上所有的文件添加到仓库中的意思，如果想添加某个特定的文件，只需把.换成这个特定的文件名即可。

### 第四步输入git commit -m "first commit"，

表示你对这次提交的注释，双引号里面的内容可以根据个人的需要改。

### 第五步输入git remote add origin https://自己的仓库url地址

（上面有说到） 将本地的仓库关联到github上，

### 最后一步，输入git push -u origin master

这是把代码上传到github仓库的意思。

执行完后，如果没有异常，会等待几秒，然后跳出一个让你输入Username和Password 的窗口，你只要输人github的登录账号和密码就行了。

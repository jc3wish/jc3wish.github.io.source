---
title: "第一篇博客"
date: 2019-10-16T22:24:26+08:00
draft: false
tags: ["杂谈"]
categories: ["杂谈"]
---

折腾了 hexo 不少时间,一直没真正用起来, 重点是那 nodejs ,npm  还有各种 依懒包的可能不兼容,太折腾人了

最终换成了 hugo 

我也是一个委喜欢 golang 的人,还是支持下  hugo 吧

hugo 直接下载二进制包就行了,真的太方便了


#### hugo 使用

1. 新增一个博客 

```
hugo new site my_blog

```


2. 进入到博客文件夹

```
cd my_blog
 
```

3. 下载主题

```
git submodule add https://github.com/olOwOlo/hugo-theme-even themes/even

```

even 主题的话,需要将 themes/even/exampleSite/config.toml 博客根目录

```
cp themes/even/exampleSite/config.toml ./

```

4. 创建博客

```
hugo new post/my-first-blog.md

```

博文头部信息

```

---
title: "第一篇博客"
date: 2019-10-16T22:24:26+08:00
draft: true
tags:[]
categories: ["杂谈"]
---

```

tags 是标签

categories 是分类

draft true 是草稿意思,在编译发布的时候,这个应该去掉

5. 运行

```
hugo server -D

```

运行之后  就可以通过  http://localhost:1313/ 进行访问了


6. 编译

```
hugo --baseUrl="https://jc3wish.github.io/"

```

baseUrl 是你的 github.io 打开的二级域名哦

（注意，以上命令并不会生成草稿页面，如果未生成任何文章，请去掉文章头部的 draft=true 再重新生成。）


7. 发布

```

cd public
git init
git remote add origin https://github.com/jc3wish/jc3wish.github.io.git
git add -A
git commit -m "first commit"
git push -u origin master


```


pubilc目录里所有文件 push 到刚创建的Repository的 master 分支。

当然也可以是其他分支下,如果是其他分支 下,需要到 github 仓库 设置下 进行设置,github.io 访问进来的时候,将读取哪个分支的静态文件




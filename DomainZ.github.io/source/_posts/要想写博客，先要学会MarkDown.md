---
title: 要想写博客，先要学会MarkDown
date: 2018-01-24 17:38:42
tags: MarkDown
---
一般写文字先开始一个描述，就是说说这篇文章是将什么的？然后开始几个大标题，大标题里面几个小标题。文章前情说完了。

## 这里要写一个大标题

### 大标题下面有几个小标题

0. 如果要写1234的列表;
1. 如果说明的文字里面有链接[Server](https://baidu.com)
2. 如果要突出文字，**加粗**、*斜体*，开始派上用场了;
3. 如果引用了一波；
 > 这是引用

### 小标题一
如果不幸是码农:

```
UIView *shareBoardView = [[[NSBundle mainBundle] loadNibNamed:@"QBDLShareBoardView" owner:self options:nil] lastObject];

```

### 小标题二
如果要放一张图片
{% asset_img avatar.jpg This is an example image %}

---
### MD编辑好了，然后用hexo相关命令发布上去

[hexo官方文档](https://hexo.io/zh-cn/docs/)

0. hexo new post "blog title"
1. hexo g	//生成
2. hexo s	//本地服务器运行，通过localhost访问
3. hexo d	//部署



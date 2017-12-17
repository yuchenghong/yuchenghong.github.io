---
title: Github上博客搭建
date: 2017-12-17 09:15:35
update: 2017-12-17 09:15:35
categories: github
tags: [github]
---
GitHub上的博客终于配置好了, 简单记录一下。这篇文章假设读者已经有一定的github的基础了。省去了很多基础步骤，如果有不清楚的可以给我邮件: chenghongyu@aliyun.com  
  
# 1, 安装node.js  
https://nodejs.org/en/  
# 2, 安装Git  
# 3, 安装hexo  
执行命令:

```
npm install -g hexo-cli 
```
 
# 4, 创建博客  
选择一个本地的文件夹，如/Users/yuchenghong/github-blog  
  执行命令:
  
```
hexo init  
npm install
```
# 5, 查看博客  
如果成功, 能在目录下看到文件

```
├── _config.yml // 网站的配置信息，你可以在此配置大部分的参数。
├── package.json 
├── scaffolds // 模板文件夹。当你新建文章时，Hexo会根据scaffold来建立文件。
├── source // 存放用户资源的地方
|   ├── _drafts
|   └── _posts
└── themes // 存放网站的主题。Hexo会根据主题来生成静态页面。
```

# 6, 启动   

```
hexo s
```

成功的话，访问:http://localhost:4000/， 就能看到页面了

# 7, 选择next主题
默认的主题不好看，因此选择next主题。  
先clone next主题
git clone https://github.com/iissnan/hexo-theme-next themes/next  

# 8，站点配置文件
根目录_config.yml。 

```
# Site
title: Blog  # 首页title名称
subtitle: Owen's Blog  
description: Owen #作者简介
author: Owen Yu # 作者名称
language: zh-Hans # 修改语言环境为简体汉字
timezone:

# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
# theme: landscape
# 设置主题为next
theme: next
```
改完后执行命令看效果：

```
hexo g
hexo s
```


# 9，主题配置文件
next目录_config.yml

```
menu:
  home: /
  categories: /categories # 分类
  archives: /archives     # 归档
  tags: /tags             # 标签
  message: /message       # 留言
  about: /about           # 关于
  # commonweal: /404.html # 公益
```

# 10，生成ategories 和 tags的页面
hexo n page "categories"
hexo n page "tags"
会生成source\categories 和 source\tags 两个文件夹，里面都有index.md文件.修改内容为

```
---
title: categories
type: categories
---

---
title: tags
type: tags
---
```
这个文件改完后直接刷新页面就能看到效果，无需重新发布。  


到这个本地的配置就做完了，下面开始介绍发布到github

---

# 11，生成ssh key  
命令如下，email那儿写github的注册邮箱
```
ssh-keygen -t rsa -C "email@aliyun.com" 
```

/Users/yuchenghong/.ssh目录下会生成两个文件: id_rsa		id_rsa.pub  
复制id_rsa.pub文件的内容  
打开https://github.com/settings/ssh new SSH key 添加密钥，title随便写，key为刚刚复制的内容

验证：

```
ssh -T git@github.com
```
提示下面的内容说明成功：  
The authenticity of host 'github.com (192.30.255.113)' can't be established.
RSA key fingerprint is XXXXX  
Are you sure you want to continue connecting (yes/no)? yes  
Warning: Permanently added 'github.com,192.30.255.113' (RSA) to the list of known hosts.  
Enter passphrase for key '/Users/yuchenghong/.ssh/id_rsa':  
Hi yuchenghong! You've successfully authenticated, but GitHub does not provide shell access.

# 12，发布到github

```
git g  #一定要先执行
git d
```


# 13，生成文章：

```
hexo n "name"    # name 文章名称
```

在对应的name.md中(source/_post目录下)可以这样添加标签和分类

```
---
title: Github上博客搭建
date: 2017-11-03 15:39:31
update: 2017-11-03 15:39:31
categories: github             # 分类
tags: [github]   # [标签1, 标签2..., 标签n]
---
```

```
hexo g #编译
hexo s #发布到本地测试
hexo d #发布到github
```




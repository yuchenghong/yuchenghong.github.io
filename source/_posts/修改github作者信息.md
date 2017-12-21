---
title: 修改github作者信息
date: 2017-12-21 21:03:35
update: 2017-12-21 21:03:35
categories: github
tags: [github]
---

最近重装了git, 不知道怎么丢失了配置信息，导致提交到github的作者信息不对。 

强迫症犯者表示不能忍受。  
还好在github官网上找到了解决办法: https://help.github.com/articles/changing-author-info/  
  

首先在本地执行git config把name和email改成正确的值。  

然后按下面的步骤操作：  
1，打开终端: 
```
git clone --bare https://github.com/yuchenghong/yuchenghong.github.io.git
```
 
  
2，到clone的目录下面：
```
cd yuchenghong.github.io.git
```

  
3，下面的代码复制到终端: 回车

```
#!/bin/sh  
git filter-branch --env-filter '  
OLD_EMAIL=yuchenghong  
CORRECT_NAME=yuchenghong  
CORRECT_EMAIL=273095916@qq.com  
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]  
then  
export GIT_COMMITTER_NAME="$CORRECT_NAME"  
export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"  
fi  
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]  
then  
export GIT_AUTHOR_NAME="$CORRECT_NAME"  
export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"  
fi  
' --tag-name-filter cat -- --branches --tags
```


4，直接复制执行下面的命令：

```
git push --force --tags origin 'refs/heads/*'
```


5，删除yuchenghong.github.io.git

```
cd ..
rm -rf yuchenghong.github.io.git
```


6, 到本地git库下，git pull更新下来。 后面就正常了。  
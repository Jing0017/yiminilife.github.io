---
title: hexo博客备份
categories:
  - blog
tags:
  - hexo
  - 备份
abbrlink: 2dd71a2b
date: 2019-07-18 19:02:57
---

### 一、将本地hexo博客的仓库初始化为git项目

注意：检查一下theme文件夹下的主题。例如如果themes/next，此目录下若有.git文件夹，请删除这个.git文件夹。

```bash
//默认已经在项目根路径下
git init  //初始化本地仓库
```

### 二、配置.gitignore文件

新建.gitignore（有则忽略），在文件中输入以下内容

```bash
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```

### 三、将本地仓库和xxx.github.io仓库的url配对，推送本地仓库文件至远端的hexo分支

```bash
git add .
git commit -m "blog hexo"
git branch hexo  //新建hexo分支
git checkout hexo  //切换到hexo分支上
git remote add origin git@github.com:xxx/xxx.github.io.git  //xxx为github用户名
git push origin hexo  //push到Github项目的hexo分支上
```



### 四、在其他终端获取hexo仓库。

#### 1.从github上拉取hexo分支的代码

```bash
git clone -b hexo git@github.com:user/user.github.io.git  //将Github中hexo分支clone到本地
cd user.github.io
npm install
```

#### 2.写文章并进行备份和部署

```bash
//进入user.github.io文件夹,应是hexo分支
git pull origin hexo //本地和远端的融合
hexo new post "new post name"  //写新文章
git add source
git commit -m "xxx"
git push origin hexo  //备份
hexo g && hexo d
```




---
layout: post
title: Git 常用操作以及一些命令速查
summary: Git 常用操作以及一些命令速查，repository，remote，workspace，commit，branch...
date: 2018-08-20 11:20:36
categories: CVS
tags: [Git]
featured-img: wool
---

![git](https://wx1.sinaimg.cn/mw690/005X6W83gy1fuh1nk7sx9j30m806bwfe.jpg)

### 基本概念
* repository
* remote
* workspace
* commit
* branch

-------

### 本地仓库
**Repository**

1. 本地初始化

    ```bash
    // 本地初始化一个git仓库，客户端仓库
    # git init
    ```
    
    ```bash
    // 创建一个裸库，供多人分享使用，常用于服务端，非本地
    # git init --bare
    ```
2. 从远端拉取仓库

    ```bash
    # git clone <版本库的网址>
    // 克隆远程仓库到当前文件夹
    # git clone https://git.coding.net/hymane/git-demo.git
    // 克隆远程仓库并指定文件夹名
    # git clone <版本库的网址> <dir>
    # git clone https://git.coding.net/hymane/git-demo.git myGitDir
    ```
    可以使用 `-o` 参数指定远程主机名
    
    ```bash
    # git clone -o remoteName <版本库的网址>
    ```
    
-------

### 远程仓库
**Remote**

为了便于管理，Git要求每个远程主机都必须指定一个主机名。

git remote命令就用于管理主机名。

* 不带选项的时候，git remote命令列出所有远程主机。

```bash
# git remote
origin
```

* 可以使用 `-v` 参数查看远程主机地址

```bash
# git remote -v
origin  https://git.coding.net/hymane/git-demo.git (fetch)
origin  https://git.coding.net/hymane/git-demo.git (push)
```
* `git remote show` 查看远程主机信息

```bash
# git remote show <主机名>
* remote origin
  Fetch URL: https://git.coding.net/hymane/git-demo.git
  Push  URL: https://git.coding.net/hymane/git-demo.git
  HEAD branch: master
  ......
```

* `git remote add` 添加远程主机，绑定到本地仓库

```bash
# git remote add <主机名> <网址>
```

* `git remote rm` 删除或解绑远程主机

```bash
# git remote rm <主机名>
```

* `git remote rename` 重命名远程主机名

```bash
# git remote rename <原主机名> <新主机名>
```

-------

### 分支（branch）
* `git branch` 查看分支

```bash
// 查看本地分支
# git branch
* master
  dev
```
`*` 表示当前分支为 master 分支
本地还有一个 dev 的分支

* `-r` 参数查看远程分支

```bash
// 查看远程分支
# git branch -r
```
* `-a` 参数查看所有分子

```bash
// 查看所有分支
# git branch -a
  dev
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/dev
  remotes/origin/master
```

* `git checkout` 切换分支

```bash
# git checkout dev
Switched to branch 'dev'
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)
```

* `-b` 参数创建分支

```bash
# git checkout -b feature <baseBranch>
Switched to a new branch 'feature'
```
使用 `-b` 参数可以切换到指定分支，若分支不存在则创建对应分支，指定 baseBranch 可以基于一个基准分支进行创建，如果不指定则以当前分支进行创建新分支。

* `git branch -d` 删除分支

```
# git branch -d feature
Deleted branch feature (was d4887a4).
```
ps: 删除分支时当前分支不可以是待删除分支
可以先切换到其他分支

```bash
# git checkout dev
# git branch -d feature
```

* 在某些场合，Git会自动在本地分支与远程分支之间，建立一种追踪关系（tracking）。比如，在git clone的时候，所有本地分支默认与远程主机的同名分支，建立追踪关系，也就是说，本地的master分支自动"追踪"origin/master分支。也可以手动指定追踪关系

```bash
# git branch --set-upstream master origin/master
```

-------

### 远程和本地仓库互联

 1. 远程仓库更改取回本地（fetch）

    * 远程仓库有了更新（一般称为commit）就可以使用 `fetch` 命令拉取下来
    
    ```bash
    // 将远程主机所有更新全部拉回本地
    # git fetch <远程主机名>
    ```
    * 默认 `git fetch` 会拉取所有分支（branch）的更新到本地，如果只想拉取特定分支的更新可以指定分支名
    
    ```bash
    # git fetch <远程主机名> <分支名>
    ```
    
    * 比如拉取远程 origin 主机的 master 分支到本地
    
    ```bash
    # git fetch origin master
    ```
 2. 远程仓库更新直接取回到工作区
  `git pull` 命令较为复杂，他首先是将远程某个分支更新拉回本地，再与本地指定分支合并。
  * 完成命令

  ```bash
  # git pull <远程主机名> <远程分支名>:<本地分支名>
  ```
  比如将远程主机 origin 上 dev 分支拉回本地并合并到本地 feature 分支
  
  ```bash
  // 当前分支不一定是 feature 分支
  # git pull origin dev:feature
  ```
  
  * 如果远程分支是准备与当前分支合并，则可以不指定本地分支名

  ```bash
  # git pull origin dev
  ```
  上面命令含义是将远程分支 origin/dev 分支拉取下来和本地当前分支合并，即先`git fetch`到本地仓库，然后`git merge`合并。
  
  ```bash
  # git fetch origin
  # git merge origin/dev
  ```
  
  * 本地分支和远程存在追踪关系时可以简写命令

  ```bash
  # git pull origin
  ```
  
  * 如果当前只有一个追踪的分支，主机名也可以省略

  ```bash
  # git pull
  ```
  
  * `git pull` 默认用的是merge合并分支，可以指定为’rebase’模式合并 

  ```bash
  # git pull --rebase <远程主机名> <远程分支名>:<本地分支名>
  ```
  
  * 默认远程主机删除了某个分支，`git pull`不会删除本地，防止误删本地已修改的文件,可以使用`-p`指定删除操作

  ```bash
  // 危险
  # git pull -p
  ```
 3. 本地更新推送到远程（push）
将本地的更新推送到远程主机仓库，供团队查看
    
    * `git push` 推送本地更新

    ```bash
    # push <远程主机名> <本地分支名>:<远程分支名>
    ```
    ps: 分支推送顺序的写法是<来源地>:<目的地>,所以`git pull`是<远程分支>:<本地分支>，而`git push`是<本地分支>:<远程分支>。

    * 如果省略远程分支名，表示推送到与之存在"追踪关系"的远程分支，一般和本地分支同名。

    ```bash
    # git push origin master
    ```
    上面的命令意思是将本地 master 分支推送到远程主机 origin 上的master 分支上。
    
    * 如果省略本地分支名，但是需要显示加上`:`符号，表示删除远程分支，也可以认为是将本地`空`分支推送到远程指定分支，即删除分支。

    ```bash
    # git push origin :master
    // 等同于
    # git push origin --delete master
    ```
    上面命令表示删除远程 origin 主机上的 master 分支
    
    * 如果当前分支与远程分支之间存在追踪关系，则本地分支和远程分支都可以省略。

    ```bash
    # git push origin
    ```
    上面命令表示将本地分支推送到远程主机 origin 对应分支上，一般与本地分支同名
    
    * 如果当前分支只有一个追踪分支，那么主机名都可以省略。

    ```bash
    # git push
    ```
    
    * 如果当前分支和多个主机存在追踪关系，还想使用上面简洁的命令推送，可以使用`-u`添加默认主机

    ```bash
    # git push -u origin master
    // 之后就可以使用简洁命令推送
    # git push
    ```
    上面命令将本地分支推送到远程 master 分支，并且指定了默认主机为 origin，后面就可以不加任何参数使用git push了。
    
    * 以上命令默认都是推送当前分支，如果想一次推送本地所有分支到远程主机该怎么办？

    ```bash
    // 推送所有分支到远程origin主机
    # git push --all origin
    ```
    
    * `--force` 强制推送所有分支到远程主机
    
    ```bash
    // 强制推送本地分支到远程主机（你确定要这样做？）
    # git push --force origin 
    ```
    
    * 推送 tags 到远程主机

    ```bash
    # git push origin --tags
    ```
    
    
### 工作区和本地仓库
平时操作最多的就是工作区了，也就是我们的代码，每一次编码就是一次工作区的修改。其次就是本地仓库，将工作区代码提交到本地仓库作为一次提交，类似做了一次备份。

1. 追踪相关
    * 添加追踪，记录自己关心的文件，以便git检测对该文件的任意操作，并记录下来

    ```bash
    # git add <file1> <file2>
    # git add <dir>
    ```
    
    * 一次添加当前文件夹下的所有文件

    ```bash
    // 添加所有文件到追踪文件列表中
    //ps: ignore列出文件除外
    # git add .
    ```
    
    * 发现些 
    
2. commit提交

### 回退与撤销
1. git reset
2. git revert
3. git checkout
4. git rm
5. git stash


git rm --cached [file] == git reset [file]
    
### 保存登录账户    
git config --local credential.helper store




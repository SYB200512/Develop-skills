# Git命令

workspace:工作区

index/Stage:暂存区

Repository:本地仓库

Remote:远程仓库

## 一、新建代码库

1：在当前目录新建一个Git代码库

$    git init

2：新建一个目录，将其初始化为Git代码库

$ git init [project-name]

3：下载一个项目和它的整个代码历史

$ git clone [url]

## 二、增加/删除文件

1：添加指定文件到暂存区

$ git add [file1] [file2] ...

2：添加指定目录到暂存区，包括子目录

$ git add [dir]

3：添加当前目录的所有文件到暂存区

$ git add .

4：删除工作区文件，并且将这次删除放入暂存区

$ git rm [file1] [file2] ...

## 三、代码提交

1：提交暂存区到仓库区

$ git commit -m [message]

2：提交暂存区的指定文件到仓库区

$ git commit [file1] [file2] ... -m [message]

3：提交工作区自上次commit之后的变化，直接到仓库区

$ git commit -a

4：提交时显示所有diff信息

$ git commit -v

5：使用一次新的commit，替代上一次提交

如果代码没有任何新变化，则用来改写上一次commit的提交信息

$ git commit --amend -m [message]

6：重做上一次commit，并包括指定文件的新变化

$ git commit --amend [file1] [file2] ...

## 四、分支

1：列出所有本地分支

$ git branch

2：列出所有远程分支

$ git branch -r

3：列出所有本地分支和远程分支

$ git branch -a

4：新建一个分支，并切换到该分支

$ git checkout -b [branch]

5：新建一个分支，指向指定commit

$ git branch [branch] [commit]

6：切换到上一个分支

$ git checkout -

7：选择一个commit，合并进当前分支

$ git cherry-pick [commit]

8：删除分支

$ git branch -d [branch-name]

9：删除远程分支

$ git push origin --delete [branch-name]
$ git branch -dr [remote/branch]

## 五、远程同步

1：下载远程仓库的所有变动

$ git fetch [remote]

2：显示所有远程仓库

$ git remote -v

3：显示某个远程仓库的信息

$ git remote show [remote]

4：增加一个新的远程仓库，并命名

$ git remote add [shortname] [url]

5：取回远程仓库的变化，并与本地分支合并

$ git pull [remote] [branch]

6：上传本地指定分支到远程仓库

$ git push [remote] [branch]

7：强行推送当前分支到远程仓库，即使有冲突

$ git push [remote] --force

8：推送所有分支到远程仓库

$ git push [remote] --all
title: hexo配置
date: 1970-1-1
categories: 
- balabala
---

一直没在新电脑上配blog,今天把旧电脑搞炸了,所以在这里弄一个新的

# hexo踩坑

## 环境准备

准备git node hexo
git和node从官网上下载安装就行了
hexo:npm install -g hexo-cli

## 环境搭建

检查git node hexo是否成功安装,cd到blog目录下,hexo init
git生成pub,增加密钥对

## github配置

新建blog,增加分支
npm install hexo-deployer-git --save,安装插件



git init
git status

git add 123.txt
git commit -m "fix bug"

git diff 123.txt //diff文件


git log查看历史记录

HEAD表示当前版本

版本回退:
git reset --hard HEAD^   上个版本
git reset --hard HEAD^^ 上上个版本
git reset --hard HEAD~100 上100个版本 

或者

git reset --hard 1ae21 (这个是每个版本对应的sha的前几位,git可以遍历找到这个,注意不要输入的太少.符合条件的太多会混乱)hard是为了避免一堆奇怪的东西

git reflog 记录指令,可以去往未来

git checkout -- readme.md 未放到暂存区时恢复到原来的版本状态
放到暂存后,又做了更改,回退到暂存区状态
(用版本库替换工作区)

git reset HEAD <file> 可以将暂存区修改撤销回工作区

git rm <file>
git commit -m "remove xxx"

git branch 查看分支
git push origin master 提交到远程 
.gitignore可以增加一些忽略的文件

## 多用户

增加分支,将默认分支改为你这个分支,然后git clone下来
git branch检查分支是否为新建的分支

在新建的blog下配置deploy的参数,type branch等.
hexo d即可使用

注意提交的时候需要:
git add .
git commit -m "xxx"
git push
hexo d -g

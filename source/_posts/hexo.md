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

## 多用户

增加分支,将默认分支改为你这个分支,然后git clone下来
git branch检查分支是否为新建的分支

在新建的blog下配置deploy的参数,type branch等.
hexo d即可使用
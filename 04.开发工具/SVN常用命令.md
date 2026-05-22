---
tags:
  - SVN
---

# SVN 常用命令

SVN（Subversion）集中式版本控制工具的常用操作速查。与 [[git一般规范|Git]] 不同，SVN 无本地仓库概念，所有操作直接与中央服务器交互。

## 初始化仓库
```sh
svn co <仓库地址>
```

## 查看文件状态
```sh
svn st
```

## 显示详细信息
```sh
svn info
```

## 查看日志
```sh
svn log
```

## 更新目录
```sh
svn up
```

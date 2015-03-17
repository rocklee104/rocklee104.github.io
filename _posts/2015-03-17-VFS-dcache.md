---
layout: post
title: Linux VFS dcache分析
published: true
---

在linux的VFS中，dcache(directory entry cache)起了至关重要的作用. 它将部分文件名保存在内存中. 在查找文件名的时候减少了进行磁盘操作. 提高了文件系统的性能. 并且它和对应的文件inode关联起来. 是文件名和对应文件内容的粘合剂. 此外, linux目录树也是通过dcache组织起来的. 本文目前只关注dcache的分配和释放过程. 
## 1.dentry的状态


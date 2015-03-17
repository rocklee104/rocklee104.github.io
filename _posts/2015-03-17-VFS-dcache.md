---
layout: post
title: Linux VFS dcache分析
published: true
---

在linux的VFS中，cache(directory entry cache)起了至关重要的作用。它将部分文件名保存在内存中。在查找文件名的时候减少了进行磁盘操作。提高了文件系统的性能。并且它和对应的文件inode关联起来。是文件名和对应文件内容的粘合剂。此外，linux目录树也是通过dcache组织起来的。本文目前只关注dcache的分配和释放过程。
<h2 id="1">1.dentry的状态</h2>
dcache中的成员就是dentry，在dcache中存在的dentry分为下面四种状态：

- free:

处于该状态的目录项对象不包含有效的信息，还没有被VFS使用。它对应的内存区由slab分配器进行管理。

- unused:

处于该状态的目录项对象当前还没有被内核使用。该对象的引用计数器d\_count的值为0。但其d\_inode域仍然指向相关的索引节点。该目录项对象包含有效的信息，但为了在必要时回收内存，它的内容可能被丢弃。
      
- in use:

处于该状态的目录项对象当前正在被内核使用，该对象的引用计数器d\_count的值为正数，而其d\_inode域指向相关的索引节点对象。该目录项对象包含有效的信息。并且不能被丢弃。

- negtive:

与目录项相关的索引节点不复存在，那是因为相应的磁盘索引节点已被删除，该目录项对象的d\_inode是空的, 但该对象仍然被保存在目录项高速缓存中，以便后续对同一文件目录名的查找操作能够快速完成。

<h2 id="2">2.dcache的组成</h2>
dcache主要由两个数据结构组成:

- 哈希链表dentry_hashtable:

dcache中的所有对象都通过d\_hash指针域连接到相应的哈希链表中。

- 未使用的dentry对象链表dentry\_unused:

dcache中所有处于unused和negative状态的dentry对象都通过其指针域d\_lru链接到dentry\_unused链表中，该链表也称为LRU链表。

<h2 id="3">3.相关API分析</h2>
<h2 id="3.1">3.1 d_alloc</h2>
##\<fs/dcache.c>##

```c
struct dentry *d_alloc(struct dentry * parent, const struct qstr *name)
```
当创建一个dentry的时候，就需要调用d\_alloc。这个函数通过kmem_cache_alloc给dentry分配空间，将dentry的使用计数d\_count初始化为1。由于刚创建的dentry还没有加入hash表，因此其d\_flags域的值为DCACHE_UNHASHED。刚创建完成的dentry属于free状态.

<h2 id="3.2">3.2 d_add</h2>
##\<include/linux/dcache.h>##

```c
static inline void d_add(struct dentry *entry, struct inode *inode)
{
	d_instantiate(entry, inode);
	d_rehash(entry);
}
```
函数d_instantiate的作用就是将dentry和inode关联起来。对于函数d\_rehash，
其作用是将dentry放入hash表中。上文提到过，调用d\_alloc创建dentry的时候，dentry还没有加入hash表中，其d_flags域的值为DCACHE\_UNHASHED。调用过d\_add之后，d\_flags域的值为~DCACHE\_UNHASHED。

[1.dentry的状态](#1)
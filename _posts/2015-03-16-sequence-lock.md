---
layout: post
title: Linux内核顺序锁分析
published: true
---

本文初步分析linux-2.6.28顺序锁实现机制.
## 什么是顺序锁？
Sequence lock,也就是所谓的顺序锁.在读写者模型中,为了让读者和写者之间不加锁访问,内核引入了顺序锁的概念.
对于读者,访问资源都不需要使用锁.而写者和读者之间也不需要锁,只有写者和写者之间需要使用自旋锁.写者进入
临界区之后将一计数---sequence加1,在其退出临界区之后再次将sequence加1.Sequence序列只增加不减少,这就是
sequence lock的名称来历.
## 顺序锁的定义
**<include/linux/seqlock.h>**

    typedef struct {
        unsigned sequence;
        spinlock_t lock;
    } seqlock_t;
这个unsigned类型的sequence用来进行递增计数,lock用来进行多个写者之间的互斥.
## 顺序锁的初始化
- 静态初始化

我在这里去掉了一些调试信息,保留了静态初始化中最核心的部分.
**<include/linux/seqlock.h>**

```c
# define __SPIN_LOCK_UNLOCKED(lockname) \
	(spinlock_t)	{	.raw_lock = __RAW_SPIN_LOCK_UNLOCKED,	\
				SPIN_DEP_MAP_INIT(lockname) }
#define __SEQLOCK_UNLOCKED(lockname) \
		 { 0, __SPIN_LOCK_UNLOCKED(lockname) }
#define DEFINE_SEQLOCK(x) \
		seqlock_t x = __SEQLOCK_UNLOCKED(x)
```
- 动态初始化
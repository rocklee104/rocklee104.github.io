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
**\<include/linux/seqlock.h>**

    typedef struct {
        unsigned sequence;
        spinlock_t lock;
    } seqlock_t;
这个unsigned类型的sequence用来进行递增计数,lock用来进行多个写者之间的互斥.
## 顺序锁的初始化
- 静态初始化:

我在这里去掉了一些调试信息,保留了静态初始化中最核心的部分.
**\<include/linux/seqlock_types.h>**

```c    
# define __SPIN_LOCK_UNLOCKED(lockname) \
	(spinlock_t)	{	.raw_lock = __RAW_SPIN_LOCK_UNLOCKED,	\
				SPIN_DEP_MAP_INIT(lockname) }
```

**\<include/linux/seqlock.h>**

```c    
#define __SEQLOCK_UNLOCKED(lockname) \
		 { 0, __SPIN_LOCK_UNLOCKED(lockname) }
         
#define DEFINE_SEQLOCK(x) \
		seqlock_t x = __SEQLOCK_UNLOCKED(x)
```
以上可以看出顺序锁的初始化就是将sequence的值置0,以及初始化了spin_lock.
- 动态初始化:

如果要动态初始化一个seqlock_t变量,就需要调用seqlock_init了
**\<include/linux/seqlock.h>**

```c
#define seqlock_init(x)					\
	do {						\
		(x)->sequence = 0;			\
		spin_lock_init(&(x)->lock);		\
	} while (0)
```
这部分的工作和静态初始化是一样的.
##写者的上锁和解锁
- 上锁:

**\<include/linux/seqlock.h>**

```c
static inline void write_seqlock(seqlock_t *sl)
{
	//spin_lock能够保证smp环境下的数据同步
	spin_lock(&sl->lock);
	++sl->sequence;
	smp_wmb();
}
```
写入者在获取到spinlock后就将sequence序列自加,sequence对写者没有什么实际作用,但是对读者来说,这就是判断读取的数据是否有效的重要依据.这里调用smp_wmb()设置了内存屏障,是为了保证在退出write_seqlock之前,squence一定是自加过的.

- 解锁:

**\<include/linux/seqlock.h>**

```c
static inline void write_sequnlock(seqlock_t *sl)
{
	smp_wmb();
	//sequence & 0 == 0表示写入过程结束
	sl->sequence++;
	spin_unlock(&sl->lock);
}
```


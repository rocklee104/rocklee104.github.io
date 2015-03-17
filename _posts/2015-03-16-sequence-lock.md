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
<include/linux/seqlock.h>

```c
typedef struct {
	unsigned sequence;
	spinlock_t lock;
} seqlock_t;
```
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

如果要动态初始化一个seqlock\_t变量,就需要调用seqlock_init了
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
在实际的使用过程中,写者代码很可能是如下形式:

```c
...
write_seqlock(&lock);
write_sth();
write_sequnlock(&lock);
...
```
接下来我们就来分析一下写者的上锁和解锁过程.

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
写入者在获取到spinlock后就将sequence序列自加,sequence对写者没有什么实际作用,但是对读者来说,这就是判断读取的数据是否有效的重要依据.这里调用smp\_wmb()设置了内存屏障,是为了保证在退出write_seqlock之前,squence一定是自加过的.

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
在更新sequence序列之前,调用smp\_wmb()保证写入的数据已经完成写入操作.最后调用spin_unlock完成写者顺序锁的解锁.
##读者的操作
上文分析了写者的上锁和解锁操作,接下来继续分析读者的操作.还是给出一个实际应用的例子:

```c
do {
	seq = read_seqbegin(&lock);
    read_sth();
} while (read_seqretry(&lock, seq));
```
对于读者来说,不存在上锁和解锁,它只涉及到read\_seqbegin和read\_seqretry这两个函数.
**\<include/linux/seqlock.h>**

```c
static __always_inline unsigned read_seqbegin(const seqlock_t *sl)
{
	unsigned ret;

repeat:
	ret = sl->sequence;
	smp_rmb();
	//sequence最低位为0才表示写入结束
	if (unlikely(ret & 1)) {
		cpu_relax();
		goto repeat;
	}

	return ret;
}
```
这个函数用来第一次获取sequece,我们来看看它的实现.smp_rmb(),又是一个有意思的内存屏障,保证在if语句之前已经读取了sequence的值.这里的if判断sequence的最低位是否为1.如果是1,表示写者还没有退出临界区,这时候sequence的值是非常不可靠的.所以需要重新读取sequence直到所有的写者都退出临界区,这时才将读取到的sequence返回.

**\<include/linux/seqlock.h>**

```c
static __always_inline int read_seqretry(const seqlock_t *sl, unsigned start)
{
	smp_rmb();

	return (sl->sequence != start);
}
```
我们回过头再看看读者的操作那个例子.在首次获取到可靠的sequence之后,进行读操作.由于读者和写者之间不是互斥的,在读者读取临界区内容的时候,写者很可能也在临界区操作,这时候读者在临界区中获取的内容不可靠,所以判断是否有写者操作过临界区就非常重要了.判断在读者读取临界区内容期间是否有写者操作过临界区的方法就是再次判断sequence.内核屏障保证sequence读取的有序性.如果这次读者读取的sequence和首次读取的不一致,那么就说明在读者进行读操作的时候,有写者也进入了临界区,那么以上例子中的do{}while操作就需要再次进行.如果写操作非常频繁,那么就要经常重复读取,势必对系统系能产生影响.
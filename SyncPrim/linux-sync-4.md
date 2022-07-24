内核同步原语. 第四部分.
================================================================================

简介
--------------------------------------------------------------------------------

这是本章的第四部份[chapter](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/index.html) ，本章描述了内核中的同步原语, 且在之前的部分我们介绍完了 [spinlocks](https://en.wikipedia.org/wiki/Spinlock) 和 [semaphore](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 两种不同的同步原语. 我们将在这个章节持续学习 [synchronization primitives](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29) ，并考虑另一简称 [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion) 的同步原语，全名是 `MUTual EXclusion`。

与本 [book](https://0xax.gitbooks.io/linux-insides/content)前面所有章节一样, 我们将先尝试从理论面来探究此同步原语，在此基础上再探究 Linux内核所提供用来操作`mutexes`的 [API](https://en.wikipedia.org/wiki/Application_programming_interface)。

那，让我们开始吧.

`mutex`的概念
--------------------------------------------------------------------------------

从前一章 [part](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/sync-3.html)，我们已经很熟悉同步原语[semaphore](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 。 其表示为:

```C
struct semaphore {
	raw_spinlock_t		lock;
	unsigned int		count;
	struct list_head	wait_list;
};
```

此结构体包含一 [lock](https://en.wikipedia.org/wiki/Lock_%28computer_science%29)的状态和资讯以及由waiters所构成的一list. 根据 `count` 栏位的值, 一 `semaphore` 能将一资源的存取提公给若干此资源的需求者.  [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion) 的概念与 [semaphore](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 相似. 却有着一些差异.  `semaphore` 和 `mutex` 同步原语的最大差异在于 `mutex` 具有更严格的语义. 不像 `semaphore`, 单一时间一 `mutex`只能由单一 [process](https://en.wikipedia.org/wiki/Process_%28computing%29) 持有，并且只有该`mutex` 的 `owner` 能够对它进行释放或解锁的动作. 此外 `lock` [API](https://en.wikipedia.org/wiki/Application_programming_interface)在实现的差异上. `semaphore` 同步原语强置rescheduling在waiter list中的进程.  `mutex` lock `API` 的实现允许避免掉这种状况，进而减少这种昂贵的 [context switches](https://en.wikipedia.org/wiki/Context_switch)操作.

`mutex` 同步原语由下结构呈现:

```C
struct mutex {
        atomic_t                count;
        spinlock_t              wait_lock;
        struct list_head        wait_list;
#if defined(CONFIG_DEBUG_MUTEXES) || defined(CONFIG_MUTEX_SPIN_ON_OWNER)
        struct task_struct      *owner;
#endif
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
        struct optimistic_spin_queue osq;
#endif
#ifdef CONFIG_DEBUG_MUTEXES
        void                    *magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
        struct lockdep_map      dep_map;
#endif
};
```

在Linux内核中的结构. 此结构被定义在 [include/linux/mutex.h](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件并且包含着与 `semaphore` 结构体类似的栏位. `mutex` 结构的第一个栏位是 - `count`. 此栏位的值代表着一 `mutex`的状态. 在`count`栏位为 `1`得情况下, 说明该 `mutex` 处于 `unlocked` 状态. 当 `count` 栏位值处于 `zero`, 表该 `mutex` 处于 `locked` 状态. 此外， `count`栏位的值可能是 `negative`的. 在这种情况下表该 `mutex` 处于 `locked` 状态且有其他可能的等待者.

`mutex` 结构的下俩个栏位 - `wait_lock` 和 `wait_list` are [spinlock](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) for the protection of a `wait queue` and list of waiters which represents this `wait queue` for a certain lock. 你可能注意到了,  `mutex` 和 `semaphore` 结构体的相似性结束了. 剩余的 `mutex` 结构体栏位, 正如同我们所见，取决于Linux内核的不同配置选项.

第一个栏位 - `owner` 代表着持有该lock的 [process](https://en.wikipedia.org/wiki/Process_%28computing%29) . 如我们看到的, 这个栏位是否于 `mutex` 结构体中存在取决于 `CONFIG_DEBUG_MUTEXES` 或 `CONFIG_MUTEX_SPIN_ON_OWNER` 的内核配置选项. 这个栏位汗下个`osq`栏位的主要用途在于支援我们之后会介绍的`optimistic spinning`功能. 最后两个部份 - `magic` 和 `dep_map` 仅被用于 [debugging](https://en.wikipedia.org/wiki/Debugging) 模式.  `magic` 栏位用来存储 `mutex` 用来除错的相关资讯而第二个栏位 - `lockdep_map` 则用于Linux内核的 [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt) 中.

于此, 在我们已经探究完 `mutex` 的结构体后, 我们可以开始思考此同步原语是如何在 Linux 内核中运作的. 你可能会猜想, 一个想获取锁的 process , 在允许的情况下，就对 `mutex->count` 值做减去的操作. 而如果process希望释放该lock, 则增加该栏位的值. 没错. 但也正如您可能也猜到的那样，它在Linux内核中可能不会那么简单。

事实上, 当一 process 试图获取一 `mutex`时, 有三种可能的路径:

* `fastpath`;
* `midpath`;
* `slowpath`.

会采取哪种路径, 将根据当前 `mutex`的状态而定. 第一条路径或 `fastpath` 是最快的，正如您从其名称中可以理解的那样。 在这种情况下锁有事情都很简单. 因一 `mutex` 没被任何人获取, 其 `mutex` 结构体中的`count`值可以直接被进行递减的操作. 在释放一 `mutex`的情况下, 算法是一样的. 一 process 对该`mutex`结构体中的`count`栏位值进行递增的操作. 当然, 所有的这些操作都必须是 [atomic](https://en.wikipedia.org/wiki/Linearizability)的.

是的, 这看起来很简单. 但是，如果一个进程想要获取一个已经被其他进程获取的`mutex` ，会发生什么呢？ 在这种情况下, 控制流成将被转移给第二条路径决定 - `midpath`. The `midpath` 或 `optimistic spinning` tries to [spin](https://en.wikipedia.org/wiki/Spinlock) with already familiar for us [MCS lock](http://www.cs.rochester.edu/~scott/papers/1991_TOCS_synch.pdf) while the lock owner is running. This path will be executed only if there are no other processes ready to run that have higher priority. 这条路径被称作 `optimistic` 是因为等待的进程不会被睡眠或是调度. 这可以避免掉昂贵的[context switch](https://en.wikipedia.org/wiki/Context_switch).

在最后一种情况, 当 `fastpath` 和 `midpath` 都不被执行时, 就会执行最后一条路径 - `slowpath` . 这条路径的行为跟 [semaphore](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 的lock操作相似. 如果该锁无法被进程获取, 那么此进程将被加入 `wait queue`，结构如下表示:

```C
struct mutex_waiter {
        struct list_head        list;
        struct task_struct      *task;
#ifdef CONFIG_DEBUG_MUTEXES
        void                    *magic;
#endif
};
```

结构定义在[include/linux/mutex.h](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件且使用上可能会被睡眠. 在研究Linux 内核提公用来操作`mutexes`的 [API](https://en.wikipedia.org/wiki/Application_programming_interface)之前，让我们先严就 `mutex_waiter` 结构. 如果你有阅读此章节的 [previous part](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/sync-3.html) , 你可以能注意到 `mutex_waiter` 结构体与 (https://github.com/torvalds/linux/blob/master/kernel/locking/semaphore.c) 源代码中的 `semaphore_waiter` 结构体相似:

```C
struct semaphore_waiter {
        struct list_head list;
        struct task_struct *task;
        bool up;
};
```

它也包含 `list` 和 `task` 栏位，用来表示该 mutex 的等待队列. 其中一个在这之间的差异是 `mutex_waiter` 没有`up` 栏位, 但却有一个可以根据内核设定`CONFIG_DEBUG_MUTEXES`选项而存在的栏位 `magic`，它能用来存储一些在除错`mutex`问题时有用的相关资讯。

现在我们知道了什么是 `mutex` 以及他如何在 Linux 内核中呈现的. 在这种情况下，我们可以开始一窥 Linux 内核所提公用来操作`mutexes`的 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 。

Mutex API
--------------------------------------------------------------------------------

Ok, 在前面的章节我们已经了解什么是 `mutex` 原语，而且看到了用来代表Linux内核中的`mutex`的`mutex`结构体. 现在是时后开始研究用来操弄mutexes的 [API](https://en.wikipedia.org/wiki/Application_programming_interface) . 详细的 `mutex` API 被记录在 [include/linux/mutex.h](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) header file. 一如既往，在考虑如何获取和释放一`mutex`之前，我们需要知道如何初始化它。

初始化 `mutex`有两种方法. 第一种事静态地. 为此，Linux内核提供以下macro:

```C
#define DEFINE_MUTEX(mutexname) \
        struct mutex mutexname = __MUTEX_INITIALIZER(mutexname)
```

让我们来研究此macro的实现. 正如我们所见,  `DEFINE_MUTEX` macro 接受新定义的 `mutex` 名称，并将其扩展成一个新的 `mutex` 解构. 此外新 `mutex` 结构体由 `__MUTEX_INITIALIZER` macro 初始化. 让我们看看 `__MUTEX_INITIALIZER`的实现:

```C
#define __MUTEX_INITIALIZER(lockname)         \
{                                                             \
       .count = ATOMIC_INIT(1),                               \
       .wait_lock = __SPIN_LOCK_UNLOCKED(lockname.wait_lock), \
       .wait_list = LIST_HEAD_INIT(lockname.wait_list)        \
}
```

这个宏被定义在 [same](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件， 并且我们可以了解到它初始化了 `mutex` 解构体中的栏位.  `count` 被初始化为 `1` ，这代表该mutex状态为 `unlocked`. `wait_lock` [spinlock](https://en.wikipedia.org/wiki/Spinlock) 被初始化为解锁状态，而最后的栏位 `wait_list` 被初始化为空的 [doubly linked list](https://0xax.gitbooks.io/linux-insides/content/DataStructures/dlist.html).

第二种做法允许我们动态初始化一个 `mutex`. 为此我们需要呼叫在 [kernel/locking/mutex.c](https://github.com/torvalds/linux/blob/master/kernel/locking/mutex.c) 源挡案的 `__mutex_init` 函数. 事实上, 大家很少直接呼叫 `__mutex_init` 函数. 取而代之， 我们使用 `mutex_init` 如下:

```C
# define mutex_init(mutex)                \
do {                                                    \
        static struct lock_class_key __key;             \
                                                        \
        __mutex_init((mutex), #mutex, &__key);          \
} while (0)
```

我们可以看到 `mutex_init` macro 定义了 `lock_class_key` 并且呼叫 `__mutex_init` 函数. 让我们看看这个函数的实现:

```C
void
__mutex_init(struct mutex *lock, const char *name, struct lock_class_key *key)
{
        atomic_set(&lock->count, 1);
        spin_lock_init(&lock->wait_lock);
        INIT_LIST_HEAD(&lock->wait_list);
        mutex_clear_owner(lock);
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
        osq_lock_init(&lock->osq);
#endif
        debug_mutex_init(lock, name, key);
}
```

如我们所见， `__mutex_init` 函数接收三个参数:

* `lock` - a mutex itself;
* `name` - name of mutex for debugging purpose;
* `key`  - key for [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt).

在 `__mutex_init` 函数的一开始, 我们可以看到 `mutex` 状态的初始化. 我们透过`atomic_set`函数atomically地将给定的变数赋予给定的值来将`mutex`的状态设为 `unlocked` 。 之后我们可以看到After this we may see initialization of the `spinlock` to the unlocked state which will protect `wait queue` of the `mutex` and initialization of the `wait queue` of the `mutex`. 之后我们清除该 `lock` 的持有者并且透过呼叫 [include/linux/osq_lock.h](https://github.com/torvalds/linux/blob/master/include/linux/osq_lock.h) 头文件中的`osq_lock_init`函数来初始化optimistic queue. 而此函数也只是将  optimistic queue 的tail设定为 unlocked state而以:

```C
static inline bool osq_is_locked(struct optimistic_spin_queue *lock)
{
        return atomic_read(&lock->tail) != OSQ_UNLOCKED_VAL;
}
```

在 `__mutex_init` 函数的结尾阶段，我们可以看到他呼叫了 `debug_mutex_init` 函数, 不过就如同我在此章节前面提及的那样 [chapter](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/index.html), 我们不会在此章节讨论与除错相关的内容。

在`mutex`结构被初始化后, 我们可以继续研究 `mutex` 同步原语的`lock` 和 `unlock` API. `mutex_lock` 和 `mutex_unlock` 函数的实现位于 [kernel/locking/mutex.c](https://github.com/torvalds/linux/blob/master/kernel/locking/mutex.c) 源码. 首先，让我们从 `mutex_lock`的实现开始吧. 他长这样:

```C
void __sched mutex_lock(struct mutex *lock)
{
        might_sleep();
        __mutex_fastpath_lock(&lock->count, __mutex_lock_slowpath);
        mutex_set_owner(lock);
}
```

我们可以看到[include/linux/kernel.h](https://github.com/torvalds/linux/blob/master/include/linux/kernel.h)镖头 We may see the call of the `might_sleep` macro from the [include/linux/kernel.h](https://github.com/torvalds/linux/blob/master/include/linux/kernel.h) header file at the beginning of the `mutex_lock` function. 该macro的实现取决于  `CONFIG_DEBUG_ATOMIC_SLEEP` 内核配置选项，而如果这个选项被启用, 此 macro 会在[atomic](https://en.wikipedia.org/wiki/Linearizability)上下文中被执行时打印 stack trace. 此 macro 作为除错用途上的帮手. 除此之外，这个宏什么都不做.

在`might_sleep` macro之后, 我们可以看到 `__mutex_fastpath_lock` 函数被呼叫. 此函数是 architecture-specific 的并且因为我们此书是探讨 [x86_64](https://en.wikipedia.org/wiki/X86-64) 架构, \ `__mutex_fastpath_lock` 的实现部份位于 [arch/x86/include/asm/mutex_64.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/mutex_64.h) 头文件. 正如我们从`__mutex_fastpath_lock`函数名称可以理解的那样，此函数将尝试透过 fast path 获取一个锁，或者换句话说此函数试图减去一个给定mutex锁的 `count`值.

`__mutex_fastpath_lock` 函数的实现由两部份组成. 第一部份是 [inline assembly](https://0xax.gitbooks.io/linux-insides/content/Theory/asm.html) statement. 让我们看看:

```C
asm_volatile_goto(LOCK_PREFIX "   decl %0\n"
                              "   jns %l[exit]\n"
                              : : "m" (v->counter)
                              : "memory", "cc"
                              : exit);
```

首先, 让我们将注意力放在 `asm_volatile_goto`. 这个宏被定义在 [include/linux/compiler-gcc.h](https://github.com/torvalds/linux/blob/master/include/linux/compiler-gcc.h) 头文件并且只是扩展成两个 inline assembly statements:

```C
#define asm_volatile_goto(x...) do { asm goto(x); asm (""); } while (0)
```

The first assembly statement contains `goto` specificator and the second empty inline assembly statement is [barrier](https://en.wikipedia.org/wiki/Memory_barrier). Now let's return to the our inline assembly statement. As we may see it starts from the definition of the `LOCK_PREFIX` macro which just expands to the [lock](http://x86.renejeschke.de/html/file_module_x86_id_159.html) instruction:

```C
#define LOCK_PREFIX LOCK_PREFIX_HERE "\n\tlock; "
```

As we already know from the previous parts, this instruction allows to execute prefixed instruction [atomically](https://en.wikipedia.org/wiki/Linearizability). So, at the first step in the our assembly statement we try decrement value of the given `mutex->counter`. At the next step the [jns](http://unixwiz.net/techtips/x86-jumps.html) instruction will execute jump at the `exit` label if the value of the decremented `mutex->counter` is not negative. The `exit` label is the second part of the `__mutex_fastpath_lock` function and it just points to the exit from this function:

```C
exit:
        return;
```

就目前而已， `__mutex_fastpath_lock` 函数的实现看起来挺简单的. 但 `mutex->counter` 的值有可能在may be negative after increment. In this case the: 

```C
fail_fn(v);
```

will be called after our inline assembly statement. `fail_fn` 是`__mutex_fastpath_lock` 函数的第二个参数，其为一指标指到获取给定锁的 `midpath/slowpath` 路径函数. 在我们的例子中`fail_fn` 是 `__mutex_lock_slowpath` 函数. 在我们一窥 `__mutex_lock_slowpath` 函数的实现前, 我们先看完 `mutex_lock` 函数的实现吧. 在最简单的情况下, 锁会被一进城透过执行 `__mutex_fastpath_lock` 路径后成功被获取. 这种情况下, 我们只差在 `mutex_lock`的结尾呼叫

```C
mutex_set_owner(lock);
```

 `mutex_set_owner` 函数被定义在 [kernel/locking/mutex.h](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件，其将一个lock的持有者设为当前进程:

```C
static inline void mutex_set_owner(struct mutex *lock)
{
        lock->owner = current;
}
```

另外一种情况, 让我们研究一进程因为锁已被其它进程持有，而无法顺利获得的情况,  我们已经知道在这种情境下 `__mutex_lock_slowpath` 函数会被呼叫. 让我们开始研究这个函数. 此函数被定义在 [kernel/locking/mutex.c](https://github.com/torvalds/linux/blob/master/kernel/locking/mutex.c) 源代码 ，并且从获取正确的mutexand starts from the obtaining of the proper mutex by the mutex state given from the `__mutex_fastpath_lock` with the `container_of` macro:

```C
__visible void __sched
__mutex_lock_slowpath(atomic_t *lock_count)
{
        struct mutex *lock = container_of(lock_count, struct mutex, count);

        __mutex_lock_common(lock, TASK_UNINTERRUPTIBLE, 0,
                            NULL, _RET_IP_, NULL, 0);
}
```

之后将获得的`mutex`带入函数 `__mutex_lock_common` 进行呼叫 .  `__mutex_lock_common` 函数一开始会关闭 [preemtion](https://en.wikipedia.org/wiki/Preemption_%28computing%29) 直到下次的重新调度:

```C
preempt_disable();
```

之后我们进到optimistic spinning阶段. 如同我们知道的，这个阶段取决于`CONFIG_MUTEX_SPIN_ON_OWNER` 内核配置选项. 如此选项被禁用, 我们就跳过这个阶段并迈入最后一条路径 - 获取一 `mutex`的`slowpath`:

```C
if (mutex_optimistic_spin(lock, ww_ctx, use_ww_ctx)) {
        preempt_enable();
        return 0;
}
```

First of all the `mutex_optimistic_spin` function check that we don't need to reschedule or in other words there are no other tasks ready to run that have higher priority. If this check was successful we need to update `MCS` lock wait queue with the current spin. In this way only one spinner can complete for the mutex at one time:

```C
osq_lock(&lock->osq)
```

At the next step we start to spin in the next loop:

```C
while (true) {
    owner = READ_ONCE(lock->owner);

    if (owner && !mutex_spin_on_owner(lock, owner))
        break;

    if (mutex_try_to_acquire(lock)) {
        lock_acquired(&lock->dep_map, ip);

        mutex_set_owner(lock);
        osq_unlock(&lock->osq);
        return true;
    }
}
```

and try to acquire a lock. First of all we try to take current owner and if the owner exists (it may not exists in a case when a process already released a mutex) and we wait for it in the `mutex_spin_on_owner` function before the owner will release a lock. If new task with higher priority have appeared during wait of the lock owner, we break the loop and go to sleep. In other case, the process already may release a lock, so we try to acquire a lock with the `mutex_try_to_acquired`. If this operation finished successfully, we set new owner for the given mutex, removes ourself from the `MCS` wait queue and exit from the `mutex_optimistic_spin` function. At this state a lock will be acquired by a process and we enable [preemtion](https://en.wikipedia.org/wiki/Preemption_%28computing%29) and exit from the `__mutex_lock_common` function:

```C
if (mutex_optimistic_spin(lock, ww_ctx, use_ww_ctx)) {
    preempt_enable();
    return 0;
}

```

That's all for this case.

In other case all may not be so successful. For example new task may occur during we spinning in the loop from the `mutex_optimistic_spin` or even we may not get to this loop from the `mutex_optimistic_spin` in a case when there were task(s) with higher priority before this loop. Or finally the `CONFIG_MUTEX_SPIN_ON_OWNER` kernel configuration option disabled. In this case the `mutex_optimistic_spin` will do nothing:

```C
#ifndef CONFIG_MUTEX_SPIN_ON_OWNER
static bool mutex_optimistic_spin(struct mutex *lock,
                                  struct ww_acquire_ctx *ww_ctx, const bool use_ww_ctx)
{
    return false;
}
#endif
```

在所有这种情况下,  `__mutex_lock_common`函数的行为就像`semaphore`. 我们就再次尝试获取锁，因为锁的持有者在此之前可能已经释放了它:

```C
if (!mutex_is_locked(lock) &&
   (atomic_xchg_acquire(&lock->count, 0) == 1))
      goto skip_wait;
```

在失败的情况下，希望获取锁的进程将被添加到等待者列表中

```C
list_add_tail(&waiter.list, &lock->wait_list);
waiter.task = task;
```

在成功的情况下，我们更新该所的持有者、允许抢占，并离开 `__mutex_lock_common` 韩ㄕㄨ:

```C
skip_wait:
        mutex_set_owner(lock);
        preempt_enable();
        return 0; 
```

In this case a lock will be acquired. If can't acquire a lock for now, we enter into the following loop:

```C
for (;;) {

    if (atomic_read(&lock->count) >= 0 && (atomic_xchg_acquire(&lock->count, -1) == 1))
        break;

    if (unlikely(signal_pending_state(state, task))) {
        ret = -EINTR;
        goto err;
    } 

    __set_task_state(task, state);

     schedule_preempt_disabled();
}
```

where try to acquire a lock again and exit if this operation was successful. Yes, we try to acquire a lock again right after unsuccessful try  before the loop. We need to do it to make sure that we get a wakeup once a lock will be unlocked. Besides this, it allows us to acquire a lock after sleep.  In other case we check the current process for pending [signals](https://en.wikipedia.org/wiki/Unix_signal) and exit if the process was interrupted by a `signal` during wait for a lock acquisition. In the end of loop we didn't acquire a lock, so we set the task state for `TASK_UNINTERRUPTIBLE` and go to sleep with call of the `schedule_preempt_disabled` function.

That's all. We have considered all three possible paths through which a process may pass when it will wan to acquire a lock. Now let's consider how `mutex_unlock` is implemented. When the `mutex_unlock` will be called by a process which wants to release a lock, the `__mutex_fastpath_unlock` will be called from the  [arch/x86/include/asm/mutex_64.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/mutex_64.h)  header file:

```C
void __sched mutex_unlock(struct mutex *lock)
{
    __mutex_fastpath_unlock(&lock->count, __mutex_unlock_slowpath);
}
```

`__mutex_fastpath_unlock` 函数的实现与`__mutex_fastpath_lock` 函数非常相似:

```C
static inline void __mutex_fastpath_unlock(atomic_t *v,
                                           void (*fail_fn)(atomic_t *))
{
       asm_volatile_goto(LOCK_PREFIX "   incl %0\n"
                         "   jg %l[exit]\n"
                         : : "m" (v->counter)
                         : "memory", "cc"
                         : exit);
       fail_fn(v);
exit:
       return;
}
```

Actually, there is only one difference. We increment value if the `mutex->count`. So it will represent `unlocked` state after this operation. As `mutex` released, but we have something in the `wait queue` we need to update it. In this case the `fail_fn` function will be called which is `__mutex_unlock_slowpath`. The `__mutex_unlock_slowpath` function just gets the correct `mutex` instance by the given `mutex->count` and calls the `__mutex_unlock_common_slowpath` function:

```C
__mutex_unlock_slowpath(atomic_t *lock_count)
{
      struct mutex *lock = container_of(lock_count, struct mutex, count);

      __mutex_unlock_common_slowpath(lock, 1);
}
```

在`__mutex_unlock_common_slowpath` 函数中，如果wait queue非空，我们将从中获取第一个条目，并唤醒相关的进程:

```C
if (!list_empty(&lock->wait_list)) {
    struct mutex_waiter *waiter =
           list_entry(lock->wait_list.next, struct mutex_waiter, list); 
                wake_up_process(waiter->task);
}
```

在此之后，前一个进程将释放互斥锁，由另一个在等待队列中的进程获取互斥锁。.

这就是全部了. 我们已经研究完主要用来操作`mutexes`的API `mutex_lock` 和 `mutex_unlock` 了.除此之外，Linux内核还提供以下API:

* `mutex_lock_interruptible`;
* `mutex_lock_killable`;
* `mutex_trylock`.

以及对应相同前缀的 `unlock` 函数. 我们就不在此解释这些 `API`了, 因为他们跟 `semaphores`所提供的`API`类似. 你可以透过阅读 [previous part](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/sync-3.html)来了解更多.

以上.

Conclusion
--------------------------------------------------------------------------------

This is the end of the fourth part of the [synchronization primitives](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29) chapter in the Linux kernel. In this part we met with new synchronization primitive which is called - `mutex`. From the theoretical side, this synchronization primitive very similar on a [semaphore](https://en.wikipedia.org/wiki/Semaphore_%28programming%29). Actually, `mutex` represents binary semaphore. But its implementation differs from the implementation of `semaphore` in the Linux kernel. In the next part we will continue to dive into synchronization primitives in the Linux kernel.

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)
* [Spinlock](https://en.wikipedia.org/wiki/Spinlock)
* [Semaphore](https://en.wikipedia.org/wiki/Semaphore_%28programming%29)
* [Synchronization primitives](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
* [API](https://en.wikipedia.org/wiki/Application_programming_interface) 
* [Locking mechanism](https://en.wikipedia.org/wiki/Lock_%28computer_science%29)
* [Context switches](https://en.wikipedia.org/wiki/Context_switch)
* [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [Atomic](https://en.wikipedia.org/wiki/Linearizability)
* [MCS lock](http://www.cs.rochester.edu/~scott/papers/1991_TOCS_synch.pdf)
* [Doubly linked list](https://0xax.gitbooks.io/linux-insides/content/DataStructures/dlist.html)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Inline assembly](https://0xax.gitbooks.io/linux-insides/content/Theory/asm.html)
* [Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier)
* [Lock instruction](http://x86.renejeschke.de/html/file_module_x86_id_159.html)
* [JNS instruction](http://unixwiz.net/techtips/x86-jumps.html)
* [preemtion](https://en.wikipedia.org/wiki/Preemption_%28computing%29)
* [Unix signals](https://en.wikipedia.org/wiki/Unix_signal)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/sync-3.html)

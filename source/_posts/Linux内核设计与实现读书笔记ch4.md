# Linux内核设计与实现第四章  进程调度

进程调度程序可以被看做是一个在可运行态进程之间分配有限处理器时间资源的内核子系统。也是Linux多任务的基础。只有有了合理的调度，系统资源才能最大化的发挥作用。多进程才会有并发执行的效果。

### 4.1 多任务

多任务系统可以分为两类：非抢占式多任务（`cooperative multitasking`），和抢占式多任务（`preemptive multitasking`）。

在抢占式的多任务模式下，由调度程序来决定什么时候停止一个进程的运行，以便其他程序能够得到执行的机会。也就是抢占。进程在被抢占之前的时间是预先设置好的，也就是我们通常说的时间片(`timeslice`)。

而相反的是，在非抢占式的多任务模式下，除非进程自己停止运行，否则将会一直执行。进程主动挂起自己的操作成为让步(`yielding`)。理想情况下，进程会做出让步，以便使每个可运行进程都享有足够的处理器时间。但这种方式缺点很多：调度程序无法对进程独占处理器的时间做出规定。这样就会导致进程独占处理器的时间超乎预料。而且一个决不让步的进程（hang住的进程）就能使得处理器崩溃。所以说Linux一开始就使用的是抢占式多任务。

### 4.2 Linux的进程调度

从内核版本2.5开始，进程调度采用了一种叫做O(1)调度程序的新程序。这个明明也是因为其算法行为来命名的。

O(1)算法虽然在数以十计的多处理器环境下表现了近乎完美的性能以及可扩展性，但是对于响应时间敏感的程序（交互进程）却存在不足。所以虽然在大服务器上工作负载很理想，但是在桌面系统下表现不佳。所以在2.6版本开发初期引入了一种新的进程调度算法，也就是反转楼梯最后期限调度算法（`Rotating Staircase Deadlilne scheduler` RSDL）。该算法吸取队列理论，引入了公平调度的概念。并且在2.6.23版本中替代了O(1)调度算法。被称为“完全公平调度算法”（CFS）

### 4.3 策略

#### 4.3.1 I/O消耗型和处理器消耗型的进程

进程可以分为两种：I/O消耗型和处理器消耗型。

I/O消耗型进程指进程的大部分时间都在提交I/O请求或者等待I/O请求，因此进程经常处于可运行状态。但通常都是运行短短的一会，因为进程在等待更多的I/O请求时最后总会堵塞。例如GUI程序，即使不读取或者写入硬盘，大多数时间中也在等待来自鼠标或者键盘的用户交互操作。

处理器消耗型的进程把大多数时间都用在执行代码上。除非被抢占，否则通常一直不停运行。因为他们没有过多的I/O请求。但是，因为他们不属于I/O驱动的类型，所以从系统响应速度考虑，系统不应该让他们经常运行。所以对于这类的程序，策略调度往往是尽量降低他们的调度频率，从而延长其运行时间。处理器消耗型最极端的例子就是无线循环的执行。

但这种分类并非绝对的。所以调度策略通常要在这两个矛盾的目标中寻找平衡：进程响应时间短和最大系统利用率。为了满足需求，调度进程会使用一套复杂的算法来保证进程被公平对待。Linux主要对进程的响应做了优化，但是也没有忽略处理器消耗型的进程。

#### 4.3.2 进程优先级

调度算法中最基本的一类就是基于优先级的调度。其通常做法是优先级高的进程先运行，优先级低的进程后运行。相同优先级的进程按照轮转方式进行调度。在某些系统中，优先级高的进程使用的时间片也更长，调度程序总会选择时间片未用尽且优先级最高的进程运行。用户和操作系统都可以通过设置进程的优先级来影响系统的调度。

Linux采用了两种不同的优先级，第一种是nice值，其范围为-20——19，默认为0，越大的nice值代表着更低的优先级。也就是说，低nice值的进程可以获得更多的处理器时间。在Linux中，nice值代表时间片的比例。可以通过`ps-el`查看操作系统中的进程列表，其中标记着`NI`的就是进程对应的nice值

```shell
root@fasii:~# ps -el
F S   UID     PID    PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
4 S     0       1       0  0  80   0 - 42400 ep_pol ?        00:00:01 systemd
1 S     0       2       0  0  80   0 -     0 kthrea ?        00:00:00 kthreadd
1 I     0       3       2  0  60 -20 -     0 rescue ?        00:00:00 rcu_gp
1 I     0       4       2  0  60 -20 -     0 rescue ?        00:00:00 rcu_par_gp
1 I     0       6       2  0  60 -20 -     0 worker ?        00:00:00 kworker/0:0H-kblockd
1 I     0       8       2  0  60 -20 -     0 rescue ?        00:00:00 mm_percpu_wq
1 S     0       9       2  0  80   0 -     0 smpboo ?        00:00:00 ksoftirqd/0
1 I     0      10       2  0  80   0 -     0 rcu_gp ?        00:00:01 rcu_sched
1 S     0      11       2  0 -40   - -     0 smpboo ?        00:00:00 migration/0
5 S     0      12       2  0   9   - -     0 smpboo ?        00:00:00 idle_inject/0
1 I     0      13       2  0  80   0 -     0 worker ?        00:00:00 kworker/0:1-events_freezable
1 S     0      14       2  0  80   0 -     0 smpboo ?        00:00:00 cpuhp/0
```

第二种范围是实时优先级，其值是可以配置的，默认情况的范围是0-99，与nice意义恰好相反，越高的实时优先级代表进程的优先级也就越高。任何实时进程的优先级都高于普通进程的优先级。所以实时优先级和nice处于两个不同的范畴。Linux实时优先级参考了Unix的相关标准，可以通过命令`ps-el`查看，对应的实时优先级在`rtprio(PRI)`列下。进程对应列显示为-的代表其不是实时进程。

#### 4.3.3 时间片

时间片是一个能代表着进程在被抢占之前能够运行的时间的数值，调度策略必须规定一个默认的时间片，时间片过长会让人觉得系统无法并发执行程序，时间片过短则会明显增大进程切换带来的处理器耗时。这里也能显示出I/O消耗型程序和处理器消耗型的进程之间的矛盾：I/O消耗型不需要长时间片，但是处理器消耗型进程则希望越长越好。（时间片越长cache命中率更高）

Linux的CFS调度器没有直接将时间片分配到进程，而只是将从处理器的使用比划分给进程。这也就是说，进程所获得的处理器时间其实与系统负载密切相关。这个比例还会进一步受到nice值的影响。Linux中的CFS调度器的抢占时机取决于新的可运行程序消耗了多少处理器使用比。如果消耗的使用比当前进程小，则新进程立刻投入运行，抢占当前进程。否则，将推迟其运行。

### 4.4 Linux调度算法

#### 4.4.1 调度器类

Linux调度器是以模块方式提供的，这样的目的是允许不同类型的进程能够针对性的选择调度算法。

这种结构被称为调度器类（`scheduler classes`）。它允许多种不同的可动态添加的调度算法并存，调度属于自己范畴内的进程。每个调度器都有一个优先级，基础的调度器代码定义在`kernel/sched.c`中，它会按照优先级顺序遍历调度类，拥有一个可执行进程的最高优先级的调度器类会胜出，来执行下面要执行的程序。

完全公平调度（CFS）是针对普通进程的调度类，在Linux中称为`SCHED_NORMAL`，其具体实现在`kernel/sched/fair.c`中。

#### 4.4.2 Unix系统中的进程调度

在讨论公平调度算法之前，我们不妨先了解以下传统Unix系统的调度过程。在Unix系统上，优先级以nice值形式输出给用户空间，听起来很简单，但是却会导致许多反常问题。

+ 若将nice值映射到时间片，可能会导致进程无法切换最优运行。给定高nice值的进程往往是后台进程，多为计算密集型。而普通优先级的进程多为前台用户任务。针对时间片100ms来说，两个nice值为0的进程会分别获得50ms运行时间，但是两个nice值为19的进程却只会分别获得5ms的运行时间。这种时间片分法严重浪费了CPU资源。
+ 第二个问题涉及相对nice值。假设我们有两个进程，nice值分别为0和1的时候两个时间片为100和95,相差不会很大，但是18和19的两个nice值就会导致两个时间片为5ms和10ms。处理器的时间差距在两倍以上。但是nice值通常使用相对值，也就是在原值上增加或者减少。
+ 第三个问题是执行nice值到时间片的映射，就要能分配一个绝对时间片。在大多数系统中，时间片的要求是定时器节拍的整数倍。其次，连续的nice值映射到时间片，其差别范围在1ms~10ms之间。最后，时间片要随着定时器节拍改变。（具体见第11章）
+ 第四个问题是基于优先级的调度器为了优化交互任务而唤醒相关进程的问题。为了能让进程尽快投入与逆行，对新唤醒的进程要提升其优先级。即使它的时间片用尽。这种情况确实能提升交互任务的性能。但是也会导致一些特殊的睡眠/唤醒用例一些优待。使得给定进程打破公平原则，损害其他进程的利益。

上面的大多数问题均可以通过对传统的unix调度器进行改造来解决，比如：将nice值几何增加而非算数增加，采用新的度量机制将nice值到时间片的映射与定时器节拍隔离。但是这些解决方案都规避了问题的实质：分配绝对的时间片引发的固定的切换频率。所以CFS做了根本性的重新设计：完全摒弃时间片而分配给进程一个处理器的比重。这样来确保进程调度中具有恒定的公平性而且将切换频率置于不断变动中。

#### 4.4.3 公平调度

CFS的出发点基于一个简单的概念：进程调度的效果应该如同一个理想中的完美多任务处理器：每个进程能获得1/n的处理器时间。同时，我们调度给这n个进程同样多的运行时间。

当然，上述理想并非现实：在一个处理器上无法同时运行多个进程。而且进程无限小的周期也是低效的。进程的换入换出的本身存在消耗，而且影响缓存的效率。CFS充分考虑了这些带来的额外消耗，允许每个进程运行一段时间、循环轮转、选择运行最少的进程作为下一个运行进程。而不再采用给每个进程分配时间片的做法。CFS以所有可运行进程总数为基础，计算出一个进程应该运行多久，nice值则用来作为进程获得处理器运行比的权重。这样，每个进程都按其权重在全部可运行线程中所占比例的时间片来运行。为了准确计算时间片，CFS为完美多任务中的无限小调度周期的近似值设立了一个目标，称作目标延迟。越小的调度周期也就代表着越好的交互性，也就更接近完美多任务。代价就是频繁切换带来的高开销以及缓存换入换出带来的系统吞吐能力变差。

从上面式子中可以看到，当任务数量趋于无限时，他们各自所获得的处理器使用比和时间均将趋于0。其切换损耗也就越来越不可接受。为此，CFS引入了进程获得的时间片底线：最小粒度。其默认值为1ms。也就是说，无论存在多少个进程等待运行，每个进程最少也能获得1ms的运行时间。这样也就确保了消耗被限制在一定的范围内。

同样再来看nice值：比如设定nice值为5的进程是nice值为5的进程的1/3。以20ms的时间片为例，两个nice值为0和5进程能分配到的时间是15ms和5ms，两个nice值为10和15进程能分配到的时间也是15ms和5ms。可见，在CFS中，绝对的nice值不再影响调度决策，只有相对值才影响处理器时间的分配比例。

### 4.5 Linux调度实现

Linux调度实现的相关代码位于`kernel/sched/fair.c`中，我们主要关注四部分：

+ 时间记账
+ 进程选择
+ 调度器入口
+ 睡眠和唤醒

#### 4.5.1 时间记账

所有调度器都必须对进程运行时间记账。每次系统节拍发生时，当前运行的进程的时间片都会减少一个节拍周期。当该进程的时间片减少到0时，就会被另一个尚未减到零的时间片的可运行进程抢占

1. ##### 调度器实体结构

CFS不再拥有时间片的概念，但是也必须维护每个进程的时间记账。以此来确保每个进程只在公平分配给它的处理器时间内运行。CFS使用调度器实体结构（`<linux/sched.h> 的 struct sched_entity`中定义）来追踪进程记账。

```c
struct sched_entity {
  struct load_weight     load;
  struct rb_node         run_node;                 // 用于连接到运行队列的红黑树中
  struct list_head       group_node;
  unsigned int           on_rq;                    // 是否已经在运行队列中
  u64                    exec_start;               // 开始统计运行时间的时间点
  u64                    sum_exec_runtime;         // 总共运行的实际时间
  u64                    vruntime;                 // 虚拟运行时间(用于红黑树的键值对比)
  u64                    prev_sum_exec_runtime;    // 总共运行的虚拟运行时间
  u64                    last_wakeup;
  u64                    avg_overlap;
  u64                    nr_migrations;
  u64                    start_runtime;
  u64                    avg_wakeup;
    ... /*这里省略了很多变量， 只有设置了CONFIG_SCHEDSTATS时才启用*/
};
```

调度器实体结构作为一个名为se的成员变量，嵌入在进程描述符`struct task_struct`中。

2. ##### 虚拟实时

在 `struct sched_entity`中，`vruntime`变量存放的是进程的虚拟运行时间，该运行时间的计算使经过了所有可运行进程总数的标准化的。也就是被加权了的运行时间。这里的虚拟运行时间以ns为单位，所以`vruntime`与定时器节拍不再相关。

在`/kernel/sched/fair.c`中的`update_curr()`函数实现了记账功能。

```c
/*
 * Update the current task's runtime statistics.
 */
static void update_curr(struct cfs_rq *cfs_rq)
{
        struct sched_entity *curr = cfs_rq->curr;
        u64 now = rq_clock_task(rq_of(cfs_rq));
        u64 delta_exec;

        if (unlikely(!curr))
                return;

        /*获取最后一次修改负载后当前任务占用的运行总时间*/
        delta_exec = now - curr->exec_start;
        if (unlikely((s64)delta_exec <= 0))
                return;

        curr->exec_start = now;

        schedstat_set(curr->statistics.exec_max,
                      max(delta_exec, curr->statistics.exec_max));

        curr->sum_exec_runtime += delta_exec;
        schedstat_add(cfs_rq->exec_clock, delta_exec);

        curr->vruntime += calc_delta_fair(delta_exec, curr);
        update_min_vruntime(cfs_rq);

        if (entity_is_task(curr)) {
                struct task_struct *curtask = task_of(curr);

                trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
                cgroup_account_cputime(curtask, delta_exec);
                account_group_exec_runtime(curtask, delta_exec);
        }

        account_cfs_rq_runtime(cfs_rq, delta_exec);
}
```

`update_curr()`计算了当前进程的执行时间，根据当前可运行进程总数对运行时间进行加权计算，最终将上述的权重值与当前运行进程的`vruntime`相加。并将其存放在了变量`delta_exec`中。这个函数由系统定时器周期性调用，无论进程处于可运行态还是堵塞态，根据这种方式，`vruntime`都可以准确测量给定进程的运行时间。并且可以知道谁应该是下一个被运行的进程。

> 由于我使用的内核为5.4版本的代码，并没有书中写的__update_curr()函数，新的内核将这两个函数统一为一个函数了。但其流程没有什么变化。故直接按新内核进行描述

#### 4.5.2 进程选择

为了均衡虚拟运行时间，CFS选择了一个比较简单的规则：当需要选择下一个运行进程时，它会挑选一个具有最小`vruntime`的进程。这一节主要就来讨论如何实现选择`vruntime`最小的进程的。

CFS使用红黑树来组织可运行进程队列，并利用其迅速找到最小的`vruntime`值的进程。在红黑树中，通过键值检索到对应节点的速度与整个树的结点规模成指数比关系。

1. 挑选下一个任务

> 这里书上的代码与5.4代码有很大不同。我这里以5.4代码为例进行了分析。对原书感兴趣可以参考原书相应章节。

假设有一个黑红树存储了系统所有可运行进程，那么节点的键值就是可运行进程的虚拟化时间。CFS期望找到所有进程中`vruntime`最小的那个。实现这一过程的函数为`__pick_next_entity()`，定义在`kernel/sched/fair.c`中。

```c
static struct sched_entity *__pick_next_entity(struct sched_entity *se)
{
        struct rb_node *next = rb_next(&se->run_node);

        if (!next)
                return NULL;

        return rb_entry(next, struct sched_entity, run_node);
}
```

在这里可以看到，5.4内核中调用了`rb_next(&se->run_node)`来选择下一个任务。那再来看看`rb_next`的实现（位于`tools/lib/rbtree.c`）

```c
struct rb_node *rb_next(const struct rb_node *node)
{
        struct rb_node *parent;

        if (RB_EMPTY_NODE(node))
                return NULL;

        /*
         * If we have a right-hand child, go down and then left as far
         * as we can.
         */
        if (node->rb_right) {
                node = node->rb_right;
                while (node->rb_left)
                        node=node->rb_left;
                return (struct rb_node *)node;
        }

        /*
         * No right-hand children. Everything down and left is smaller than us,
         * so any 'next' node must be in the general direction of our parent.
         * Go up the tree; any time the ancestor is a right-hand child of its
         * parent, keep going up. First time it's a left-hand child of its
         * parent, said parent is our 'next' node.
         */
        while ((parent = rb_parent(node)) && node == parent->rb_right)
                node = parent;

        return parent;
}
```

可以看出，`rb_next()`获取的结果可能有三种：

+ 如果节点为空节点，直接返回空
+ 如果节点存在右孩子，那么返回右子树下的最左叶子节点
+ 递归向上访问父节点，返回最后一个节点作为右孩子的父节点。

这样也就实现了下一个节点的选取。也就是说，新的内核中寻找的是`vrantime`排序后相邻的右侧第一个（case2）或者相邻的左侧第一个（case 1）。这里就显示出了与原书的区别。原书中寻找的进程是`vrantime`最小的那个。也就是整颗黑红树的最左侧叶子节点。

2. 向树中加入进程

向书中插入进程发生在进程变更为可运行状态时或者是通过fork第一次创建进程时，是通过`enqueue_entity()`函数实现的。

```c
static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
        bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);
        bool curr = cfs_rq->curr == se;

        /*
         * If we're the current task, we must renormalise before calling
         * update_curr().
         */
        if (renorm && curr)
                se->vruntime += cfs_rq->min_vruntime;

        update_curr(cfs_rq);

        /*
         * Otherwise, renormalise after, such that we're placed at the current
         * moment in time, instead of some random moment in the past. Being
         * placed in the past could significantly boost this task to the
         * fairness detriment of existing tasks.
         */
        if (renorm && !curr)
                se->vruntime += cfs_rq->min_vruntime;

        /*
         * When enqueuing a sched_entity, we must:
         *   - Update loads to have both entity and cfs_rq synced with now.
         *   - Add its load to cfs_rq->runnable_avg
         *   - For group_entity, update its weight to reflect the new share of
         *     its group cfs_rq
         *   - Add its new weight to cfs_rq->load.weight
         */
        update_load_avg(cfs_rq, se, UPDATE_TG | DO_ATTACH);
        update_cfs_group(se);
        enqueue_runnable_load_avg(cfs_rq, se);
        account_entity_enqueue(cfs_rq, se);

        if (flags & ENQUEUE_WAKEUP)
                place_entity(cfs_rq, se, 0);

        check_schedstat_required();
        update_stats_enqueue(cfs_rq, se, flags);
        check_spread(cfs_rq, se);
        if (!curr)
                __enqueue_entity(cfs_rq, se);
        se->on_rq = 1;

        /*
         * When bandwidth control is enabled, cfs might have been removed
         * because of a parent been throttled but cfs->nr_running > 1. Try to
         * add it unconditionnally.
         */
        if (cfs_rq->nr_running == 1 || cfs_bandwidth_used())
                list_add_leaf_cfs_rq(cfs_rq);

        if (cfs_rq->nr_running == 1)
                check_enqueue_throttle(cfs_rq);
}
```

可以看到，函数最主要的调用是` __enqueue_entity`。

```c
/*
 * Enqueue an entity into the rb-tree:
 */
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
        struct rb_node **link = &cfs_rq->tasks_timeline.rb_root.rb_node;
        struct rb_node *parent = NULL;
        struct sched_entity *entry;
        bool leftmost = true;

        /*
         * Find the right place in the rbtree:
         */
        while (*link) {
                parent = *link;
                entry = rb_entry(parent, struct sched_entity, run_node);
                /*
                 * We dont care about collisions. Nodes with
                 * the same key stay together.
                 */
                if (entity_before(se, entry)) {
                        link = &parent->rb_left;
                } else {
                        link = &parent->rb_right;
                        leftmost = false;
                }
        }

        rb_link_node(&se->run_node, parent, link);
        rb_insert_color_cached(&se->run_node,
                               &cfs_rq->tasks_timeline, leftmost);
}
```

在上述函数中，`while`循环遍历寻找适合的匹配键值。由于是平衡二叉树，所以右分支的值永远大于左分支。故而一旦需要向右走子树，`leftmost`就会置为false。推出循环后调用`rb_link_node`使得新插入的进程称为其子节点。最后函数`rb_insert_color_cached`更新树的自平衡属性。

3. 从树中删除进程

删除动作发生在进程堵塞或者终止时，调用函数`dequeue_entity`来进行。

```c
static void
dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
        /*
         * Update run-time statistics of the 'current'.
         */
        update_curr(cfs_rq);

        /*
         * When dequeuing a sched_entity, we must:
         *   - Update loads to have both entity and cfs_rq synced with now.
         *   - Subtract its load from the cfs_rq->runnable_avg.
         *   - Subtract its previous weight from cfs_rq->load.weight.
         *   - For group entity, update its weight to reflect the new share
         *     of its group cfs_rq.
         */
        update_load_avg(cfs_rq, se, UPDATE_TG);
        dequeue_runnable_load_avg(cfs_rq, se);

        update_stats_dequeue(cfs_rq, se, flags);

        clear_buddies(cfs_rq, se);

        if (se != cfs_rq->curr)
                __dequeue_entity(cfs_rq, se);
        se->on_rq = 0;
        account_entity_dequeue(cfs_rq, se);

        /*
         * Normalize after update_curr(); which will also have moved
         * min_vruntime if @se is the one holding it back. But before doing
         * update_min_vruntime() again, which will discount @se's position and
         * can move min_vruntime forward still more.
         */
        if (!(flags & DEQUEUE_SLEEP))
                se->vruntime -= cfs_rq->min_vruntime;

        /* return excess runtime on last dequeue */
        return_cfs_rq_runtime(cfs_rq);

        update_cfs_group(se);

        /*
         * Now advance min_vruntime if @se was the entity holding it back,
         * except when: DEQUEUE_SAVE && !DEQUEUE_MOVE, in this case we'll be
         * put back on, and if we advance min_vruntime, we'll be placed back
         * further than we started -- ie. we'll be penalized.
         */
        if ((flags & (DEQUEUE_SAVE | DEQUEUE_MOVE)) != DEQUEUE_SAVE)
                            update_min_vruntime(cfs_rq);
}
```

同样的，这个工作实际是由`__dequeue_entity`完成的

```c
static void __dequeue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
        rb_erase_cached(&se->run_node, &cfs_rq->tasks_timeline);
}
```

#### 4.5.3 调度器入口

这里后期内核的实现方式与书中过于不同，所以这部分参考的是[linux调度子系统8 - schedule函数](https://zhuanlan.zhihu.com/p/363791563)

进程调度的主要入口点是函数`schedule`，定义在`kernel/sched/core.c`中。其实现异常简单

```c
asmlinkage __visible void __sched schedule(void)
{
        struct task_struct *tsk = current;

        sched_submit_work(tsk);
        do {
                preempt_disable();
                __schedule(false);
                sched_preempt_enable_no_resched();
        } while (need_resched());
        sched_update_worker(tsk);
}
```

`schedule()`只是一个外包装，实际的操作还是`__shedule()`函数。`__shedule()`接受一个bool作为形参，false表示非抢占，自愿调度，true则相反。

在执行整个系统调度的过程中，需要关闭抢占，这也很好理解，内核的抢占本身也是为了执行调度，现在本身就已经在调度了，如果不关抢占，递归地执行进程调度怎么看都是一件没必要的事。当然，在调度过程完成之后，也就是 `__schedule` 返回之后，这个过程中可能会被设置抢占标志，这时候还是需要重新执行调度的。

在调用 `__schedule`时，实际上已经发生了进程切换。假设存在A，B两个进程， A进程调用`__schedule`返回的时候是A进程被调度回来的时候。也就是说，在`__schedule`执行期间，已经发生了从A到B的进程切换。所以`__schedule`函数的执行周期也就会达到几百毫秒甚至更长。

> 因此，实际上，在当前进程中禁止的抢占，而使能抢占的 sched_preempt_enable_no_resched 函数却是在另一个进程上执行的.好在所有的进程都是通过 schedule() 进行进程切换的，也就保证了 disable 和 enable 总是成对的.

 `__schedule`的实现大概分为四个部分：针对当前进程处理、选择下一个需要执行的进程、执行切换以及收尾。

```c
static void __sched notrace __schedule(bool preempt)
{
        struct task_struct *prev, *next;
        unsigned long *switch_count;
        struct rq_flags rf;
        struct rq *rq;
        int cpu;
    
    	//针对当前进程处理
        cpu = smp_processor_id();
        rq = cpu_rq(cpu);
        prev = rq->curr;

        schedule_debug(prev, preempt);

        if (sched_feat(HRTICK))
                hrtick_clear(rq);

    	//disable local interrupt  防止中断的竞争行为
        local_irq_disable();
        rcu_note_context_switch(preempt);

        /*
         * Make sure that signal_pending_state()->signal_pending() below
         * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
         * done by the caller to avoid the race with signal_wake_up().
         *
         * The membarrier system call requires a full memory barrier
         * after coming from user-space, before storing to rq->curr.
         */
        rq_lock(rq, &rf);
        smp_mb__after_spinlock();

        /* Promote REQ to ACT */
        rq->clock_update_flags <<= 1;
        update_rq_clock(rq);

        switch_count = &prev->nivcsw;
        if (!preempt && prev->state) {
                if (signal_pending_state(prev->state, prev)) {
                        prev->state = TASK_RUNNING;
                } else {
                        deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);

                        if (prev->in_iowait) {
                                atomic_inc(&rq->nr_iowait);
                                delayacct_blkio_start();
                        }
                }
                switch_count = &prev->nvcsw;
        }
 
    	//选择下一个需要执行的进程
        next = pick_next_task(rq, prev, &rf);
        clear_tsk_need_resched(prev);
        clear_preempt_need_resched();

    	//执行进程切换
        if (likely(prev != next)) {
                rq->nr_switches++;
                /*
                 * RCU users of rcu_dereference(rq->curr) may not see
                 * changes to task_struct made by pick_next_task().
                 */
                RCU_INIT_POINTER(rq->curr, next);
                /*
                 * The membarrier system call requires each architecture
                 * to have a full memory barrier after updating
                 * rq->curr, before returning to user-space.
                 *
                 * Here are the schemes providing that barrier on the
                 * various architectures:
                 * - mm ? switch_mm() : mmdrop() for x86, s390, sparc, PowerPC.
                 *   switch_mm() rely on membarrier_arch_switch_mm() on PowerPC.
                 * - finish_lock_switch() for weakly-ordered
                 *   architectures where spin_unlock is a full barrier,
                 * - switch_to() for arm64 (weakly-ordered, spin_unlock
                 *   is a RELEASE barrier),
                 */
                ++*switch_count;

                trace_sched_switch(preempt, prev, next);

                /* Also unlocks the rq: */
                rq = context_switch(rq, prev, next, &rf);
        } else {
                rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);
                rq_unlock_irq(rq, &rf);
        }

    	//收尾
        balance_callback(rq);
}
```

##### 针对当前进程的处理

首先要注意的是这个代码`prev = rq->curr;`这里将当前运行的进程赋值为prev，然后后续中一直在执行prev中的进程。这与上文中描述的`__shedule()`的进程切换有关。 在`__shedule()`内部会进行一次进程调度，在调度前，这个prev是当前进程curr，而调度回来后，这个prev就是真正上次执行的进程。

`if (!preempt && prev->state)`这里主要是对自愿调度的处理。`preempt`代表是否资源让出。`prev->state`代表当前进程状态，0对应状态为`TASK_RUNNING`，正常情况下，进程自愿让出CPU会将进程设置为其他状态。

if模块中的代码直接引用知乎大佬的描述

> 进入到 if 判断子代码块中，首先需要先检查该进程是否有未处理信号，如果有，就需要先处理信号，将当前进程的状态设置回 TASK_RUNNING.这种情况下，当前进程会重新参与调度，有很大概率选取到的下一个进程依旧是当前进程，从而不执行实际的进程切换.
>
> 如果没有未处理信号，就调用 deactivate_task() 将进程从队列中移除，实际上就是调用了 dequeue_task 函数，同时将进程的 on_rq 设置为 0，表示当前进程已经不在 CPU 上了(但实际上还在，只是提前设置了标志位)，对于工作队列的内核线程，需要进行一些特殊处理，毕竟这个内核线程关系到中断下半部的执行，这时候需要根据 workqueue 上的负载来确定是否需要唤醒其它的内核线程协助处理worker.
>
> 做完上述的处理之后，需要对进程的 nvcsw 或者 nivcsw 进行更新，这两个标志是调度相关的统计，如果是自愿调度，则选中nvcsw，否则选中 nivcsw，多出的 'i' 表示 involuntarily.这里并没有操作，在后面选中了待运行进程且待运行进程不为 curr 的时候再对其进行加 1 操作。
>
> 最后，再考虑一个问题:如果进程没有设置为非 TASK_RUNNING 状态的情况下，直接调用 schedule() 函数，会怎么样?
>
> 当然，这和上面提到的关抢占然后调度的问题一样属于不正常的情况，对于这种情况的处理其实和检测到有未处理信号一样，调度器会照样选择下一个执行的进程，而且大概率会是当前进程.
>
> 理论上来说，cfs 调度器上运行的就是 vruntime 最小的进程，如果是这样，下一个被选择的进程几乎一定是当前进程.但是具体的实现来理论还是有些差别，其中包括:
>
> - 进程有一个最小执行时间的限制，可能当前进程的 vruntime 大于 leftmost，但是依旧在运行.
> - 在周期性的调度检查中，并不是 curr->vruntime 大于 leftmost->vruntime 就立马调度，而是需要这两者的差值大于 curr 的 idle_time.可能当前进程的 vruntime 大于 leftmost，但是依旧在运行.
> - 由于检查调度的粒度问题，进程已经超出理论应该运行的时间，但是没有出现检查是否需要调度的点.
>
> 这三种情况都可能导致 leftmost->vruntime 小于 curr->vruntime，在这几种情况下执行调度，调度器就会选择到 leftmost，而不是继续执行当前进程了.

##### 选择下一个需要执行的进程

选择下一个进程的接口为`pick_next_task`函数，也是内核调度的核心接口，针对了所有的调度器类。其代码如下：

```c
 * Pick up the highest-prio task:
 */
static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
        const struct sched_class *class;
        struct task_struct *p;

        /*
         * Optimization: we know that if all tasks are in the fair class we can
         * call that function directly, but only if the @prev task wasn't of a
         * higher scheduling class, because otherwise those loose the
         * opportunity to pull in more work from other CPUs.
         */
        if (likely((prev->sched_class == &idle_sched_class ||
                    prev->sched_class == &fair_sched_class) &&
                   rq->nr_running == rq->cfs.h_nr_running)) {                      //注1

                p = fair_sched_class.pick_next_task(rq, prev, rf);                 //注2
                if (unlikely(p == RETRY_TASK))
                        goto restart;

                /* Assumes fair_sched_class->next == idle_sched_class */
                if (unlikely(!p))
                        p = idle_sched_class.pick_next_task(rq, prev, rf);  

                return p;
        }

restart:
#ifdef CONFIG_SMP
        /*
         * We must do the balancing pass before put_next_task(), such
         * that when we release the rq->lock the task is in the same
         * state as before we took rq->lock.
         *
         * We can terminate the balance pass as soon as we know there is
         * a runnable task of @class priority or higher.
         */
        for_class_range(class, prev->sched_class, &idle_sched_class) {          //注3
                if (class->balance(rq, prev, rf))
                        break;
        }
#endif

        put_prev_task(rq, prev);

        for_each_class(class) {
                p = class->pick_next_task(rq, NULL, NULL);
                if (p)
                         return p;
        }

        /* The idle class should always have a runnable task: */
        BUG();
}
```

注1:在 pick_next_task 中，其实并不是想象中的直接按照调度器的优先级对所有调度器类进行遍历，而是假设下一个运行的进程属于 cfs 调度器类，毕竟，系统中绝大多数的进程都是由 cfs 调度器进行管理，这样做可以从整体上提高执行效率.

判断一个下一个进程是不是属于 cfs 调度器的方式是当前进程是不是属于 cfs 调度器类或者 idle 调度器类管理的进程，同时满足 rq->nr_running == rq->cfs.h_nr_running， 其中 rq->nr_running 表示当前 runqueue 上所有进程的数量，包括组调度中的递归统计，而 rq 的 root cfs_rq 的 h_nr_running 表示 root cfs_rq 上所有进程的数量，如果两者相等，自然就没有其它调度器的进程就绪.

注2**:如果确定了下一个进程会在 cfs_rq 中选，就调用 cfs 调度器类的 pick_next_task 回调函数，这个我们在后面详细讨论**.

注3:如果确定下一个进程不在 cfs_rq 中选，就需要依据优先级对所有的调度器类进行遍历，找到一个进程之后就返回该进程，由此可以看出，在系统的调度行为中，不同的调度器类拥有绝对的优先级区分，高优先级的调度器类并不会与低优先级的调度器类共享 CPU，而是独占(会有一些特殊情况，在具体实现中，实时进程会好心地让出那么一点 CPU 时间给非实时进程，这个时间可配置).

遍历调度器类的接口为 for_each_class，在多核架构中，最高优先级调度器类为 stop_sched_class，其次是 dl_sched_class，单核架构中，最高优先级调度器类为 dl_sched_class，接下来依次是 rt_sched_class->fair_sched_class->idle_sched_class. stop_sched_class 主要用来停止 CPU 的时候使用的，比如热插拔.其遍历过程也是通过调用对应调度器类的 pick_next_task 函数，如果该调度器有进程就绪就返回进程，否则返回 NULL，继续遍历下一个调度器类，至于其它调度器类的实现，暂不讨论.

在`pick_next_task_fair`里面主要就调用了前面说的`pick_next_entity()`也就回到了前面所描述的进程选择的位置。

##### 执行进程切换

经过千辛万苦，终于选到了合适的进程，在选取了合适的待运行进程之后，就进入下一个环节:进程的切换.

如果是抢占调度，就需要先清除抢占标志。

然后，判断选出来的下一个进程是否是当前运行的进程，经过上面的源码分析，其实资源调度和抢占调度两种方式选到的待运行进程都可能是当前进程，这种情况下就不需要做什么处理，直接清理调度设置准备退出。

大部分情况下待运行进程都不会是 curr，这时候就需要进入到真正的切换流程，在切换之前，执行一些必要的计数统计和更新：

```cpp
rq->nr_switches++;
rq->curr = next;
++*switch_count;   // 这个变量是 curr->nivcsw 或者 curr->nvcsw
```

然后执行切换函数：context_switch。

```cpp
schedule->__schedule->context_switch:

static __always_inline struct rq *
context_switch(struct rq *rq， struct task_struct *prev，
           struct task_struct *next， struct rq_flags *rf)
{
    struct mm_struct *mm， *oldmm;

    // 切换的准备工作，包括更新统计信息、设置 next->on_cpu 为 1 等。 
    prepare_task_switch(rq， prev， next);

    // 获取即将切换到的进程的 mm。
    mm = next->mm;
    // 当前的 active_mm
    oldmm = prev->active_mm;

    // mm = NULL 表示下一个进程还是一个内核线程
    if (!mm) {
        // 如果是内核线程，依旧不需要切换 mm，依旧保存 active_mm
        next->active_mm = oldmm;
        // 相当于添加引用计数：atomic_inc(&mm->mm_count);
        mmgrab(oldmm);
        enter_lazy_tlb(oldmm， next);
    } else
        switch_mm_irqs_off(oldmm， mm， next);

    if (!prev->mm) {
        prev->active_mm = NULL;
        rq->prev_mm = oldmm;
    }

    ...

    rq_unpin_lock(rq， rf);
    spin_release(&rq->lock.dep_map， 1， _THIS_IP_);

    switch_to(prev， next， prev);
    barrier();

    return finish_task_switch(prev);
}
```

从硬件上来说，CPU 为了加速数据的访问过程，大量地使用了缓存技术，包括指令、数据和 TLB 缓存，事实证明，缓存的使用给系统运行效率带来了巨大的提升。在多核架构中，缓存是 percpu 的，通常只有 L3 缓存的 global 的，因此，对于进程的切换而言，通常操作的都是本地 CPU 的缓存。

在 context_switch 中，一个比较重要的部分是用户空间内存映射的处理，即 mm 结构，整个用户空间的内存布局都是由一个 struct mm 的结构保存的，对于内核线程而言，并没有所谓的用户空间的概念，其 mm 为 NULL。

内核是统一的，并不像用户空间一样每个进程享有独立的空间，因此，从缓存的角度来看，对于内核部分和用户空间部分的处理是完全不一样的。对于指令和数据 cache 而言，会随着程序的执行逐渐被替换，用户空间的 cache 是必须要被 flush 的，而内核空间中的某些 cache 依旧可以利用在下一个进程。

重点在于 TLB 的缓存处理，TLB 中缓存了页表对应的虚拟地址到物理地址的映射，有了这一层缓存，对某片内存的重复访问只需要从缓存中取，而不需要重新执行翻译过程，当执行进程切换时，尽管每个进程虚拟空间都是一致的，但是其对应的物理地址通常是不相等的，因此需要将上个进程的 TLB 缓存清除，不然会影响下一个进程的执行。

但是实际情况却有一些优化空间，比如对于用户进程之间的互相切换，其用户空间的 TLB 缓存自然是要刷新，但是内核空间可以保留，因为内核中是共用的。同时，linux 中的内核线程和用户空间没有关系，假设存在这样的情况：用户进程 A 切换为内核线程 B，内核线程 B 运行完之后又切换回 A，这时候从 A 切换到 B 的时候如果清除了 TLB 缓存，在 B 切换回 A 的时候，TLB 又需要重新填充，实际上这种情况可以在切换到内核线程时保留用户空间的 TLB 缓存，如果又切换回 A 的时候就正好可以直接使用，提升了效率。

如果此时从 B 切换到其它内核线程 C，TLB 缓存依旧可以保留，直到切换到下一个用户进程，如果这个进程不是原本的 A，这时候才会把 TLB 清除，这就是我们经常听到的 "惰性 TLB"，这种 lazy operation 可以在很多地方可以看到，比如 fork 的执行，动态库的绑定，其核心思想就是直到资源在真正需要的时候才进行操作。

上面的源代码就是基于上述的逻辑，**获取 next->mm** ，这是待运行进程的 mm，**同时获取 oldmm = prev->active_mm**，如果当前进程是用户进程，oldmm 等于 NULL，因为切换到其它用户进程时 mm 是肯定需要替换的，而如果当前进程是内核线程，oldmm 就是上个进程保留的 mm，当然，上个进程也可能是内核线程，这里保存的就是上一个用户进程的 mm。

判断如果待运行进程的 mm 为空，即为内核线程，那么就不需要切换 mm 结构，相对应的 TLB 也不需要刷新，而如果待运行进程是用户进程，分两种情况：第一个是缓存的 mm 正好是下一个待运行进程，也就不需要做什么事，另一种情况就是当前进程不是缓存的 mm，那么就需要替换 mm，然后刷新 TLB，同时只需要刷新用户空间的 TLB，而不需要刷新内核 TLB。

##### switch_to

在处理完上下文环境之后，就来到了真正的进程切换操作了，具体操作自然是和硬件强相关的，被定义在 arch/ 下，同时操作的代码都是通过汇编实现的。

对于进程的实际切换，有两个点需要弄清楚：

第一点就是，**switch_to 是进程的一个断点**，也就是在 switch_to 中，会完成进程的切换，在 switch_to 之后的代码，实际上是在进程切换回来之后执行的。比如，从上面的代码来看，switch_to 后调用了 finish_task_switch，实际上，进程 A 在调用 switch_to 中切换到了进程 B，而紧接着执行的是 B 的 finish_task_switch，因此，在 B 中的 finish_task_switch 中做的一些调用清理工作，其实是针对进程 A 的。

第二点也就是基于第一点的延伸，switch_to 的调用为 switch_to(prev， next， prev)，函数需要传入三个参数，按理说，只需要两个参数，一个是当前进程，一个是 next，也就是待运行进程，为什么还需要第三个参数？

switch_to 中完成了进程的切换，switch_to 之后的代码实际上是进程切换回来执行的，那么，我们何从知道当前进程是从哪个进程切换过来的呢?所以，多出来的一个参数就是上次运行的进程，实际上是 last，比如由 A 切换到 B，对于 B 而言，多出来的一个参数保存的就是 A 的 task_struct.

在 arm 的实现中，switch_to 将会调用 __swtich_to：

```cpp
#define switch_to(prev，next，last)                   \
do {                                    \
    __complete_pending_tlbi();                  \
    last = __switch_to(prev，task_thread_info(prev)， task_thread_info(next));    \
} while (0)
```

通过 switch_to 的定义更能体现切换的过程，切换之后将会返回上次运行的进程。而 __switch_to 函数则被定义在 arch/arm/kernel/entry-armv.S 中。这部分汇编代码的实现为：

- 将当前进程大部分通用寄存器保存到栈上，需要注意的是，这里并不是压栈，因为每个进程的内核栈的底部保存的是当前进程的 thread_info，在 arm 中，保存寄存器的地址为：thread_info->cpu_context，保存的通用寄存器的值为 r4 ~ r14，其中 lr 是 __switch_to 函数的返回值。
- 保存或者替换其它的寄存器值，比如线程本地存储 tls、浮点寄存器或者其它，这一部分是硬件强相关的，需要看硬件是否提供对应的功能。
- r0 寄存器保持原样，传入的 r0 是第一个参数，表示当前进程，切换到另一个进程 B 之后，其返回值就是当前进程 A，按照 aapcs 规范，函数调用时返回值保存在 r0 中，也就是上一个运行的进程。
- 将 next 进程对应内核栈上保存的寄存器值恢复到寄存器中，也就完成了切换，r4~r13 是原样恢复，而 pc 指针则使用保存的 lr 寄存器，也就是新进程会从 __switch_to 函数返回处开始执行。

#### 4.5.4 睡眠和唤醒

休眠（被堵塞）的进程处于一个特殊的不可执行状态。休眠的原因有很多，但是肯定都是为了等待一些事件的发生。无论哪种情况，内核的操作都相同：进程将自己的`task_struct`中的`state`设置为休眠状态，从可执行的红黑树中移除，放入等待队列，然后调用`schedule()`选择一个其他进程。而唤醒进程则刚好相反。

复习一下第三章的知识，休眠中存在两种进程相关的状态：`TASK_INTERRUPTABLE`以及`TASK_UNINTERRUPTIBLE`。他们的区别为后者会忽略信号，而前者在接受到信号后会唤醒并且进行响应。这两种状态的进程会位于同一个等待队列上，等待某些事件，不能够运行。

1. 等待队列


按前面的描述我们知道，休眠通过等待队列来进行处理。这个队列在内核中的表示为`wake_queue_head_t`。等待队列的创建有两种形式：一是通过`DECLARE_WAITQUEUE()`静态创建，二是通过`init_waitqueue_head()`动态创建。进程将通过执行下面几个步骤将自己加入等待队列中：

+ 调用宏`DEFINE_WAIT`创建一个等待队列项
+ 调用`add_wait_queue()`把自己加入到队列中。该队列会在满足进程等待条件时将其唤醒。（需要代码对等待队列执行`wake_up()`）
+ 调用`prepare_to_wait()`方法将进程状态变更为`TASK_INTERRUPTABLE`或`TASK_UNINTERRUPTIBLE`
+ 如果状态被设置为`TASK_INTERRUPTABLE`，则信号唤醒进程（伪唤醒），因此检查并处理信号
+ 进程再次被唤醒的时候，会再次检查条件是否为真。为真便退出循环；否则再次调用`schedule()`并一直重复检查。
+ 条件满足后进程将自身设置为`TASK_RUNNING`并调用`finish_wait()`将自己移出等待队列

如果进程在开始休眠之前就已经达成条件，则循环会退出，进程不会存在错误的进入休眠的倾向。

> 需要注意，内核代码在循环体内通常要做一些其他任务。比如需要在调用`schedule()`之前释放锁，在之后进行重新获取，或者响应其他事件。

2. 唤醒

唤醒操作通过`wake_up()`进行。会唤醒指定等待队列的所有进程。它调用了函数`try_to_wake_up()`。该函数将进程设置为`TASK_RUNNING`状态。并调用`enqueue_task()`将进程放入红黑树中。若被唤醒的进程比当前进程优先级高，则要设置`need_resched`标志。促使等待条件达成的代码要负责随后调用`wake_up()`函数。

> 由于存在虚假的唤醒，所以进程被唤醒也并不是因为它等待的条件达成了。所以才需要用循环处理来保证它等待的条件真正达成。

### 4.7 实时调度策略

Linux提供了两种实时调度策略：`SCHED_FIFO`以及`SCHED_RR`。而普通非实时的调度策略是`SCHED_NORMAL`。协助调度类的框架，这些实时调度不完全被CFS来管理。而是被定义在`kernel/sched_rt.c`中的一个特殊调度器来管理。

`SCHED_FIFO`使用先进先出的策略，任何处于`SCHED_FIFO`级的进程都会比`SCHED_NORMAL`级的进程更先得到调度。这个进程不受时间片限制，可以一直执行下去。只有更高优先级的`SCHED_FIFO`以及`SCHED_RR`进程到来才能抢占。如果存在两个相同级别的`SCHED_FIFO`进程，他们只会在另一方自愿让出内核时才能执行。

`SCHED_RR`与前者大体相同。可以理解为带时间片的`SCHED_FIFO`。只是`SCHED_RR`的进程在耗尽时间片之后就不能继续执行了。

Linux给实时调度算法提供了一种软实时的工作方式。所谓的软实时，是指内核调度进程尽力使进程在它的限定时间到来之前运行，硬实时系统则是保证在一定条件下，可以满足任何调度的要求。

> 在默认情况下，nice值从-20到+19对应的是从100到139的实时优先级范围。

### 4.8 与调度有关的系统调用

|         系统调用         |        描   述         |
| :----------------------: | :--------------------: |
|          nice()          |    设置系统的nice值    |
|   sched_setscheduler()   |   设置进程的调度策略   |
|   sched_getscheduler()   |   获取进程的调度策略   |
|     sched_setparam()     |  设置进程的实时优先级  |
|     sched_getparam()     |  获取进程的实时优先级  |
| sched_get_priority_max() | 获取实时优先级的最大值 |
| sched_get_priority_min() | 获取实时优先级的最小值 |
| sched_rr_get_interval()  |    获取进程时间片值    |
|   sched_setaffinity()    | 设置进程处理器的亲和力 |
|   sched_getaffinity()    | 获取进程处理器的亲和力 |
|      sched_yield()       |     暂时让出处理器     |


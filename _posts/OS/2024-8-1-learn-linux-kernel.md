---
layout: post
title: Linux内核学习笔记1-进程与调度
subtitle: 内核学习笔记；封面：X：@fleia0124
banner:
  image: /assets/images/banners/10.jpg
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: OS linux
---

## 单内核和微内核

&emsp;&emsp;单内核是指将内核作为一个整体的大过程实现，运行在一个单独的地址空间上。在磁盘上以一个单独的二进制文件存储。好处是简单和性能高效。<br>
&emsp;&emsp;微内核是指将内核的功能分成一个个小的服务器，各自保持独立运行在自己的内存空间里。所以，就不能像单内核那样直接调用函数，而是采用进程间通讯IPC机制互通消息。这样的设计保证了一个崩溃了不会殃及全局，并且，内核的组件可以动态改变。但是同样的，IPC占用时间大于函数调用，效率没有单内核高。

> 任何改变都要能通过简洁的设计和可靠的实现来解决现实中确实存在的问题。

## 进程和线程

&emsp;&emsp;进程是指处于执行期的程序以及其它资源的总称。线程是在进程中活动的对象。

- 每个线程都拥有自己独立的程序计数器、进程栈和一组进程寄存器
- 内核的调度对象是线程而不是进程
- 线程之间可以共享虚拟内存，但是每一个都拥有自己的虚拟处理器
- 使用fork创建进程，从内核返回两次，一次是这个函数创建完成，返回调用fork的进程，另一次是返回新进程。
- 进程又称作task

### 进程描述符和任务结构

&emsp;&emsp;内核把进程的列表放在名为任务队列的双向循环链表中，每一项都是类型为`task_struct`，成为进程描述符的结构，该结构能够完整的描述一个正在执行的程序，包括但不限于打开的文件、地址空间等等。该描述符存在内核栈中。Linux2.6以后的版本会在`task_struct`栈底追加`thread_info`

&emsp;&emsp;进程状态有运行、停止、可中断（正在被某种原因阻塞，一旦满足要求就会重新运行）、不可中断（信号阻塞，并且不会被外界打断这种状态，开霸体了）、TRACE跟踪状态。

&emsp;&emsp;进程在自己的内存空间运行，当触发系统调用的时候，称陷入内核空间，内核处于进程上下文中，当然，此时也会听调度器的。

&emsp;&emsp;系统中所有的进程都是PID=1的init进程后代，进程关系存放在`task_struct`中，包含了指向其父进程`task_struct`的指针，也包含了一个`children`子进程链表。由此，如果你愿意，可以一直上溯到1号进程。

&emsp;&emsp;进程创建是将父进程所有资源复制到新的进程空间中，目前使用的方法是`copy-on-write`，就是在新进程需要写入数据之前，和父进程共享资源，当然，是只读的。当需要写入时，再进行资源的复制。

> 这里我不明白，那如果父进程要写入呢？为什么要复制父进程的资源呢？

### `current`宏

&emsp;&emsp;`current`宏：

```c
#define current get_current()

static __always_inline struct task_struct *get_current(void)
{
	if (IS_ENABLED(CONFIG_USE_X86_SEG_SUPPORT))
		return this_cpu_read_const(const_pcpu_hot.current_task);

	return this_cpu_read_stable(pcpu_hot.current_task);
}

#define this_cpu_read_stable(pcp)			__pcpu_size_call_return(this_cpu_read_stable_, pcp)
```

总之就是返回目前正在运行的，也就是现在这个进程的`task_struct`，我觉得类似于cpp的this。在arch/x86/kernel/process_64.c文件中的__switch_to函数中有如下代码：

```c
this_cpu_write(current_task, next_p);  
```

在x86体系结构中，使用了current_task这个每CPU变量，来存储当前正在使用cpu的进程的struct task_struct。 由于采用了每cpu变量current_task来保存当前运行进程的task_struct，所以在进程切换时，就需要更新该变量。

### 进程的结束

&emsp;&emsp;进程的结束包含下列操作：删除内核所有正在运行的定时器、释放空间、释放使用的文件资源（这里有auto_ptr的思想）、调用函数通知父进程，给子进程找养父，养父可以是进程组里面的其他进程，也可以是init进程。最后设置当前进程状态为EXIT_ZOMBIE，即僵尸进程。调度器切换上下文，永不返回，成为僵尸进程。

&emsp;&emsp;此时，所有资源释放已经完毕，唯一占用的空间是描述进程的结构体、内核栈。此时它存在的意义是等着父进程处理。等到父进程通知内核这是无关信息后，所有内存都将被释放。最后的清理称之为删除进程描述符。上一段的删除操作是释放进程资源。

## 进程调度

&emsp;&emsp;多任务系统分为两类：非抢占式和抢占式。抢占式系统中的进程在抢占后使用CPU的时间是被设定好的，成为进程的时间片。非抢占式系统则是需要“物质生活极大丰富（不是），道德水平极大提高”的前提————每个程序都需要主动挂起自己，称之为让步，如果谁不让步，就会一直占着CPU不放，如你所想，一个进入死锁的程序又拒不让步，那么电脑将会崩溃。显然，一个非定制的操作系统这样设计并不明智。不过历史上还真有这样的系统————windows3.1以及MacOS9。

&emsp;&emsp;进程可以被分为两类：IO密集型和计算密集型。前者许多时间用于等待，后者许多时间用于处理。前者的典例比如图形操作界面，他们总是在等待键盘鼠标的输入，经过计算后也总是进入阻塞，再次等待；后者的典例比如正在进行计算的MATLAB程序，甚至不需要什么IO，全部数据已经加载进入内存，需要的只是大规模的计算。当然，这两者也并不总是绝对的，比如带有图形操作界面的MATLAB程序（笑）

### 进程的优先级

&emsp;&emsp;一个朴素的想法就是根据进程的价值和对处理器的需求进行评判。为每一个进程设置优先级，高的先执行，统计轮换执行，高的进程时间片也长，系统总是选择优先级高并且时间片没用完的进程执行。

&emsp;&emsp;一个正在使用的优先级就是nice值。范围是-20~+19，nice值越大，优先级越低；也就是说-20是优先级最高的进程。0是默认的nice值。Mac系统中，nice值决定分配时间片的绝对值大小；Linux系统中，决定了分配时间片的比例（好像也没啥区别，就是写法不一样）。

&emsp;&emsp;另一种优先级系统是实时优先级。范围0~99，越大优先级越高。任何实时进程的优先级都高于普通进程————这意味着这个优先级系统和nice优先级系统泾渭分明。我运行了`ps -eo state,uid,pid,ppid,rtprio`命令，rtprio列下全是‘-’，代表当前没有实时进程。

&emsp;&emsp;可以通过命令指定某个进程为实时进程，以获得更高的优先级。

&emsp;&emsp;显然，调度器必须有一个默认的时间片，太大了让人感觉不是实时并行的，太短了又把时间浪费在切换上下文上。值得记录的是，对于计算密集类的进程，长的时间片可以提高其缓存命中率。一般默认的时间片为10ms。不过，linux系统使用的是比例调度，这样使用时间其实就和系统负载相关了，高nice值能够获得更多的处理器使用比。

&emsp;&emsp;朴素的抢占式：当进程进入可运行状态，立即投入运行。在linux系统中，抢占时机取决于就绪的进程消耗了多少处理器时间比，如果小于正在运行的进程，那么发生抢占。下面是一个实际例子：系统正在运行一个文本编辑器和一个视频编码器，对于前者，我们希望我们键入之后立即反应，对于后者，我们并不关心它是不是立刻运行的，当然，我们希望能够尽早完成编码任务。在百分比的使用调度下，假设二者的nice值都是0，那么二者将均分处理器使用时间。但是，编辑器在处理完输入后，进入等待，也即非可运行状态，从而被编码器抢占；但是当编辑器收到了键入被唤醒之后，因为其使用时间远小于50%，编码器远大于50%，编辑器没有消耗系统承诺的50%时间片，自然触发了抢占。

> 评：真是天才的想法，谁想出来的！不过这样，我们希望IO密集型能够自己挂起，如果不挂起，属于是损人不利己：既让计算密集型没有抢到时间，自己也被等待时间耽误了，下一次触发输入的时候还不能以高优先级抢占。不知道linux是怎么处理这一情况的？

&emsp;&emsp;除了这样的规则，还应当设置最小时间片，否则当进程数量十分多的时候，按照百分比分配那么每个进程分到的时间片都很小，资源全部浪费在切换上下文上面了。所以要设立最小时间片。

&emsp;&emsp;调度延迟：每个进程都至少运行一次的时间间隔。在达不到最小时间片的情况下，这个调度延迟是动态变化的。当每个进程的占用时间到最小时间片之后，调度延迟=进程数量*最小时间片

### CFS调度器的实现

#### 总体描述

&emsp;&emsp;首先，存在一个调度对象队列`cfs_rq`，其中每个对象进程都具有自己的虚拟时钟。某个进程抢占CPU时间n后，其虚拟运行时间会增长n而其它进程的不变。所谓虚拟运行时间，是指被优先级加权之后的时间，例如A进程实际运行了1ms，B进程实际运行了3ms，但是经过优先级加权公式计算之后，它们的实际运行时间都是3ms。所以，CFS调度器只需要保证虚拟运行时间相等就可以了；所以调度策略就是找到虚拟运行时间最小的进程。注意这里，如果有许多进程虚拟运行时间都不达标，CFS也不需要再考虑优先级的问题，因为虚拟运行时间已经是加权之后的了。

&emsp;&emsp;一个进程所能获取的CPU时间比例是：

$$A进程获取的时间比例=\frac{A的权重}{所有进程的权重之和}$$

&emsp;&emsp;分配时间是按照调度区来分配的，所以实际获得的CPU时间是：

$$进程获取的CPU时间=调度区\times\frac{进程权重}{所有的进程权重之和}$$

&emsp;&emsp;虚拟运行时间的计算公式是：

$$ 虚拟运行时间 = 实际运行时间 \times \frac{NICE\_0\_LOAD}{进程权重} $$

$$实际运行时间=调度周期\times\frac{进程权重}{所有的进程权重之和}$$

代入上式，有：

$$虚拟运行时间=调度周期\times\frac{1024}{所有进程的权重之和}$$

得出结论：在 相同的 调度周期 中 , 所有 运行在该 CPU 上的进程 的 " 虚拟运行时间 " 是相同的 ,在 CFS 调度器 对 进程 进行调度运行时 , 找到 " 虚拟运行时间 " 最小的进程 运行即可。<br>

#### nice值对于权重的影响：

```c
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
}; 
```

&emsp;&emsp;计算方式是:$weight=\frac{1024}{1.25^{nice}}$ .公式中的1.25取值依据是：进程每降低一个nice值，将多获得10% cpu的时间。公式中以1024权重为基准值计算得来，1024权重对应nice值为0，其权重被称为NICE_0_LOAD。默认情况下，大部分进程的权重基本都是NICE_0_LOAD。<br>

值得一提的是，考察这个公式：

$$ 虚拟运行时间 = 实际运行时间 \times \frac{NICE\_0\_LOAD}{进程权重} $$

其中分子是1024，这里涉及到除法，为了提升除法的精度，内核采用先放大再缩小的方法：

$$
\begin{aligned}
  虚拟运行时间 &= 实际运行时间 \times \frac{NICE\_0\_LOAD}{进程权重}   \\
  &= (实际运行时间 \times \frac{1024 * 2 ^{32}}{进程权重}) >> 32 \\
  &= (实际运行时间 \times 1024 \times inv\_weight) >> 32 \\
\end{aligned}
$$

其中，

$$inv\_weight = \frac{2^{32}}{进程权重}$$

这个值同样已经写死了：

```c
const u32 sched_prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
}; 
```

这是将实际时间转化为虚拟时间的函数：实现了 $虚拟运行时间=实际运行时间 \times 1024 \times inv\_weight$ .其中传入的weight是1024.这里我一开始总是没看懂以为是真正的weight。

```c
/*
 * delta_exec * weight / lw.weight
 *   OR
 * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHIFT
 *
 * Either weight := NICE_0_LOAD and lw \e sched_prio_to_wmult[], in which case
 * we're guaranteed shift stays positive because inv_weight is guaranteed to
 * fit 32 bits, and NICE_0_LOAD gives another 10 bits; therefore shift >= 22.
 *
 * Or, weight =< lw.weight (because lw.weight is the runqueue weight), thus
 * weight/lw.weight <= 1, and therefore our shift will also be positive.
 */
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
	u64 fact = scale_load_down(weight);
	u32 fact_hi = (u32)(fact >> 32);
	int shift = WMULT_SHIFT;
	int fs;

	__update_inv_weight(lw);

	if (unlikely(fact_hi)) {
		fs = fls(fact_hi);
		shift -= fs;
		fact >>= fs;
	}

	fact = mul_u32_u32(fact, lw->inv_weight);

	fact_hi = (u32)(fact >> 32);
	if (fact_hi) {
		fs = fls(fact_hi);
		shift -= fs;
		fact >>= fs;
	}

	return mul_u64_u32_shr(delta_exec, fact, shift);
}

/*
 * delta /= w
 */
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
  // 这里竟然写了unlikely，他们认为用户大概率不会手动调这个
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);

	return delta;
}
```

从`calc_delta_fair`函数可以看出，当NICE值为0的时候，虚拟运行时间就等于实际运行时间，不需要计算。所以calc_delta_fair函数返回的就是进程的虚拟运行时间。

#### 时间记账

&emsp;&emsp;分配一个时间片给到当前的进程，每当CPU时钟过了一个时钟周期，时间片都会减一，减到0为止，此进程可以被其它进程抢占。程序中实现了一个时间记录结构体，记录时间片相关的数据，该结构体会被嵌入进程描述符`task_struct`中。

```c
/*
 * Update the current task's runtime statistics.
 */

// cfs_rq：CFS-related fields in a runqueue
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq->curr;
	s64 delta_exec;

  // 异常处理
	if (unlikely(!curr))
		return;

  // 此函数计算两次调用之间的间隔时间，看下面
	delta_exec = update_curr_se(rq_of(cfs_rq), curr);
	if (unlikely(delta_exec <= 0))
		return;

  // 计算绝对时间
	curr->vruntime += calc_delta_fair(delta_exec, curr);
  // 更新deadline，下面有解析
	update_deadline(cfs_rq, curr);
  // 维护进程列表里面的最小运行时间
	update_min_vruntime(cfs_rq);

	if (entity_is_task(curr))
		update_curr_task(task_of(curr), delta_exec);

	account_cfs_rq_runtime(cfs_rq, delta_exec);
}
```

下面是更新delta时间的函数，写的很清楚：

```c
static s64 update_curr_se(struct rq *rq, struct sched_entity *curr)
{
  // 获取当前时间
	u64 now = rq_clock_task(rq);
	s64 delta_exec;

  // 当前 - 之前调用的时间
	delta_exec = now - curr->exec_start;
	if (unlikely(delta_exec <= 0))
		return delta_exec;

  // 更新“之前调用的时间”
	curr->exec_start = now;
  // 更新总用时
	curr->sum_exec_runtime += delta_exec;

	if (schedstat_enabled()) {
		struct sched_statistics *stats;

		stats = __schedstats_from_se(curr);
		__schedstat_set(stats->exec_max,
				max(delta_exec, stats->exec_max));
	}

	return delta_exec;
}
```

下面是核心的，计算终结时间的函数：

```c
/*
 * XXX: strictly: vd_i += N*r_i/w_i such that: vd_i > ve_i
 * this is probably good enough.  // 来自老一辈开发者的自信
 */
// vd_i:虚拟终结时间，ve_i：虚拟结束时间，w_i：权重，由nice值决定
// 这个函数运行完了之后，会更新cfs_rq里面的deadline和slice
static void update_deadline(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
  // 如果当前的运行时间已经超过了该死亡的时间，那么啥也不干（已经超时了）
	if ((s64)(se->vruntime - se->deadline) < 0)
		return;

	/*
	 * For EEVDF the virtual time slope is determined by w_i (iow.
	 * nice) while the request time r_i is determined by
	 * sysctl_sched_base_slice.
	 */
  // 分配一个新的时间片
	se->slice = sysctl_sched_base_slice;

	/*
	 * EEVDF: vd_i = ve_i + r_i / w_i
	 */
  // 更新它该死的时间
	se->deadline = se->vruntime + calc_delta_fair(se->slice, se);

	/*
	 * The task has consumed its request, reschedule.
	 */
  // 当前是否是多任务队列，如果是多任务的，那么调用上下文调度器重新调度
	if (cfs_rq->nr_running > 1) {
		resched_curr(rq_of(cfs_rq));
		clear_buddies(cfs_rq, se);
	}
}

```

更新最小

```c
// 更新最小运行时间的进程，决定了谁该下一个使用CPU，其实就是个打擂台哈哈，我以为是什么非常高级的数据结构呢
static u64 __update_min_vruntime(struct cfs_rq *cfs_rq, u64 vruntime)
{
  // 获取当前的最小值
	u64 min_vruntime = cfs_rq->min_vruntime;
	/*
	 * open coded max_vruntime() to allow updating avg_vruntime
	 */
  // 相减以判正负
	s64 delta = (s64)(vruntime - min_vruntime);
  // 大于0，也就是当前运行时间大于最小运行时间
	if (delta > 0) {
    // 更新平均虚拟运行时间
		avg_vruntime_update(cfs_rq, delta);
    // 更新最小值
		min_vruntime = vruntime;
	}
	return min_vruntime;
}
```

## 后记

&emsp;&emsp;这次我首先在代码阅读工具上弄了一天时间，因为vscode是无法解析如此巨量的内核代码的；然后听说vim+ctag可以方便的进行阅读，可是尝试之后发现非常不智能，当我需要跳转到函数的定义的时候，它总是给我搞出来一堆引用的地方，我还得一个个看哪一个才是定义。最后尝试了clang，发现确实好用。

&emsp;&emsp;学习内核代码是一件需要正确方法的事。古人说，吾尝终日而思矣，不如须臾之所学也。又有云，学而不思则罔，思而不学则殆。对于这种新知识尤其是如此。在这件事里，思就是自己看代码思考，学就是阅读别人的学习成果。一开始我以为我能自己看懂调度器的算法，磕了一天虽然有所理解，但是仍然似懂非懂，可能看明白了一些函数，但是没法串联起来说个明白。后来我改变方法，搜索了其它的博客，只需须臾阅读，便可知其原理。然后再反过来阅读代码，方才是学会。

## 参考博客

[CFS调度器基本原理](https://www.cnblogs.com/linhaostudy/p/10298511.html)
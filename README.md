# Design a scheduler class Utilize the Multilevel Queue Algorithm

New Scheduler class utilize the Multilevel Queue (MLQ) algorithm. This Scheduler must support various scheduler-related system call in Linux.

## Implementation

- Scheduler class name: `sched_mlq_class`
- Scheduler policy: `SCHED_MLQ`
- The Priority expected to be higher than `sched_fair_class` but lower than `sched_rt_class` .

---

`SCHED_MLQ` 中有1 (highest priority), 2, 3 (lowest priority)，每個都有連接到一個 task queue而且高的要 preempt 低的：

1. Queue1: RR policy → 50 ms 
2. Queue2: RR policy → 100 ms
3. Queue3: FIFO policy

要使用`sched_setscheduler()` system call 來設定 `SCHED_MLQ` policy

**You should not change the default scheduling policy for init and kernel threads. The only way to alter a task's priority is through invoking the sched_setparam() function. Otherwise, a task's priority must remain unchanged.**

---

`kernel/core.c`

要讓我們的 `SCHED_MLQ` 符合以下這些 system call: 

1. `sched_setscheduler()`
    1. This system call should be able to set the task's scheduling policy to `SCHED_MLQ` and assigning the task a priority level (1, 2, or 3) via the `sched_priority` field in the param parameter. The system call must return an error if an invalid priority is provided.
2. `sched_getscheduler()`
    1. The system call should return SCHED_MLQ if the task's policy is set to it.
3. `sched_setparam()`
    1. The system call should be able to set the task's priority, similar to the `sched_setscheduler()` system call.
4. `sched_getparam()`
    1. The system call should be able to **set the task's priority**, similar to the `sched_setscheduler()` system call.
5. `sched_get_priority_max()`
    1. The system call **should return 3** if the task is using the SCHED_MLQ policy.
6. `sched_get_priority_min()`
    1. The system call **should return 1** if the task is using the SCHED_MLQ policy.
7. `sched_setaffinity()`
    1. set the CPU affinity mask for a task using the SCHED_MLQ policy. The scheduler should assign a task to one of the specified CPUs on which it is eligible to run.
8. `sched_getaffinity()`
    1. The system call should retrive the cpu_set_t of the task using the SCHED_MLQ policy.

---

### `include/asm-generic/vmlinux.lds.h`

包含了生成kernel link 的 script，用於控制vmlinux 的 link 過程，確保 kernel 在 Memory 正確的位置，這邊新增了`__mlq_sched_class` 。

- code
    
    ```c
    #define SCHED_DATA				\
    	STRUCT_ALIGN();				\
    	__begin_sched_classes = .;		\
    	*(__idle_sched_class)			\
    	*(__fair_sched_class)			\
    	*(__mlq_sched_class)			\
    	*(__rt_sched_class)			\
    	*(__dl_sched_class)			\
    	*(__stop_sched_class)			\
    	__end_sched_classes = .;
    ```
    

### `include/linux/sched.h`

裡面是包含 linux scheduling 相關的 struct, macro, function 等等，有關 process creation, management, scheduling policy, etc。我們在這邊新定義了 `sched_mlq_entity` 以及在 `task_struct` 裡面新增了了 `mlq_prioriy` and `mlq` entity 

- code
    
    ```c
    ...
    struct sched_mlq_entity {
    	struct list_head		run_list;
    	unsigned long			timeout;
    	unsigned long			watchdog_stamp;
    	unsigned int			time_slice;
    	unsigned short			on_rq;
    	unsigned short			on_list;
    
    	struct sched_mlq_entity		*back;
    };
    ...
    struct task_struct{
    	unsigned int			mlq_priority;
    	struct sched_mlq_entity		mlq;
    }
    ...
    
    ```
    

### `include/linux/sched/mlq.h`

這個 header file 是新增的用來放幾個mlq相關的 function來判斷當前 task 的 priority 是否為mlq_policy底下之符合的 priority.

- code
    
    ```c
    /* SPDX-License-Identifier: GPL-2.0 */
    #ifndef _LINUX_SCHED_MLQ_H
    #define _LINUX_SCHED_MLQ_H
    
    #include <linux/sched.h>
    
    struct task_struct;
    
    static inline int mlq_prio(int prio)
    {
    	if (unlikely(prio > MAX_RT_PRIO && prio < MAX_RT_PRIO + MAX_MLQ_PRIO))
    		return 1;
    	return 0;
    }
    
    static inline int mlq_task(struct task_struct *p)
    {
    	return mlq_prio(p->prio);
    }
    
    #endif /* _LINUX_SCHED_RT_H */
    
    ```
    

### `include/linux/sched.h/prio.h`

這裡是跟 priority 相關的設定

- code
    
    ```c
    
    #define MAX_MLQ_PRIO 3
    #define MAX_PRIO		(MAX_RT_PRIO + MAX_MLQ_PRIO + NICE_WIDTH)
    #define DEFAULT_PRIO		(MAX_RT_PRIO + MAX_MLQ_PRIO + NICE_WIDTH / 2)
    ```
    

### `include/uapi/linux/sched.h`

提供一個可以讓 user interact with the kernerl in a controlled manner without exposing internal kernel details 的環境。

- code
    
    ```c
    #define SCHED_MLQ		7
    ```
    

### `include/kernel/sched/sched.h`

- code
    
    ```c
    // new add
    #include <linux/sched/mlq.h>
    
    static inline int mlq_policy(int policy)
    {
    	return policy == SCHED_MLQ;
    }
    
    static inline bool valid_policy(int policy)
    {
    	return idle_policy(policy) || fair_policy(policy) ||
    		mlq_policy(policy) ||
    		rt_policy(policy) || dl_policy(policy);
    }
    
    static inline int task_has_mlq_policy(struct task_struct *p)
    {
    	return mlq_policy(p->policy);
    }
    
    struct mlq_prio_array {
    	DECLARE_BITMAP(bitmap, MAX_MLQ_PRIO+1); /* include 1 bit for delimiter */
    	struct list_head queue[MAX_MLQ_PRIO];
    };
    
    struct mlq_rq {
    	struct mlq_prio_array	active;
    	unsigned int		mlq_nr_running;
    	unsigned int		rr_nr_running;
    	int			mlq_queued;
    
    	u64			mlq_time;
    	u64			mlq_runtime;
    	/* Nests inside the rq lock: */
    	raw_spinlock_t		mlq_runtime_lock;
    };
    
    static inline bool mlq_rq_is_runnable(struct mlq_rq *mlq_rq)
    {
    	return mlq_rq->mlq_queued && mlq_rq->mlq_nr_running;
    }
    
    extern const struct sched_class mlq_sched_class;
    
    static inline bool sched_mlq_runnable(struct rq *rq)
    {
    	return rq->mlq.mlq_queued > 0;
    }
    
    extern void init_sched_mlq_class(void);
    
    extern void print_mlq_stats(struct seq_file *m, int cpu);
    extern void print_mlq_rq(struct seq_file *m, int cpu, struct rt_rq *rt_rq);
    extern void init_mlq_rq(struct mlq_rq *mlq_rq);
    ```
    

### `include/kernel/sched/core.c`

跟排程執行相關的一些 function 定義在這邊，包含 system call 之類的定義。

- code
    
    ```c
    static inline int __normal_prio(int policy, int rt_prio, int nice)
    {
    	int prio;
    
    	if (dl_policy(policy))
    		prio = MAX_DL_PRIO - 1;
    	else if (rt_policy(policy))
    		prio = MAX_RT_PRIO - 1 - rt_prio;
    		// add mlq_policy
    	else if (mlq_policy(policy)) 
    		prio = MAX_RT_PRIO + MAX_MLQ_PRIO - rt_prio;
    	else
    		prio = NICE_TO_PRIO(nice);
    
    	return prio;
    }
    
    ...
    
    static void __setscheduler_prio(struct task_struct *p, int prio)
    {
    	if (dl_prio(prio))
    		p->sched_class = &dl_sched_class;
    	else if (rt_prio(prio))
    		p->sched_class = &rt_sched_class;
    	// add mlq_prio chech function
    	else if (mlq_prio(prio))
    		p->sched_class = &mlq_sched_class;
    	else
    		p->sched_class = &fair_sched_class;
    
    	p->prio = prio;
    }
    
    ...
    
    static void __setscheduler_params(struct task_struct *p,
    		const struct sched_attr *attr)
    {
    	int policy = attr->sched_policy;
    
    	if (policy == SETPARAM_POLICY)
    		policy = p->policy;
    
    	p->policy = policy;
    
    	if (dl_policy(policy))
    		__setparam_dl(p, attr);
    	else if (fair_policy(policy))
    		p->static_prio = NICE_TO_PRIO(attr->sched_nice);
    
    	/*
    	 * __sched_setscheduler() ensures attr->sched_priority == 0 when
    	 * !rt_policy. Always setting this ensures that things like
    	 * getparam()/getattr() don't report silly values for !rt tasks.
    	 */
    	p->mlq_priority = attr->sched_priority;
    	p->rt_priority = attr->sched_priority;
    	p->normal_prio = normal_prio(p);
    	set_load_weight(p, true);
    }
    
    ...
    
    static int __sched_setscheduler(struct task_struct *p,
    				const struct sched_attr *attr,
    				bool user, bool pi)
    {
    .
    .
    if (attr->sched_priority > MAX_RT_PRIO-1)
    		return -EINVAL;
    	if (mlq_policy(policy) && attr->sched_priority > MAX_MLQ_PRIO)
    		return -EINVAL;
    	if ((dl_policy(policy) && !__checkparam_dl(attr)) ||
    	    ((rt_policy(policy) || mlq_policy(policy)) != (attr->sched_priority != 0)))
    		return -EINVAL;				
    .
    .
    if (mlq_policy(policy) && attr->sched_priority != p->mlq_priority)
    			goto change;
    }
    
    ...
    
    static void get_params(struct task_struct *p, struct sched_attr *attr)
    {
    	if (task_has_dl_policy(p))
    		__getparam_dl(p, attr);
    	else if (task_has_rt_policy(p))
    		attr->sched_priority = p->rt_priority;
    	else if (task_has_mlq_policy(p))
    		attr->sched_priority = p->mlq_priority;
    	else
    		attr->sched_nice = task_nice(p);
    }
    
    ...
    
    SYSCALL_DEFINE2(sched_getparam, pid_t, pid, struct sched_param __user *, param)
    {
    	// new add
    	if (task_has_mlq_policy(p))
    		lp.sched_priority = p->mlq_priority;
    }
    
    ...
    
    SYSCALL_DEFINE1(sched_get_priority_max, int, policy)
    {
    	switch (policy) {
    	// new add
    	case SCHED_MLQ:
    		ret = MAX_MLQ_PRIO;
    		break;
    	}
    }
    
    ...
    
    SYSCALL_DEFINE1(sched_get_priority_min, int, policy)
    {
    	switch (policy) {
    	 
    	case SCHED_MLQ:
    		ret = 1;
    		break;
    	}
    }
    
    ...
    
    void __init sched_init(void)
    {
    	// new add
    	BUG_ON(&idle_sched_class + 1 != &fair_sched_class ||
    	       &fair_sched_class + 1 != &mlq_sched_class ||
    	       &mlq_sched_class + 1 != &rt_sched_class ||
    	       &rt_sched_class + 1   != &dl_sched_class);
    	       
    	 init_mlq_rq(&rq->mlq);
    }
    ```
    

### `include/kernel/sched/mlq.c`

mlq.c 相關之 function pointer array 宣告以及實作

- code
    
    ```c
    
    //SPDX-License-Identifier: GPL-2.0
    /*
     * Real-Time Scheduling Class (mapped to the SCHED_FIFO and SCHED_RR
     * policies)
     */
    #include "sched.h"
    
    enum {
    	MLQ_Zero,
    	MLQ_FirstRR,
    	MLQ_SecondRR,
    	MLQ_FIFO,
    };
    
    int sched_rr_1_timeslice = (50 * HZ / 1000);
    int sched_rr_2_timeslice = (100 * HZ / 1000);
    
    void init_mlq_rq(struct mlq_rq *mlq_rq)
    {
    	struct mlq_prio_array *array;
    	int i;
    
    	array = &mlq_rq->active;
    	for (i = 0; i < MAX_MLQ_PRIO; i++) {
    		INIT_LIST_HEAD(array->queue + i);
    		__clear_bit(i, array->bitmap);
    	}
    	/* delimiter for bitsearch: */
    	__set_bit(MAX_MLQ_PRIO, array->bitmap);
    
    	/* We stamlq is dequeued state, because no RT tasks are queued */
    	mlq_rq->mlq_queued = 0;
    
    	mlq_rq->mlq_time = 0;
    	mlq_rq->mlq_runtime = 0;
    	raw_spin_lock_init(&mlq_rq->mlq_runtime_lock);
    }
    
    static inline struct task_struct *mlq_task_of(struct sched_mlq_entity *mlq_se)
    {
    	return container_of(mlq_se, struct task_struct, mlq);
    }
    
    static inline struct rq *rq_of_mlq_rq(struct mlq_rq *mlq_rq)
    {
    	return container_of(mlq_rq, struct rq, mlq);
    }
    
    static inline struct rq *rq_of_mlq_se(struct sched_mlq_entity *mlq_se)
    {
    	struct task_struct *p = mlq_task_of(mlq_se);
    
    	return task_rq(p);
    }
    
    static inline struct mlq_rq *mlq_rq_of_se(struct sched_mlq_entity *mlq_se)
    {
    	struct rq *rq = rq_of_mlq_se(mlq_se);
    
    	return &rq->mlq;
    }
    
    static void enqueue_top_mlq_rq(struct mlq_rq *mlq_rq);
    static void dequeue_top_mlq_rq(struct mlq_rq *mlq_rq);
    
    static inline int on_mlq_rq(struct sched_mlq_entity *mlq_se)
    {
    	return mlq_se->on_rq;
    }
    
    static inline int mlq_se_prio(struct sched_mlq_entity *mlq_se)
    {
    	return mlq_task_of(mlq_se)->mlq_priority - 1;
    }
    
    /*
     * Update the current task's runtime statistics. Skip current tasks that
     * are not in our scheduling class.
     */
    static void update_curr_mlq(struct rq *rq)
    {
    	struct task_struct *curr = rq->curr;
    	struct sched_mlq_entity *mlq_se = &curr->mlq;
    	u64 delta_exec;
    	u64 now;
    
    	if (curr->sched_class != &mlq_sched_class)
    		return;
    
    	now = rq_clock_task(rq);
    	delta_exec = now - curr->se.exec_start;
    	if (unlikely((s64)delta_exec <= 0))
    		return;
    
    	schedstat_set(curr->se.statistics.exec_max,
    		      max(curr->se.statistics.exec_max, delta_exec));
    
    	curr->se.sum_exec_runtime += delta_exec;
    	curr->se.exec_start = now;
    }
    
    static void
    dequeue_top_mlq_rq(struct mlq_rq *mlq_rq)
    {
    	struct rq *rq = rq_of_mlq_rq(mlq_rq);
    
    	WARN_ON(&rq->mlq != mlq_rq);
    
    	if (!mlq_rq->mlq_queued)
    		return;
    
    	WARN_ON(!rq->nr_running);
    
    	sub_nr_running(rq, mlq_rq->mlq_nr_running);
    	mlq_rq->mlq_queued = 0;
    
    }
    
    static void
    enqueue_top_mlq_rq(struct mlq_rq *mlq_rq)
    {
    	struct rq *rq = rq_of_mlq_rq(mlq_rq);
    
    	WARN_ON(&rq->mlq != mlq_rq);
    
    	if (mlq_rq->mlq_queued)
    		return;
    
    	if (mlq_rq->mlq_nr_running) {
    		add_nr_running(rq, mlq_rq->mlq_nr_running);
    		mlq_rq->mlq_queued = 1;
    	}
    
    	/* Kick cpufreq (see the comment in kernel/sched/sched.h). */
    	cpufreq_update_util(rq, 0);
    }
    
    static inline
    unsigned int mlq_se_nr_running(struct sched_mlq_entity *mlq_se)
    {
    	return 1;
    }
    
    static inline
    unsigned int mlq_se_rr_nr_running(struct sched_mlq_entity *mlq_se)
    {
    	struct task_struct *tsk;
    
    	tsk = mlq_task_of(mlq_se);
    
    	return (tsk->policy == SCHED_RR) ? 1 : 0;
    }
    
    static inline
    void inc_mlq_tasks(struct sched_mlq_entity *mlq_se, struct mlq_rq *mlq_rq)
    {
    	WARN_ON(!mlq_prio(mlq_se_prio(mlq_se)));
    	mlq_rq->mlq_nr_running += mlq_se_nr_running(mlq_se);
    	mlq_rq->rr_nr_running += mlq_se_rr_nr_running(mlq_se);
    }
    
    static inline
    void dec_mlq_tasks(struct sched_mlq_entity *mlq_se, struct mlq_rq *mlq_rq)
    {
    	WARN_ON(!mlq_prio(mlq_se_prio(mlq_se)));
    	WARN_ON(!mlq_rq->mlq_nr_running);
    	mlq_rq->mlq_nr_running -= mlq_se_nr_running(mlq_se);
    	mlq_rq->rr_nr_running -= mlq_se_rr_nr_running(mlq_se);
    }
    
    /*
     * Change mlq_se->run_list location unless SAVE && !MOVE
     *
     * assumes ENQUEUE/DEQUEUE flags match
     */
    static inline bool move_entity(unsigned int flags)
    {
    	if ((flags & (DEQUEUE_SAVE | DEQUEUE_MOVE)) == DEQUEUE_SAVE)
    		return false;
    
    	return true;
    }
    
    static void __delist_mlq_entity(struct sched_mlq_entity *mlq_se, struct mlq_prio_array *array)
    {
    	list_del_init(&mlq_se->run_list);
    
    	if (list_empty(array->queue + mlq_se_prio(mlq_se)))
    		__clear_bit(mlq_se_prio(mlq_se), array->bitmap);
    
    	mlq_se->on_list = 0;
    }
    
    static void __enqueue_mlq_entity(struct sched_mlq_entity *mlq_se, unsigned int flags)
    {
    	struct mlq_rq *mlq_rq = mlq_rq_of_se(mlq_se);
    	struct mlq_prio_array *array = &mlq_rq->active;
    	struct list_head *queue = array->queue + mlq_se_prio(mlq_se);
    
    	if (move_entity(flags)) {
    		WARN_ON_ONCE(mlq_se->on_list);
    		if (flags & ENQUEUE_HEAD)
    			list_add(&mlq_se->run_list, queue);
    		else
    			list_add_tail(&mlq_se->run_list, queue);
    
    		__set_bit(mlq_se_prio(mlq_se), array->bitmap);
    		mlq_se->on_list = 1;
    	}
    	mlq_se->on_rq = 1;
    
    	inc_mlq_tasks(mlq_se, mlq_rq);
    }
    
    static void __dequeue_mlq_entity(struct sched_mlq_entity *mlq_se, unsigned int flags)
    {
    	struct mlq_rq *mlq_rq = mlq_rq_of_se(mlq_se);
    	struct mlq_prio_array *array = &mlq_rq->active;
    
    	if (move_entity(flags)) {
    		WARN_ON_ONCE(!mlq_se->on_list);
    		__delist_mlq_entity(mlq_se, array);
    	}
    	mlq_se->on_rq = 0;
    
    	dec_mlq_tasks(mlq_se, mlq_rq);
    }
    
    static void enqueue_mlq_entity(struct sched_mlq_entity *mlq_se, unsigned int flags)
    {
    	struct rq *rq = rq_of_mlq_se(mlq_se);
    
    	dequeue_top_mlq_rq(mlq_rq_of_se(mlq_se));
    
    	if (on_mlq_rq(mlq_se))
    		__dequeue_mlq_entity(mlq_se, flags);
    
    	__enqueue_mlq_entity(mlq_se, flags);
    
    	enqueue_top_mlq_rq(&rq->mlq);
    }
    
    static void dequeue_mlq_entity(struct sched_mlq_entity *mlq_se, unsigned int flags)
    {
    	struct rq *rq = rq_of_mlq_se(mlq_se);
    
    	dequeue_top_mlq_rq(mlq_rq_of_se(mlq_se));
    
    	if (on_mlq_rq(mlq_se))
    		__dequeue_mlq_entity(mlq_se, flags);
    
    	enqueue_top_mlq_rq(&rq->mlq);
    }
    
    /*
     * Put task to the head or the end of the run list without the overhead of
     * dequeue followed by enqueue.
     */
    static void requeue_mlq_entity(struct mlq_rq *mlq_rq, struct sched_mlq_entity *mlq_se, int head)
    {
    	if (on_mlq_rq(mlq_se)) {
    		struct mlq_prio_array *array = &mlq_rq->active;
    		struct list_head *queue = array->queue + mlq_se_prio(mlq_se);
    
    		if (head)
    			list_move(&mlq_se->run_list, queue);
    		else
    			list_move_tail(&mlq_se->run_list, queue);
    	}
    }
    
    /*
     * Adding/removing a task to/from a priority array:
     */
    static void enqueue_task_mlq(struct rq *rq, struct task_struct *p, int flags)
    {
    	struct sched_mlq_entity *mlq_se = &p->mlq;
    
    	if (flags & ENQUEUE_WAKEUP)
    		mlq_se->timeout = 0;
    
    	enqueue_mlq_entity(mlq_se, flags);
    }
    
    static void dequeue_task_mlq(struct rq *rq, struct task_struct *p, int flags)
    {
    	struct sched_mlq_entity *mlq_se = &p->mlq;
    
    	update_curr_mlq(rq);
    	dequeue_mlq_entity(mlq_se, flags);
    }
    
    static void requeue_task_mlq(struct rq *rq, struct task_struct *p, int head)
    {
    	struct sched_mlq_entity *mlq_se = &p->mlq;
    	struct mlq_rq *mlq_rq;
    
    	mlq_rq = mlq_rq_of_se(mlq_se);
    	requeue_mlq_entity(mlq_rq, mlq_se, head);
    }
    
    static void yield_task_mlq(struct rq *rq)
    {
    	requeue_task_mlq(rq, rq->curr, 0);
    }
    
    /*
     * Preempt the current task with a newly woken task if needed:
     */
    static void check_preempt_curr_mlq(struct rq *rq, struct task_struct *p, int flags)
    {
    	if (p->mlq_priority < rq->curr->mlq_priority) {
    		resched_curr(rq);
    		return;
    	}
    }
    
    static inline void set_next_task_mlq(struct rq *rq, struct task_struct *p, bool first)
    {
    	p->se.exec_start = rq_clock_task(rq);
    }
    
    static struct sched_mlq_entity *pick_next_mlq_entity(struct rq *rq, struct mlq_rq *mlq_rq)
    {
    	struct mlq_prio_array *array = &mlq_rq->active;
    	struct sched_mlq_entity *next = NULL;
    	struct list_head *queue;
    	int idx;
    
    	idx = sched_find_first_bit(array->bitmap);
    	WARN_ON(idx >= MAX_MLQ_PRIO);
    
    	queue = array->queue + idx;
    	next = list_entry(queue->next, struct sched_mlq_entity, run_list);
    
    	return next;
    }
    
    static struct task_struct *_pick_next_task_mlq(struct rq *rq)
    {
    	struct sched_mlq_entity *mlq_se;
    	struct mlq_rq *mlq_rq  = &rq->mlq;
    
    	mlq_se = pick_next_mlq_entity(rq, mlq_rq);
    	// WARN_ON(!mlq_se);
    
    	return mlq_task_of(mlq_se);
    }
    
    static struct task_struct *pick_task_mlq(struct rq *rq)
    {
    	struct task_struct *p;
    
    	if (!sched_mlq_runnable(rq))
    		return NULL;
    
    	p = _pick_next_task_mlq(rq);
    
    	return p;
    }
    
    static struct task_struct *pick_next_task_mlq(struct rq *rq)
    {
    	struct task_struct *p = pick_task_mlq(rq);
    
    	if (p)
    		set_next_task_mlq(rq, p, true);
    
    	return p;
    }
    
    static void put_prev_task_mlq(struct rq *rq, struct task_struct *p)
    {
    	update_curr_mlq(rq);
    }
    
    /*
     * When switching a task to RT, we may overload the runqueue
     * with RT tasks. In this case we try to push them off to
     * other runqueues.
     */
    static void switched_to_mlq(struct rq *rq, struct task_struct *p)
    {
    	/*
    	 * If we are running, update the avg_mlq tracking, as the running time
    	 * will now on be accounted into the latter.
    	 */
    	if (task_current(rq, p))
    		return;
    
    	/*
    	 * If we are not running we may need to preempt the current
    	 * running task. If that current running task is also an RT task
    	 * then see if we can move to another run queue.
    	 */
    	if (task_on_rq_queued(p)) {
    
    		if ((rq->curr->sched_class >= &fair_sched_class) ||
    			(rq->curr->sched_class == &mlq_sched_class &&
    			 rq->curr->mlq_priority > p->mlq_priority)) {
    			if (cpu_online(cpu_of(rq)))
    				resched_curr(rq);
    		}
    	}
    }
    
    /*
     * Priority of the task has changed. This may cause
     * us to initiate a push or pull.
     */
    static void prio_changed_mlq(struct rq *rq, struct task_struct *p, int oldprio)
    {
    	if (!task_on_rq_queued(p))
    		return;
    
    	if (task_current(rq, p)) {
    		/* For UP simply resched on drop of prio */
    		if (oldprio < p->mlq_priority)
    			resched_curr(rq);
    	} else {
    		/*
    		 * This task is not running, but if it is
    		 * greater than the current running task
    		 * then reschedule.
    		 */
    		if (p->mlq_priority < rq->curr->mlq_priority)
    			resched_curr(rq);
    	}
    }
    
    /*
     * scheduler tick hitting a task of our scheduling class.
     *
     * NOTE: This function can be called remotely by the tick offload that
     * goes along full dynticks. Therefore no local assumption can be made
     * and everything must be accessed through the @rq and @curr passed in
     * parameters.
     */
    static void task_tick_mlq(struct rq *rq, struct task_struct *task, int queued)
    {
    	struct sched_mlq_entity *mlq_se = &task->mlq;
    
    	update_curr_mlq(rq);
    
    	/*
    	 * RR tasks need a special form of timeslice management.
    	 * FIFO tasks have no timeslices.
    	 */
    	if (task->mlq_priority == MLQ_FIFO)
    		return;
    
    	if (--task->mlq.time_slice)
    		return;
    
    	if (task->mlq_priority == MLQ_FirstRR)
    		task->mlq.time_slice = sched_rr_1_timeslice;
    
    	if (task->mlq_priority == MLQ_SecondRR)
    		task->mlq.time_slice = sched_rr_2_timeslice;
    
    	/*
    	 * Requeue to the end of queue if we (and all of our ancestors) are not
    	 * the only element on the queue
    	 */
    	if (mlq_se->run_list.prev != mlq_se->run_list.next) {
    		requeue_task_mlq(rq, task, 0);
    		resched_curr(rq);
    		return;
    	}
    #ifdef CONFIG_MLQ_PRIORITY_CHANGE_POLICY
    
    #include<linux/random.h>
    
    	if (task->se.sum_exec_runtime > 10000000000) {
    
    		int flags = DEQUEUE_MOVE;
    
    		dequeue_task_mlq(rq, task, flags);
    		task->mlq_priority = prandom_u32() % 3 + 1;
    		enqueue_task_mlq(rq, task, flags);
    		resched_curr(rq);
    		return;
    	}
    #endif
    }
    
    static unsigned int get_rr_interval_mlq(struct rq *rq, struct task_struct *task)
    {
    	/*
    	 * Time slice is 0 for SCHED_FIFO tasks
    	 */
    
    	if (task->mlq_priority == 1)
    		return 50;
    	else if (task->mlq_priority == 2)
    		return 100;
    	else
    		return 0;
    }
    
    #ifdef CONFIG_SMP
    static int balance_mlq(struct rq *rq, struct task_struct *p, struct rq_flags *rf)
    {
    	return sched_mlq_runnable(rq);
    }
    #endif
    
    DEFINE_SCHED_CLASS(mlq) = {
    
    	.enqueue_task			= enqueue_task_mlq,
    	.dequeue_task			= dequeue_task_mlq,
    	.yield_task				= yield_task_mlq,
    
    	.check_preempt_curr		= check_preempt_curr_mlq,
    
    	.pick_next_task			= pick_next_task_mlq,
    	.put_prev_task			= put_prev_task_mlq,
    	.set_next_task          = set_next_task_mlq,
    
    	.task_tick				= task_tick_mlq,
    
    	.get_rr_interval		= get_rr_interval_mlq,
    
    	.prio_changed			= prio_changed_mlq,
    	.switched_to			= switched_to_mlq,
    
    	.update_curr			= update_curr_mlq,
    
    	#ifdef CONFIG_SMP
    	.balance				= balance_mlq,
    	.pick_task				= pick_task_mlq,
    	.set_cpus_allowed       = set_cpus_allowed_common,
    	#endif
    };
    
    #ifdef CONFIG_SCHED_DEBUG
    void print_mlq_stats(struct seq_file *m, int cpu)
    {
    	rt_rq_iter_t iter;
    	struct mlq_rq *mlq_rq;
    
    	rcu_read_lock();
    	for_each_mlq_rq(mlq_rq, iter, cpu_rq(cpu))
    		print_mlq_rq(m, cpu, mlq_rq);
    	rcu_read_unlock();
    }
    #endif /* CONFIG_SCHED_DEBUG */
    
    ```
    

---

最後要改 Makefile

```makefile
# add mlq.o
obj-y += idle.o fair.o rt.o mlq.o deadline.o 
```

```bash
git diff > mlq_schduler.patch

git apply [path to patch]

```
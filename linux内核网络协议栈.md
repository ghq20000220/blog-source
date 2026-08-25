---
title: linux内核网络协议栈
date: 2026-08-20 13:55:53
mermaid: false
tags:
    - linux
    - 以太网
    - NAPI
    - 软中断
    - top-half
    - bottom-half
hide: false
---

# 环境
1. linux version: 5.10.237
2. 以太网驱动：stmmac xgmac（dw xgmac）
# 架构概览


# 收发数据包过程


# stmmac xgmac初始化


## 中断上半部、下半部
linux 内核中对中断嵌套的定义是：这里的中断嵌套应当指的是中断上半部的嵌套，即硬件的中断不会被新的硬件的中断打断

### 中断上半部  
又叫硬中断，是通常意义上的中断处理程序。在关中断的状态下处理，不可被中断  
简单理解为:由request_irq注册的处理函数，称为中断上半部

### 中断下半部  
处理中断中的耗时，非紧急任务。在开中断状态下处理，可被中断
- 中断下半部的具体实现主要有三种机制：软中断，Tasklet，work queue

## 软中断
软中断是中断下半部机制的其中一种实现  
软中断的触发（执行时机）：总计有三种路径  
1. 快速路径，每次有硬件中断，硬中断处理结束后**irq_exit**中会调用 **__do_softirq**，以riscv为例

    - 调用顺序
        ```text
        handle_exception
            handle_arch_irq（riscv_intc_irq）
                handle_domain_irq
                    __handle_domain_irq
                        generic_handle_irq --> 硬件中断处理
                        irq_exit
                            __irq_exit_rcu
                                invoke_softirq
                                    do_softirq_own_stack
                                        __do_softirq  --> 软中断处理
        ```

    - linux/arch/riscv/kernel/head.S中，将**handle_exception**设置为中断和异常的入口
        ```asm
            la a0, handle_exception
            csrw CSR_TVEC, a0   // TVEC（trap vector）寄存器用于存储中断和异常的入口
        ```
    - linux/arch/riscv/kernel/entry.S， **handle_exception**函数调用**handle_arch_irq**为处理中断。  
    riscv_intc_init函数将**handle_arch_irq**初始化为**riscv_intc_irq**
        ```c
            static int __init riscv_intc_init(struct device_node *node,
                        struct device_node *parent)
            {
                int rc, hartid;

                hartid = riscv_of_parent_hartid(node);
                if (hartid < 0) {
                    pr_warn("unable to find hart id for %pOF\n", node);
                    return 0;
                }

                /*
                * The DT will have one INTC DT node under each CPU (or HART)
                * DT node so riscv_intc_init() function will be called once
                * for each INTC DT node. We only need to do INTC initialization
                * for the INTC DT node belonging to boot CPU (or boot HART).
                */
                if (riscv_hartid_to_cpuid(hartid) != smp_processor_id())
                    return 0;

                intc_domain = irq_domain_add_linear(node, BITS_PER_LONG,
                                    &riscv_intc_domain_ops, NULL);
                if (!intc_domain) {
                    pr_err("unable to add IRQ domain\n");
                    return -ENXIO;
                }

                rc = set_handle_irq(&riscv_intc_irq);

                if (rc) {
                    pr_err("failed to set irq handler\n");
                    return rc;
                }

                cpuhp_setup_state(CPUHP_AP_IRQ_RISCV_STARTING,
                        "irqchip/riscv/intc:starting",
                        riscv_intc_cpu_starting,
                        riscv_intc_cpu_dying);

                pr_info("%d local interrupts mapped\n", BITS_PER_LONG);

                return 0;
            }
        ```
    - riscv_intc_irq函数如下，非RV_IRQ_SOFT执行**handle_domain_irq**
        ```c
            static asmlinkage void riscv_intc_irq(struct pt_regs *regs)
            {
                unsigned long cause = regs->cause & ~CAUSE_IRQ_FLAG;

                if (unlikely(cause >= BITS_PER_LONG))
                    panic("unexpected interrupt cause");

                switch (cause) {
            #ifdef CONFIG_SMP
                case RV_IRQ_SOFT:
                    /*
                    * We only use software interrupts to pass IPIs, so if a
                    * non-SMP system gets one, then we don't know what to do.
                    */
                    handle_IPI(regs);
                    break;
            #endif
                default:
                    handle_domain_irq(intc_domain, cause, regs);
                    break;
                }
            }
        ```
    - handle_domain_irq：generic_handle_irq处理对应的中断
        ```C
            static inline int handle_domain_irq(struct irq_domain *domain,
                                unsigned int hwirq, struct pt_regs *regs)
            {
                return __handle_domain_irq(domain, hwirq, true, regs);
            }


            int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
                bool lookup, struct pt_regs *regs)
            {
                struct pt_regs *old_regs = set_irq_regs(regs);
                unsigned int irq = hwirq;
                int ret = 0;

                irq_enter();

            #ifdef CONFIG_IRQ_DOMAIN
                if (lookup)
                    irq = irq_find_mapping(domain, hwirq);
            #endif

                /*
                * Some hardware gives randomly wrong interrupts.  Rather
                * than crashing, do something sensible.
                */
                if (unlikely(!irq || irq >= nr_irqs)) {
                    ack_bad_irq(irq);
                    ret = -EINVAL;
                } else {
                    generic_handle_irq(irq);
                }

                irq_exit();
                set_irq_regs(old_regs);
                return ret;
            }
        ```

    - __do_softirq: 可以看到在软中断处理过程中打开了中断 **local_irq_enable**，当一轮软中断遍历后，如果还有软中断则会调用**wakeup_softirqd**唤醒ksoftirqd线程
        ```c
            asmlinkage __visible void __softirq_entry __do_softirq(void)
            {
                unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
                unsigned long old_flags = current->flags;
                int max_restart = MAX_SOFTIRQ_RESTART;
                struct softirq_action *h;
                bool in_hardirq;
                __u32 pending;
                int softirq_bit;

                /*
                * Mask out PF_MEMALLOC as the current task context is borrowed for the
                * softirq. A softirq handled, such as network RX, might set PF_MEMALLOC
                * again if the socket is related to swapping.
                */
                current->flags &= ~PF_MEMALLOC;

                pending = local_softirq_pending();
                account_irq_enter_time(current);

                __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
                in_hardirq = lockdep_softirq_start();

            restart:
                /* Reset the pending bitmask before enabling irqs */
                set_softirq_pending(0);

                local_irq_enable();

                h = softirq_vec;

                while ((softirq_bit = ffs(pending))) {
                    unsigned int vec_nr;
                    int prev_count;

                    h += softirq_bit - 1;

                    vec_nr = h - softirq_vec;
                    prev_count = preempt_count();

                    kstat_incr_softirqs_this_cpu(vec_nr);

                    trace_softirq_entry(vec_nr);
                    h->action(h);
                    trace_softirq_exit(vec_nr);
                    if (unlikely(prev_count != preempt_count())) {
                        pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
                            vec_nr, softirq_to_name[vec_nr], h->action,
                            prev_count, preempt_count());
                        preempt_count_set(prev_count);
                    }
                    h++;
                    pending >>= softirq_bit;
                }

                if (__this_cpu_read(ksoftirqd) == current)
                    rcu_softirq_qs();
                local_irq_disable();

                pending = local_softirq_pending();
                if (pending) {
                    if (time_before(jiffies, end) && !need_resched() &&
                        --max_restart)
                        goto restart;

                    wakeup_softirqd();
                }

                lockdep_softirq_end(in_hardirq);
                account_irq_exit_time(current);
                __local_bh_enable(SOFTIRQ_OFFSET);
                WARN_ON_ONCE(in_interrupt());
                current_restore_flags(old_flags, PF_MEMALLOC);
            }
        ```
        1. 进入handle_exception函数中断处于disable状态（硬件自动关闭中断），在处理硬中断对应的中断服务函数时，中断始终处于关闭状态，进入__do_softirq后调用**local_irq_enable**开启中断，使**action**部分可以被打断。
        2. __do_softirq中轮询中断标志，处理软中断。
        3. **softirq_vec**是一个函数指针类型的数组，用于存储对应类型的软中断处理函数，内核现有10种类型的软中断  
            ```c
            enum
            {
                HI_SOFTIRQ=0,
                TIMER_SOFTIRQ,
                NET_TX_SOFTIRQ,
                NET_RX_SOFTIRQ,
                BLOCK_SOFTIRQ,
                IRQ_POLL_SOFTIRQ,
                TASKLET_SOFTIRQ,
                SCHED_SOFTIRQ,
                HRTIMER_SOFTIRQ,
                RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

                NR_SOFTIRQS
            };
            ```
        4. 其中**NET_TX_SOFTIRQ**，**NET_RX_SOFTIRQ**，对应的软中断处理函数用于以太网的收发。在net_dev_init（linux/net/core/dev.c）中进行了注册初始化


2. 慢速路径，ksoftirqd线程被唤醒时会执行 __do_softirq
    - ksoftirqd是一个内核线程和其它线程一样由内核调度，在spawn_ksoftirqd（linux/kernel/softirq.c）中创建了ksoftirqd线程

        ```c
        static void run_ksoftirqd(unsigned int cpu)
        {
            local_irq_disable();
            if (local_softirq_pending()) {
                /*
                * We can safely run softirq on inline stack, as we are not deep
                * in the task stack here.
                */
                __do_softirq();
                local_irq_enable();
                cond_resched();
                return;
            }
            local_irq_enable();
        }

        static struct smp_hotplug_thread softirq_threads = {
            .store			= &ksoftirqd,
            .thread_should_run	= ksoftirqd_should_run,
            .thread_fn		= run_ksoftirqd,
            .thread_comm		= "ksoftirqd/%u",
        };

        static __init int spawn_ksoftirqd(void)
        {
            cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
                        takeover_tasklets);
            BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

            return 0;
        }
        ```

    - smpboot_register_percpu_thread，以smpboot_thread_fn作为对应的线程fn，在smpboot_thread_fn中执行对应的run_ksoftirqd函数。也就是smpboot_thread_fn作为各种内核线程的框架，利用参数执行对应的线程处理函数

    - smpboot_register_percpu_thread在注册内核线程时，调度策略会被配置为***SCHED_NORMAL***。  
        1. 这里引入一个曾经遇到的问题，具体如下：  
            - 运行环境为单核linux
            - 使用实时线程**SCHED_FIFO**调度策略处理密集计算型任务
            - 使用实时线程**SCHED_FIFO**调度策略，在任务中执行以太网接收
            - 使用上位机给board发送数据，board进行接收
            - 在所有任务都运行起来后，发现上位机的以太网发送速率极低，远低于预期带宽  

        2. 现分析如下：  
            - 由于实时任务优先级高于SCHED_NORMAL的任务，导致ksoftirqd没有机会执行，以太网的接收处理基本处于被饿死的状态。
            - 我们通过上面的内容也可以知道，ksoftirqd作为内核任务，会处理以太网接收的下半部，这个任务并不会继承用户任务的优先级，当用户任务以高优先级SCHED_FIFO策略进行recv时，内核依然要通过SCHED_NORMAL管理ksoftirqd线程。

        ```c
        static __printf(4, 0)
        struct task_struct *__kthread_create_on_node(int (*threadfn)(void *data),
                                    void *data, int node,
                                    const char namefmt[],
                                    va_list args)
        {
            DECLARE_COMPLETION_ONSTACK(done);
            struct task_struct *task;
            struct kthread_create_info *create = kmalloc(sizeof(*create),
                                    GFP_KERNEL);

            if (!create)
                return ERR_PTR(-ENOMEM);
            create->threadfn = threadfn;
            create->data = data;
            create->node = node;
            create->done = &done;

            spin_lock(&kthread_create_lock);
            list_add_tail(&create->list, &kthread_create_list);
            spin_unlock(&kthread_create_lock);

            wake_up_process(kthreadd_task);
            /*
            * Wait for completion in killable state, for I might be chosen by
            * the OOM killer while kthreadd is trying to allocate memory for
            * new kernel thread.
            */
            if (unlikely(wait_for_completion_killable(&done))) {
                /*
                * If I was SIGKILLed before kthreadd (or new kernel thread)
                * calls complete(), leave the cleanup of this structure to
                * that thread.
                */
                if (xchg(&create->done, NULL))
                    return ERR_PTR(-EINTR);
                /*
                * kthreadd (or new kernel thread) will call complete()
                * shortly.
                */
                wait_for_completion(&done);
            }
            task = create->result;
            if (!IS_ERR(task)) {
                static const struct sched_param param = { .sched_priority = 0 };
                char name[TASK_COMM_LEN];

                /*
                * task is already visible to other tasks, so updating
                * COMM must be protected.
                */
                vsnprintf(name, sizeof(name), namefmt, args);
                set_task_comm(task, name);
                /*
                * root may have changed our (kthreadd's) priority or CPU mask.
                * The kernel thread should not inherit these properties.
                */
                sched_setscheduler_nocheck(task, SCHED_NORMAL, &param);
                set_cpus_allowed_ptr(task,
                            housekeeping_cpumask(HK_FLAG_KTHREAD));
            }
            kfree(create);
            return task;
        }
        ```
    - 具体的唤醒流程暂时不了解

3. __local_bh_enable_ip（进程上下文中调用）：
    ```c
    void __local_bh_enable_ip(unsigned long ip, unsigned int cnt)
    {
        WARN_ON_ONCE(in_irq());
        lockdep_assert_irqs_enabled();
    #ifdef CONFIG_TRACE_IRQFLAGS
        local_irq_disable();
    #endif
        /*
        * Are softirqs going to be turned on now:
        */
        if (softirq_count() == SOFTIRQ_DISABLE_OFFSET)
            lockdep_softirqs_on(ip);
        /*
        * Keep preemption disabled until we are done with
        * softirq processing:
        */
        preempt_count_sub(cnt - 1);

        if (unlikely(!in_interrupt() && local_softirq_pending())) {
            /*
            * Run softirq if any pending. And do it in its own stack
            * as we may be calling this deep in a task call stack already.
            */
            do_softirq();
        }

        preempt_count_dec();
    #ifdef CONFIG_TRACE_IRQFLAGS
        local_irq_enable();
    #endif
        preempt_check_resched();
    }
    ```

## NAPI
[NAPI](https://docs.linuxkernel.org.cn/networking/napi.html)是 Linux 网络栈使用的事件处理机制。NAPI 这个名字现在已经没有任何特定的具体含义了  
新版本内核（5.14+ ?）引入了线程化NAPI，线程化 NAPI 是一种操作模式，它使用专用的内核线程而不是软件 IRQ 上下文来进行 NAPI 处理(暂不了解，仅介绍软中断相关的NAPI)
### NAPI和软中断的关系
- 从上面的软中断介绍，我们可以得NAPI是运行于软中断上下文中的。

### xgmac中断
- 中断注册  
    在stmmac_open中注册了中断，对应的中断服务函数为**stmmac_interrupt**
    ```c
    ret = request_irq(dev->irq, stmmac_interrupt,
        IRQF_SHARED, dev->name, dev);
    ```

- 中断响应  
    在中断服务函数stmmac_interrupt中，并不会直接调用**stmmac_rx**将sk_buff向上层传递
    ```text
    调用如下
    stmmac_interrupt
        stmmac_dma_interrupt
            stmmac_napi_check
                __napi_schedule（会将NET_RX_SOFTIRQ的软中断标志置位）
    ```
    ```c
    static irqreturn_t stmmac_interrupt(int irq, void *dev_id)
    {
        ***省略部分代码***

        /* To handle DMA interrupts */
        stmmac_dma_interrupt(priv);

        return IRQ_HANDLED;
    }
    ```
    ```c
    static void stmmac_dma_interrupt(struct stmmac_priv *priv)
    {
        ***省略部分代码***
        for (chan = 0; chan < channels_to_check; chan++)
            status[chan] = stmmac_napi_check(priv, chan);
        ***省略部分代码***
    }
    ```
    ```c
    static int stmmac_napi_check(struct stmmac_priv *priv, u32 chan)
    {
        int status = stmmac_dma_interrupt_status(priv, priv->ioaddr,
                            &priv->xstats, chan);
        struct stmmac_channel *ch = &priv->channel[chan];
        unsigned long flags;

        if ((status & handle_rx) && (chan < priv->plat->rx_queues_to_use)) {
            if (napi_schedule_prep(&ch->rx_napi)) {
                spin_lock_irqsave(&ch->lock, flags);
                stmmac_disable_dma_irq(priv, priv->ioaddr, chan, 1, 0);
                spin_unlock_irqrestore(&ch->lock, flags);
                __napi_schedule(&ch->rx_napi);
            }
        }

        if ((status & handle_tx) && (chan < priv->plat->tx_queues_to_use)) {
            if (napi_schedule_prep(&ch->tx_napi)) {
                spin_lock_irqsave(&ch->lock, flags);
                stmmac_disable_dma_irq(priv, priv->ioaddr, chan, 0, 1);
                spin_unlock_irqrestore(&ch->lock, flags);
                __napi_schedule(&ch->tx_napi);
            }
        }

        return status;
    }
    ```

### xgmac NAPI注册
NAPI运行于软中断的上下文中，网卡收到第一包时触发中断，中断上半部主动屏蔽网卡的中断。在软中断上下文中执行对应的poll方法，一次性从网卡中批量取出多个数据包

- stmmac_dvr_probe调用**stmmac_napi_add**调用**netif_napi_add**向系统中初始化注册 NAPI 实例。并指定了调用的poll函数（**stmmac_napi_poll_rx**，**stmmac_napi_poll_tx**）和预算NAPI_POLL_WEIGHT
    ```c
    static void stmmac_napi_add(struct net_device *dev)
    {
        struct stmmac_priv *priv = netdev_priv(dev);
        u32 queue, maxq;

        maxq = max(priv->plat->rx_queues_to_use, priv->plat->tx_queues_to_use);

        for (queue = 0; queue < maxq; queue++) {
            struct stmmac_channel *ch = &priv->channel[queue];

            ch->priv_data = priv;
            ch->index = queue;
            spin_lock_init(&ch->lock);

            if (queue < priv->plat->rx_queues_to_use) {
                netif_napi_add(dev, &ch->rx_napi, stmmac_napi_poll_rx,
                        NAPI_POLL_WEIGHT);
            }
            if (queue < priv->plat->tx_queues_to_use) {
                netif_tx_napi_add(dev, &ch->tx_napi,
                        stmmac_napi_poll_tx,
                        NAPI_POLL_WEIGHT);
            }
        }
    }
    ```

### NAPI内核实例

- 硬中断服务程序：在硬中断服务程序中基本没做什么事情，只是调用__napi_schedule，触发中断下半部
    ```text
    stmmac_interrupt
        stmmac_dma_interrupt
            stmmac_napi_check
                __napi_schedule
    ```

- __napi_schedule  
将指定的struct napi_struct添加到**当前CPU**的poll_list链表尾部  
触发NET_RX_SOFTIRQ软中断，内核在执行__do_softirq时，就会执行net_rx_action，从而遍历poll_list,执行对应的poll（**stmmac_napi_poll_rx**）函数
    1. 调用栈
    ```text
    __napi_schedule
        ____napi_schedule
            list_add_tail(将napi加入到当前cpu的poll_list尾部)
            __raise_softirq_irqoff
                or_softirq_pending(置NET_RX_SOFTIRQ标志位)
    ```
    2. 相关函数和宏定义
    ```c
    void __napi_schedule(struct napi_struct *n)
    {
        unsigned long flags;

        local_irq_save(flags);
        ____napi_schedule(this_cpu_ptr(&softnet_data), n);
        local_irq_restore(flags);
    }

    static inline void ____napi_schedule(struct softnet_data *sd,
                        struct napi_struct *napi)
    {
        list_add_tail(&napi->poll_list, &sd->poll_list);
        __raise_softirq_irqoff(NET_RX_SOFTIRQ);
    }

    void __raise_softirq_irqoff(unsigned int nr)
    {
        lockdep_assert_irqs_disabled();
        trace_softirq_raise(nr);
        or_softirq_pending(1UL << nr);
    }

    #define local_softirq_pending()	(__this_cpu_read(local_softirq_pending_ref))
    #define set_softirq_pending(x)	(__this_cpu_write(local_softirq_pending_ref, (x)))
    #define or_softirq_pending(x)	(__this_cpu_or(local_softirq_pending_ref, (x)))

    #define local_softirq_pending_ref irq_stat.__softirq_pending

    #ifndef __ARCH_IRQ_STAT
    DEFINE_PER_CPU_ALIGNED(irq_cpustat_t, irq_stat);
    EXPORT_PER_CPU_SYMBOL(irq_stat);
    #endif
    ```
- net_dev_init  
注册了NET_RX_SOFTIRQ中断对应的处理函数
    1. 调用关系
    ```text
    net_dev_init
        open_softirq(NET_TX_SOFTIRQ, net_tx_action);
        open_softirq(NET_RX_SOFTIRQ, net_rx_action);

    ```
    2. 相关函数
    ```c
    void open_softirq(int nr, void (*action)(struct softirq_action *))
    {
        softirq_vec[nr].action = action;
    }

    static __latent_entropy void net_rx_action(struct softirq_action *h)
    {
        struct softnet_data *sd = this_cpu_ptr(&softnet_data);
        unsigned long time_limit = jiffies +
            usecs_to_jiffies(READ_ONCE(netdev_budget_usecs));
        int budget = READ_ONCE(netdev_budget);
        LIST_HEAD(list);
        LIST_HEAD(repoll);

        local_irq_disable();
        list_splice_init(&sd->poll_list, &list);
        local_irq_enable();

        for (;;) {
            struct napi_struct *n;

            if (list_empty(&list)) {
                if (!sd_has_rps_ipi_waiting(sd) && list_empty(&repoll))
                    goto out;
                break;
            }

            n = list_first_entry(&list, struct napi_struct, poll_list);
            budget -= napi_poll(n, &repoll);

            /* If softirq window is exhausted then punt.
            * Allow this to run for 2 jiffies since which will allow
            * an average latency of 1.5/HZ.
            */
            if (unlikely(budget <= 0 ||
                    time_after_eq(jiffies, time_limit))) {
                sd->time_squeeze++;
                break;
            }
        }

        local_irq_disable();

        list_splice_tail_init(&sd->poll_list, &list);
        list_splice_tail(&repoll, &list);
        list_splice(&list, &sd->poll_list);
        if (!list_empty(&sd->poll_list))
            __raise_softirq_irqoff(NET_RX_SOFTIRQ);

        net_rps_action_and_irq_enable(sd);
    out:
        __kfree_skb_flush();
    }
    ```

- __do_softirq  
软中断上下文，在其中处理各类软中断
    1. 调用关系
    ```text
    __do_softirq
        h->action(h); ---> 执行对应的软中断处理函数，以NET_RX_SOFTIRQ为例，即执行net_rx_action
        net_rx_action ---> 假设执行net_rx_action
            napi_poll --> 执行对应的以太网rx poll函数
                work = n->poll(n, weight);  stmmac_napi_poll_rx
                stmmac_napi_poll_rx
                    stmmac_rx   --> 最终的以太网rx接收函数
                        napi_gro_receive   --> 将skb向上层传递（其中包括包的合并处理）
    ```

    2. 相关函数
    ```c
    asmlinkage __visible void __softirq_entry __do_softirq(void)
    {
        unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
        unsigned long old_flags = current->flags;
        int max_restart = MAX_SOFTIRQ_RESTART;
        struct softirq_action *h;
        bool in_hardirq;
        __u32 pending;
        int softirq_bit;

        /*
        * Mask out PF_MEMALLOC as the current task context is borrowed for the
        * softirq. A softirq handled, such as network RX, might set PF_MEMALLOC
        * again if the socket is related to swapping.
        */
        current->flags &= ~PF_MEMALLOC;

        pending = local_softirq_pending();        --> 通过irq_stat变量获取对应的软中断pending
        account_irq_enter_time(current);

        __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
        in_hardirq = lockdep_softirq_start();

    restart:
        /* Reset the pending bitmask before enabling irqs */
        set_softirq_pending(0);                  --> 清除所有软中断

        local_irq_enable();

        h = softirq_vec;

        while ((softirq_bit = ffs(pending))) {
            unsigned int vec_nr;
            int prev_count;

            h += softirq_bit - 1;        -->获取对应软中断的中断服务函数

            vec_nr = h - softirq_vec;
            prev_count = preempt_count();

            kstat_incr_softirqs_this_cpu(vec_nr);

            trace_softirq_entry(vec_nr);
            h->action(h);        -->执行对应软中断的中断服务函数
            trace_softirq_exit(vec_nr);
            if (unlikely(prev_count != preempt_count())) {
                pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
                    vec_nr, softirq_to_name[vec_nr], h->action,
                    prev_count, preempt_count());
                preempt_count_set(prev_count);
            }
            h++;
            pending >>= softirq_bit;
        }

        if (__this_cpu_read(ksoftirqd) == current)
            rcu_softirq_qs();
        local_irq_disable();

        pending = local_softirq_pending();      --> 再次读取软中断状态，在h->action(h);中可能会再次置起标志位
        if (pending) {
            if (time_before(jiffies, end) && !need_resched() &&
                --max_restart)
                goto restart;

            wakeup_softirqd();        --> 唤醒ksoftirqd线程处理
        }

        lockdep_softirq_end(in_hardirq);
        account_irq_exit_time(current);
        __local_bh_enable(SOFTIRQ_OFFSET);
        WARN_ON_ONCE(in_interrupt());
        current_restore_flags(old_flags, PF_MEMALLOC);
    }

    #define local_softirq_pending_ref irq_stat.__softirq_pending
    #define local_softirq_pending()	(__this_cpu_read(local_softirq_pending_ref))
    ```

    ```c
    static __latent_entropy void net_rx_action(struct softirq_action *h)
    {
        struct softnet_data *sd = this_cpu_ptr(&softnet_data);        --> 获取当前core的softnet_data
        unsigned long time_limit = jiffies +
            usecs_to_jiffies(READ_ONCE(netdev_budget_usecs));
        int budget = READ_ONCE(netdev_budget);
        LIST_HEAD(list);
        LIST_HEAD(repoll);

        local_irq_disable();
        list_splice_init(&sd->poll_list, &list);        --> 将 sd->poll_list 移动到 list，sd->poll_list中即存储了由 __napi_schedule 链入的struct napi_struct
        local_irq_enable();

        for (;;) {
            struct napi_struct *n;

            if (list_empty(&list)) {
                if (!sd_has_rps_ipi_waiting(sd) && list_empty(&repoll))
                    goto out;
                break;
            }

            n = list_first_entry(&list, struct napi_struct, poll_list);
            budget -= napi_poll(n, &repoll);    --> 内部执行对应的poll函数

            /* If softirq window is exhausted then punt.
            * Allow this to run for 2 jiffies since which will allow
            * an average latency of 1.5/HZ.
            */
            if (unlikely(budget <= 0 ||
                    time_after_eq(jiffies, time_limit))) {
                sd->time_squeeze++;
                break;
            }
        }

        local_irq_disable();

        list_splice_tail_init(&sd->poll_list, &list);
        list_splice_tail(&repoll, &list);
        list_splice(&list, &sd->poll_list);
        if (!list_empty(&sd->poll_list))
            __raise_softirq_irqoff(NET_RX_SOFTIRQ);     --> 置起软中断标志位

        net_rps_action_and_irq_enable(sd);
    out:
        __kfree_skb_flush();
    }
    ```
    ```c
    static int napi_poll(struct napi_struct *n, struct list_head *repoll)
    {
        void *have;
        int work, weight;

        list_del_init(&n->poll_list);

        have = netpoll_poll_lock(n);

        weight = n->weight;

        /* This NAPI_STATE_SCHED test is for avoiding a race
        * with netpoll's poll_napi().  Only the entity which
        * obtains the lock and sees NAPI_STATE_SCHED set will
        * actually make the ->poll() call.  Therefore we avoid
        * accidentally calling ->poll() when NAPI is not scheduled.
        */
        work = 0;
        if (test_bit(NAPI_STATE_SCHED, &n->state)) {
            work = n->poll(n, weight);     --> 执行 stmmac_napi_poll_rx
            trace_napi_poll(n, work, weight);
        }

        if (unlikely(work > weight))
            pr_err_once("NAPI poll function %pS returned %d, exceeding its budget of %d.\n",
                    n->poll, work, weight);

        if (likely(work < weight))
            goto out_unlock;

        /* Drivers must not modify the NAPI state if they
        * consume the entire weight.  In such cases this code
        * still "owns" the NAPI instance and therefore can
        * move the instance around on the list at-will.
        */
        if (unlikely(napi_disable_pending(n))) {
            napi_complete(n);
            goto out_unlock;
        }

        if (n->gro_bitmask) {
            /* flush too old packets
            * If HZ < 1000, flush all packets.
            */
            napi_gro_flush(n, HZ >= 1000);
        }

        gro_normal_list(n);

        /* Some drivers may have called napi_schedule
        * prior to exhausting their budget.
        */
        if (unlikely(!list_empty(&n->poll_list))) {
            pr_warn_once("%s: Budget exhausted after napi rescheduled\n",
                    n->dev ? n->dev->name : "backlog");
            goto out_unlock;
        }

        list_add_tail(&n->poll_list, repoll);

    out_unlock:
        netpoll_poll_unlock(have);

        return work;
    }

    ```

    ```c
    static int stmmac_napi_poll_rx(struct napi_struct *napi, int budget)
    {
        struct stmmac_channel *ch =
            container_of(napi, struct stmmac_channel, rx_napi);
        struct stmmac_priv *priv = ch->priv_data;
        u32 chan = ch->index;
        int work_done;

        priv->xstats.napi_poll++;

        work_done = stmmac_rx(priv, budget, chan);      --> 当消耗超出预算时，即此次处理没有完成，不会执行napi_complete_done，还保存在list中
        if (work_done < budget && napi_complete_done(napi, work_done)) {
            unsigned long flags;

            spin_lock_irqsave(&ch->lock, flags);
            stmmac_enable_dma_irq(priv, priv->ioaddr, chan, 1, 0);
            spin_unlock_irqrestore(&ch->lock, flags);
        }

        return work_done;
    }

    ```

### stmmac rx向上层传递数据包的流程
```text
stmmac_rx
    napi_gro_receive
        napi_skb_finish
            gro_normal_one
                gro_normal_list
                    netif_receive_skb_list_internal
                        __netif_receive_skb_list
                            __netif_receive_skb_list_core
                                __netif_receive_skb_core
                                    deliver_skb
                                    deliver_ptype_list_skb
                                        deliver_skb
                                            pt_prev->func(skb, skb->dev, pt_prev, orig_dev); --> ip_rcv
                                                ip_rcv
                                                    ip_rcv_finish
                                                        ip_rcv_finish_core --> ?路由
                                                        dst_input
                                                            ip_local_deliver
                                                                ip_local_deliver_finish
                                                                    ip_protocol_deliver_rcu
                                                                        tcp_v4_rcv
                                                                            tcp_v4_do_rcv
                                                                                tcp_rcv_established
                                                                                    tcp_queue_rcv --> 保存数据到socket的接收队列
                                                                                    tcp_data_ready --> 唤醒等待的进程
```

### tcp接收阻塞
```text


```

# RISCV PLIC
PLIC 不支持中断抢占或嵌套机制。在中断触发后硬件自动关闭外部中断，因此想要实现软中断机制，需要在执行__do_softirq时打开外部中断，使得外部中断可以抢占软中断。

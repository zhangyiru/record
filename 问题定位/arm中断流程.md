## arm64异常类型

1. 中断(Interrupts)，就是我们平常理解的中断，主要由外设触发，是典型的异步异常。 

   ARM64中主要包括两种类型的中断：IRQ(普通中断)和FIQ(高优先级中断，处理更快)。 Linux内核中好像没有使用FIQ。

2. Aborts。 可能是同步或异步异常。包括指令异常(取指令时产生)、数据异常(读写内存数据时产生)，可以由MMU产生(比如典型的缺页异常)，也可以由外部存储系统产生(通常是硬件问题)。

3. Reset。复位被视为一种特殊的异常。

4. Exception generating instructions。由异常触发指令触发的异常，比如Supervisor Call (SVC)、Hypervisor Call (HVC)、Secure monitor Call (SMC)



## 异常级别

ARM中，异常由级别之分，具体如下图所示，只要关注：

普通的用户程序处于EL0，级别最低

内核处于EL1，HyperV处于EL2，EL1-3属于特权级别

![Arm64中的异常级别](https://github.com/HappySeeker/happyseeker.github.io/raw/master/_posts/Exception-Level-in-AArch64.png)

当异常发生在用户态时，异常级别(EL)会发生切换，默认切换到EL1(内核态)



## 中断向量表

Arm64架构中的中断向量表有点特别(相对于X86来说~)，包含16个entry，这16个entry分为4组，每组包含4个entry，每组中的4个entry分别对应4种类型的异常：

1. SError
2. FIQ
3. IRQ
4. Synchronous Aborts



## armv8/arm64 中断/系统调用流程

https://cloud.tencent.com/developer/article/1413292



实现的函数有el1_sync，el1_irq，el0_sync，el0_irq。其余都是invalid。

el1_sync：当前处于内核态时，发生了指令执行异常、缺页中断（跳转地址或者取地址）。

el1_irq：当前处于内核态时，发生硬件中断。

el0_sync：当前处于用户态时，发生了指令执行异常、缺页中断（跳转地址或者取地址）、系统调用。

el0_irq：当前处于用户态时，发生了硬件中断。



## Linux中断处理: 脉络分析

https://zhuanlan.zhihu.com/p/185851980

<img src="https://pic3.zhimg.com/v2-ca533f3a152e45fe782de4097d4724d3_1440w.jpg?source=172ae18b" alt="Linux中断处理: 脉络分析" style="zoom: 67%;" />



```shell
[13432014.096473] Call trace:
[13432014.096477]  dump_backtrace+0x0/0x198
[13432014.096479]  show_stack+0x24/0x30
[13432014.096481]  dump_stack+0xa4/0xcc
[13432014.096483]  watchdog_timer_fn+0x300/0x3e8
[13432014.096485]  __hrtimer_run_queues+0x114/0x358
[13432014.096486]  hrtimer_interrupt+0x104/0x2d8
[13432014.096489]  arch_timer_handler_phys+0x38/0x58
[13432014.096490]  handle_percpu_devid_irq+0x90/0x248
[13432014.096492]  generic_handle_irq+0x34/0x50
[13432014.096493]  __handle_domain_irq+0x68/0xc0
[13432014.096494]  gic_handle_irq+0x6c/0x170
[13432014.096495]  el1_irq+0xb8/0x140
[13432014.096498]  next_positive.isra.2+0x70/0xb8
[13432014.096499]  dcache_readdir+0x130/0x1b0
[13432014.096501]  iterate_dir+0x8c/0x1a8
[13432014.096502]  ksys_getdents64+0xa4/0x190
[13432014.096503]  __arm64_sys_getdents64+0x28/0x38
[13432014.096505]  el0_svc_common+0x78/0x130
[13432014.096507]  el0_svc_handler+0x38/0x78
[13432014.096508]  el0_svc+0x8/0xc
```



```c
/*
 * EL0 mode handlers.
 */
    .align  6
el0_sync:
    kernel_entry 0
    mrs x25, esr_el1            // read the syndrome register
    lsr x24, x25, #ESR_ELx_EC_SHIFT // exception class
    cmp x24, #ESR_ELx_EC_SVC64      // SVC in 64-bit state
    b.eq    el0_svc //当x24==#ESR_ELx_EC_SVC64相等时跳转
    cmp x24, #ESR_ELx_EC_DABT_LOW   // data abort in EL0
    ...
```

如果该值等于#ESR_ELx_EC_SVC64说明该同步异常是由SVC指令触发的（系统调用也由该指令触发），程序跳转到el0_svc处



```c
/*
 * SVC handler.
 */
        .align  6
el0_svc:
        mov     x0, sp
        bl      el0_svc_handler //执行系统调用，
        b       ret_to_user //使用内核栈保存的寄存器值恢复进程的寄存器并返回用户模式
ENDPROC(el0_svc)
```





```c
调用流程：
el0_svc_handler
    el0_svc_common
		invoke_syscall
    		__invoke_syscall
    			syscall_fn
    
asmlinkage void el0_svc_handler(struct pt_regs *regs)
{
        sve_user_discard();
        el0_svc_common(regs, regs->regs[8], __NR_syscalls, sys_call_table);
}

static void el0_svc_common(struct pt_regs *regs, int scno, int sc_nr,
                           const syscall_fn_t syscall_table[])
{
        unsigned long flags = current_thread_info()->flags;

        regs->orig_x0 = regs->regs[0];//x0中保存了栈指针sp
        regs->syscallno = scno;//保存系统调用号到pt_regs结构体

        local_daif_restore(DAIF_PROCCTX);
        user_exit();

        if (has_syscall_work(flags)) {//检查_TIF_SYSCALL_WORK位，如果为1
                /* set default errno for user-issued syscall(-1) */
                if (scno == NO_SYSCALL)
                        regs->regs[0] = -ENOSYS;
                scno = syscall_trace_enter(regs);//用于跟踪系统调用的情况，将系统调用进入时的信息写入审计上下文中，与安全审计功能有关
                if (scno == NO_SYSCALL)
                        goto trace_exit;
        }

        invoke_syscall(regs, scno, sc_nr, syscall_table);//执行系统调用

        /*
         * The tracing status may have changed under our feet, so we have to
         * check again. However, if we were tracing entry, then we always trace
         * exit regardless, as the old entry assembly did.
         */
        if (!has_syscall_work(flags) && !IS_ENABLED(CONFIG_DEBUG_RSEQ)) {
                local_daif_mask();
                flags = current_thread_info()->flags;
                if (!has_syscall_work(flags)) {
                        /*
                         * We're off to userspace, where interrupts are
                         * always enabled after we restore the flags from
                         * the SPSR.
                         */
                        trace_hardirqs_on();
                        return;
                }
                local_daif_restore(DAIF_PROCCTX);
        }

trace_exit:
        syscall_trace_exit(regs);//用于跟踪系统调用的情况，将系统调用退出时的信息写入审计上下文中，与安全审计功能有关
}

static long __invoke_syscall(struct pt_regs *regs, syscall_fn_t syscall_fn)
{
        return syscall_fn(regs);
}

static void invoke_syscall(struct pt_regs *regs, unsigned int scno,
                           unsigned int sc_nr,
                           const syscall_fn_t syscall_table[])
{
        long ret;

        if (scno < sc_nr) {
                syscall_fn_t syscall_fn;
                syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
                ret = __invoke_syscall(regs, syscall_fn);
        } else {
                ret = do_ni_syscall(regs, scno);
        }

        regs->regs[0] = ret;
}
```



```c
/*
 * "slow" syscall return path.
 */
ret_to_user:
        disable_daif
        ldr     x1, [tsk, #TSK_TI_FLAGS]
        and     x2, x1, #_TIF_WORK_MASK
        cbnz    x2, work_pending
finish_ret_to_user:
        enable_step_tsk x1, x2
#ifdef CONFIG_GCC_PLUGIN_STACKLEAK
        bl      stackleak_erase
#endif
        kernel_exit 0
ENDPROC(ret_to_user)
```



## 相关UE错误处理

![image-20230325110609766](C:\Users\z00585918\AppData\Roaming\Typora\typora-user-images\image-20230325110609766.png)

不可纠正错误（Uncorrected，缩写标记为UE）该错误被检测到且未被纠正或延迟，错误在系统中潜伏

它又可划分为下面几个子类：潜伏错误或可重启错误（UEO），带标记错误或可恢复错误（UER），不可恢复错误（UEU），不可抑制错误（UC）



```c
//arch/arm64/kernel/entry.S
el1_error:
    kernel_entry 1
    mrs x1, esr_el1
    enable_dbg
    mov x0, sp
    bl  do_serror //called
    kernel_exit 1
ENDPROC(el1_error)
```



```c
//arch/arm64/kernel/traps.c
asmlinkage void do_serror(struct pt_regs *regs, unsigned int esr)
{
	nmi_enter();

	/* non-RAS errors are not containable */
	if (!arm64_is_ras_serror(esr) || arm64_is_fatal_ras_serror(regs, esr)) //UEU,UER,UC会panic
		arm64_serror_panic(regs, esr);

	nmi_exit();
}
```



```c
bool arm64_is_fatal_ras_serror(struct pt_regs *regs, unsigned int esr)
{
	u32 aet = arm64_ras_serror_get_severity(esr);

	switch (aet) {
	case ESR_ELx_AET_CE:	/* corrected error */
	case ESR_ELx_AET_UEO:	/* restartable, not yet consumed */
		/*
		 * The CPU can make progress. We may take UEO again as
		 * a more severe error.
		 */
		return false;

	case ESR_ELx_AET_UEU:	/* Uncorrected Unrecoverable */
	case ESR_ELx_AET_UER:	/* Uncorrected Recoverable */
		/*
		 * The CPU can't make progress. The exception may have
		 * been imprecise.
		 */
		return true;

	case ESR_ELx_AET_UC:	/* Uncontainable or Uncategorized error */
	default:
		/* Error has been silently propagated */
		arm64_serror_panic(regs, esr);
	}
}
```



## 参考链接

1、ARM Linux内核的系统调用（2）

https://zhuanlan.zhihu.com/p/149267399

2、进程调度时机

https://blog.csdn.net/u012294613/article/details/124221978

3、Linux中断管理机制

https://www.cnblogs.com/arnoldlu/p/8659981.html

4、CPU故障注入技术指导（ARM）

http://3ms.huawei.com/km/blogs/details/12371077

cccvvcccc\# start kernel part IX

`CONFIG_PREEMPT` is not set in our environment, `preempt_disable` do nothing, ignore it.

`irqs_disabled` is a macro, it could be found in include/linux/irqflags.h:114, as well as `raw_local_save_flags`, with all macros expanded, actual code of `irqs_disabled` shown as follow:

```irqs\_disabled
unsigned long _flags;
do { (flags) = __raw_local_save_flags(); } while (0)
raw_irqs_disabled_flags(_flags);
```

`__raw_local_save_flags` \(arch/x86/include/asm/irqflags.h:64\) invovles `native_save_fl` which reads the flags register.

```native\_save\_fl
static inline unsigned long native_save_fl(void)
{
    unsigned long flags;

    /*
     * "=rm" is safe here, because "pop" adjusts the stack before
     * it evaluates its effective address -- this is part of the
     * documented behavior of the "pop" instruction.
     */
    asm volatile("# __raw_save_flags\n\t"
             "pushf ; pop %0"
             : "=rm" (flags)
             : /* no input */
             : "memory");

    return flags;
}
```

`raw_irqs_disabled_flags` checks if raw interrupt is disabled and returns the result.

If interrupt is enabled, disable it with `local_irq_disable`.

## _initialize read-copy-update_

In computer science, [read-copy-update \(RCU\)](https://en.wikipedia.org/wiki/Read-copy-update) is a synchronization mechanism based on mutual exclusion.\[note 1\] It is used when performance of reads are crucial and is an example of space-time tradeoff, enabling fast operations at the cost of more space.

Read-copy-update allows multiple threads to efficiently read from shared memory by deferring updates after pre-existing reads to a later time while simultaneously marking the data, ensuring new readers will read the updated data. This makes all readers proceed as if there were no synchronization involved, hence they will be fast, but also making updates more difficult.

`rcu_init` involves `__rcu_init` which initializes rcu\_state structure of `rcu_sched_state` and `rcu_bh_state`, as well as per cpu rcu date, after initialization completed, `__rcu_init` registers response routine for `RCU_SOFTIRQ`.

`rcu_init` register `rcu_barrier_cpu_hotplug` as the response routine when cpu up/down to create/delete rcu data.

`rcu_init` initialzes per cpu rcu data for current up CPU.

## _initialize hardware interrupt_

There are two types of interaction between the CPU and the rest of the computer's hardware. The first type is when the CPU gives orders to the hardware, the other is when the hardware needs to tell the CPU something. The second, called interrupts, is much harder to implement because it has to be dealt with when convenient for the hardware, not the CPU. Hardware devices typically have a very small amount of RAM, and if you don't read their information when available, it is lost.

Under Linux, hardware interrupts are called IRQ's \(Interrupt Requests\)\[1\]. There are two types of IRQ's, short and long. A short IRQ is one which is expected to take a very short period of time, during which the rest of the machine will be blocked and no other interrupts will be handled. A long IRQ is one which can take longer, and during which other interrupts may occur \(but not interrupts from the same device\). If at all possible, it's better to declare an interrupt handler to be long.

When the CPU receives an interrupt, it stops whatever it's doing \(unless it's processing a more important interrupt, in which case it will deal with this one only when the more important one is done\), saves certain parameters on the stack and calls the interrupt handler. This means that certain things are not allowed in the interrupt handler itself, because the system is in an unknown state. The solution to this problem is for the interrupt handler to do what needs to be done immediately, usually read something from the hardware or send something to the hardware, and then schedule the handling of the new information at a later time \(this is called the "bottom half"\) and return. The kernel is then guaranteed to call the bottom half as soon as possible -- and when it does, everything allowed in kernel modules will be allowed.

`early_irq_init`:

* allocates cpu variables for all possible cpus and set all cpus in the allocated cpumask.
* initialize nr\_irqs based on nr\_cpu\_ids.

```arch\_probe\_nr\_irqs
arch_probe_nr_irqs () at arch/x86/kernel/apic/io_apic.c:3868
3868        if (nr_irqs > (NR_VECTORS * nr_cpu_ids))
(gdb) p nr_irqs
$1 = 2304

......

3882    }
(gdb) p nr_irqs
$6 = 256
```

* allocates memory for array `irq_desc_ptrs` based on `nr_irqs`.
* allocates memory for array `kstat_irqs_legacy` based on `nr_cpu_ids`.
* initializes array `irq_desc_ptrs`

## _initialize interrupt call gate of interrupt description table_

`native_init_IRQ` initialize interrupt call gates and register function of interrupts.

## _initialize data used for priority search tree_

`prio_tree_init` initializes array `index_bits_to_maxindex` which is used to quickly find node in priority search tree. About priority search tree, we can find many references by google.

## _initialize timer_

In the Linux kernel, time is measured by a global variable named jiffies, which identifies the number of ticks that have occurred since the system was booted. The manner in which ticks are counted depends, at its lowest level, on the particular hardware platform on which you're running; however, it is typically incremented through an interrupt. The tick rate \(jiffies's least significant bit\) is configurable, but in a recent 2.6 kernel for x86, a tick equals 4ms \(250Hz\). The jiffies global variable is used broadly in the kernel for a number of purposes, one of which is the current absolute time to calculate the time-out value for a timer \(you'll see examples of this later\).

There are few different schemes for timers in recent 2.6 kernels. The simplest and least accurate of all timers \(though suitable for most instances\) is the timer API. This API permits the construction of timers that operate in the jiffies domain \(minimum 4ms time-out\). There's also the high-resolution timer API, which permits timer constructions in which time is defined in nanoseconds. Depending upon your processor and the speed at which it operates, your mileage may vary, but the API does offer a way to schedule time-outs below the jiffies tick interval.

`init_timers`:

* Initializes per cpu variable `boot_tvec_bases` for cpu 0
* Initializes per cpu look up lock
* Adds notifier to `cpu_chain`
* Registers function for running timers and timer-tq in bootom half context for `TIMER_SOFTIRQ`

  The initialization of high resolution timers is similar, ignore it.

## _initialize soft interrupt_

The softirq mechanism is meant to handle processing that is almost — but not quite — as important as the handling of hardware interrupts. Softirqs run at a high priority \(though with an interesting exception, described below\), but with hardware interrupts enabled. They thus will normally preempt any work except the response to a "real" hardware interrupt.

Once upon a time, there were 32 hardwired software interrupt vectors, one assigned to each device driver or related task. Drivers have, for the most part, been detached from software interrupts for a long time — they still use softirqs, but that access has been laundered through intermediate APIs like tasklets and timers. In current kernels there are ten softirq vectors defined; two for tasklet processing, two for networking, two for the block layer, two for timers, and one each for the scheduler and read-copy-update processing. The kernel maintains a per-CPU bitmask indicating which softirqs need processing at any given time. So, for example, when a kernel subsystem calls tasklet\_schedule\(\), the TASKLET\_SOFTIRQ bit is set on the corresponding CPU and, when softirqs are processed, the tasklet will be run.

There are two places where software interrupts can "fire" and preempt the current thread. One of them is at the end of the processing for a hardware interrupt; it is common for interrupt handlers to raise softirqs, so it makes sense \(for latency and optimal cache use\) to process them as soon as hardware interrupts can be re-enabled. The other possibility is anytime that kernel code re-enables softirq processing \(via a call to functions like local\_bh\_enable\(\) or spin\_unlock\_bh\(\)\). The end result is that the accumulated softirq work \(which can be substantial\) is executed in the context of whichever process happens to be running at the wrong time; that is the "randomly chosen victim" aspect that Thomas was talking about.

`softirq_init`:

* Initializes per cpu variables `tasklet_vec`, `tasklet_hi_vec` and `softirq_work_list` for each cpus
* Registers notifier for cpu hot plugin
* Opens soft interrupt for `TASKLET_SOFTIRQ` and `HI_SOFTIRQ`

## _initializes timekeeping_

To provide timekeeping for your platform, the clock source provides the basic timeline, whereas clock events shoot interrupts on certain points on this timeline, providing facilities such as high-resolution timers. sched\_clock\(\) is used for scheduling and timestamping, and delay timers provide an accurate delay source using hardware counters.

`timekeeping_init` initilizes clocksource and common timekeeping values.

## _initialize profile_

There are several facilities to see where the kernel spends its resources. A simple one is the profiling function, that stores the current EIP \(instruction pointer\) at each clock tick.

Boot the kernel with command line option profile=2 \(or some other number instead of 2\). This will cause a file /proc/profile to be created. The number given after profile= is the number of positions EIP is shifted right when profiling. So a large number gives a coarse profile. The counters are reset by writing to /proc/profile. The utility readprofile will output statistics for you. It does not sort - you have to invoke sort explicitly. But given a memory map it will translate addresses to kernel symbols.

See kernel/profile.c and fs/proc/proc\_misc.c and readprofile\(1\).

For example:

```
# echo > /proc/profile
...
# readprofile -m System.map-2.5.59 | sort -nr | head -2
510502 total                                      0.1534
508548 default_idle                           10594.7500
```

The first column gives the number of timer ticks. The last column gives the number of ticks divided by the size of the function.

The command readprofile -r is equivalent to echo &gt; /proc/profile.

`profile_init` initializes profile length\(only code is profiled\) and allocates buffer of profile if profile feature enabled.

## _enable interrupt_

```
asmlinkage void __init start_kernel(void)

    ......

    early_boot_irqs_on();
    local_irq_enable();

    ......
```

`early_boot_irqs_on` sets flag `early_boot_irqs_enabled`, `local_irq_enable` is a macro, it's defination could be found in include/linux/irqflags.h:59

```local\_irq\_enable
#define local_irq_enable() \
    do { trace_hardirqs_on(); raw_local_irq_enable(); } while (0)
```

`trace_hardirqs_on` enables hardirqs of current task and `raw_local_irq_enable` set interrupt flag `IF = 1`

## _set GFP mask_

`gfp_allowed_mask` is set to `GFP_BOOT_MASK` during early boot to restrict what GFP flags are used before interrupts are enabled. Once interrupts are enabled, it is set to \_\_GFP\_BITS\_MASK while the system is running. During hibernation, it is used by PM to avoid I/O during memory allocation while devices are suspended.

## _initialze console device_

Do some early initialization and do the complex setup later.

`console_init`:

* Sets up the default TTY line discipline
* Sets up the console device

```console\_init
void __init console_init(void)
{
    ......

    call = __con_initcall_start;
    while (call < __con_initcall_end) {
(gdb) p __con_initcall_start
$17 = 0xc177e008 <__initcall_con_init>
(gdb) p __con_initcall_end
$18 = 0xc177e014 <__initcall_selinux_init>
(gdb) p __con_initcall_start+1
$19 = (initcall_t *) 0xc177e00c <__initcall_hvc_console_init>
(gdb) p __con_initcall_start+2
$20 = (initcall_t *) 0xc177e010 <__initcall_serial8250_console_init>
(gdb) p __con_initcall_start+3
$21 = (initcall_t *) 0xc177e014 <__initcall_selinux_init>
        (*call)();
        call++;
    }

    .......
}
```

## _initialize idr cache_

Idr is a set of library functions for the management of small integer ID numbers. In essence, an idr object can be thought of as a sparse array mapping integer IDs onto arbitrary pointers, with a "get me an available entry" function as well. This code was first added in February, 2003 as part of the POSIX clocks patch, and has seen various tweaks since.

`idr_init_cache` allocates cache for `idr_layer_cache`.

## _late time init_

Global function pointer `late_time_init` is initialized in `time_init`.

```time\_init
/*
 * Initialize TSC and delay the periodic timer init to
 * late x86_late_time_init() so ioremap works.
 */
void __init time_init(void)
{
    late_time_init = x86_late_time_init;
}
```

From the defination of routine `x86_late_time_init`, it involves the time initialize function of `x86_init` which initialized in arch/x86/kernel/x86\_init.c

```x86\_init
/*
 * The platform setup functions are preset with the default functions
 * for standard PC hardware.
 */
struct x86_init_ops x86_init __initdata = {

        ......

    .timers = {
        .setup_percpu_clockev    = setup_boot_APIC_clock,
        .tsc_pre_init        = x86_init_noop,
        .timer_init        = hpet_time_init,
    },
};
```

The defination of `hpet_time_init` shown as follow:

```hpet\_time\_init
/* Default timer init function */
void __init hpet_time_init(void)
{
    if (!hpet_enable())
        setup_pit_timer();
    setup_default_timer_irq();
}
```

`hpet_enable` tries to setup HPET timer.

* Checks hpet capability.
* Mapps hpet memory region

```hpet\_set\_mapping
hpet_set_mapping () at arch/x86/kernel/hpet.c:71
71        hpet_virt_address = ioremap_nocache(hpet_address, HPET_MMAP_SIZE);
(gdb) p /x hpet_address 
$2 = 0xfed00000
```

* Reads the HPET period, HPET config and HPET ID, checks the value.

  `hpet_readl` reads value from specified memory address with `readl`

```hpet\_readl
unsigned long hpet_readl(unsigned long a)
{
    return readl(hpet_virt_address + a);
}
```

Defination of routine `readl` shown as follow

```readl
#define build_mmio_read(name, size, type, reg, barrier) \
static inline type name(const volatile void __iomem *addr) \
{ type ret; asm volatile("mov" size " %1,%0":reg (ret) \
:"m" (*(volatile type __force *)addr) barrier); return ret; }

build_mmio_read(readl, "l", unsigned int, "=r", :"memory")
```

After macro expanded, `readl` looks as follow, it reads the value from specified address.

```readl
static inline unsigned int readl(const volatile void __iomem *addr)
{
    unsigned int ret;
    asm volatile("movl %1,%0"
                  :reg(ret)
                  :"m" (*(volatile type __force *)addr) barrier);
    return ret;
}
```

The result of `hpet_period`:

```hpet\_period
856        hpet_period = hpet_readl(HPET_PERIOD);
(gdb) 
871        for (i = 0; hpet_readl(HPET_CFG) == 0xFFFFFFFF; i++) {
(gdb) 
880        if (hpet_period < HPET_MIN_PERIOD || hpet_period > HPET_MAX_PERIOD)
(gdb) p hpet_period 
$3 = 10000000
```

* Starts HPET counter and verified the counter, after that installs new cloocksources
* Register clockevent if legacy HPET ID detected

  After `hpet_enable` complete, setup timter interrupt

  Next in `x86_late_time_init` is `tsc_init` which initializes variables used by tsc and registers clocksource for tsc.

  The Time Stamp Counter \(TSC\) is a 64-bit register present on all x86 processors since the Pentium. It counts the number of cycles since reset. The instruction RDTSC returns the TSC in EDX:EAX. In x86-64 mode, RDTSC also clears the higher 32 bits of RAX and RDX. Its opcode is 0F 31.\[1\] Pentium competitors such as the Cyrix 6x86 did not always have a TSC and may consider RDTSC an illegal instruction. Cyrix included a Time Stamp Counter in their MII.

  The Time Stamp Counter was once an excellent high-resolution, low-overhead way for a program to get CPU timing information. With the advent of multi-core/hyper-threaded CPUs, systems with multiple CPUs, and hibernating operating systems, the TSC cannot be relied upon to provide accurate results — unless great care is taken to correct the possible flaws: rate of tick and whether all cores \(processors\) have identical values in their time-keeping registers. There is no promise that the timestamp counters of multiple CPUs on a single motherboard will be synchronized. Therefore, a program can get reliable results only by limiting itself to run on one specific CPU. Even then, the CPU speed may change because of power-saving measures taken by the OS or BIOS, or the system may be hibernated and later resumed, resetting the TSC. In those latter cases, to stay relevant, the program must re-calibrate the counter periodically.

  Relying on the TSC also reduces portability, as other processors may not have a similar feature. Recent Intel processors include a constant rate TSC \(identified by the kern.timecounter.invariant\_tsc sysctl on FreeBSD or by the "constant\_tsc" flag in Linux's /proc/cpuinfo\). With these processors, the TSC ticks at the processor's nominal frequency, regardless of the actual CPU clock frequency due to turbo or power saving states. Hence TSC ticks are counting the passage of time, not the number of CPU clock cycles elapsed.

## _initialize schedule clock_

```sched\_clock\_init
void sched_clock_init(void)
{
    u64 ktime_now = ktime_to_ns(ktime_get());
    int cpu;

    for_each_possible_cpu(cpu) {
        struct sched_clock_data *scd = cpu_sdc(cpu);

        scd->tick_raw = 0;
        scd->tick_gtod = ktime_now;
        scd->clock = ktime_now;
    }

    sched_clock_running = 1;
}
```

`sched_init` gets per cpu schedule clock data variable and initializes the data, set flag `sched_clock_running` indicating schedule clock is running.

## _calculate loop\_per\_jiffy and print it in BogoMIPS_

For the boot cpu we can skip the delay calibration and assign `loop_per_jiffy` a value calculated based on the timer frequency.

```calibrate\_delay
start_kernel () at init/main.c:655
655        calibrate_delay();
(gdb) s
calibrate_delay () at init/calibrate.c:128
128        if (preset_lpj) {      
(gdb) p preset_lpj
$6 = 0
(gdb) n
133        } else if ((!printed) && lpj_fine) {
(gdb) p lpj_fine 
$7 = 13568640
(gdb) p loops_per_jiffy
$8 = 4096
134            loops_per_jiffy = lpj_fine;
(gdb) 
135            pr_info("Calibrating delay loop (skipped), "
(gdb) 
176        if (!printed)
(gdb) 
177            pr_cont("%lu.%02lu BogoMIPS (lpj=%lu)\n",
(gdb) 
181        printed = true;
(gdb) 
182    }
```

`lpj_fine` is initialized during `tsc_init`.

## _initialize pid map_

`pidmap_init` allocates page for pidmap, reserves PID 0 and creates slab cache for pid structure.

## _initialize anonymous VMA_

Anonymous pages contain the heap information associated with a process. These pages are called anonymous pages because there no associated file pages on-disk i.e. these pages do not map to a file. When the system is low on memory, these pages are removed from memory onto the swap space. Anonymous VMA stands for Anonymous Virtual Memory Address. Each process contains a list of virtual addresses that map to physical anonymous pages. If multiple processes use the same physical anonymous page, they will have different anonymous virtual memory addresses pointing to the same anonymous physical page\(s\).

The question is how do these anonymous pages work?

When a process is forked, it opens a bunch of files and associated file-backed pages. It may also create new anonymous pages or use existing ones. From within a process, one can get hold of all associated anonymous pages. This means that using the page tables, the kernel can establish the virtual address associated with the process and the real physical pages. In order to swap-in, the kernel needs to establish a reverse link i.e. which physical pages map to which processes. To facilitate, all pages contain a _reverse mapping_ mapping physical pages to virtual pages. Hence, when a process is forked these reverse mappings are created. If you want more details about the creation part, you can take a look at the file mm/rmap.c \(Linux Cross Reference\).  \_\_page\_set\_anon\_rmap and page\_add\_file\_rmap describe the anonymous and file-based page mappings, respectively.

Paging-in is the tricky part with anonymous pages:

1\) For pages backed by files, that need to be swapped back in, it is easy to search what files are opened by which processes/users and page them in/out. This is a very over-simplified explanation, in reality there are lots of details.

2\) For anonymous pages, this is tricky since there are no associated files \(or file based mappings\)\[2\]. Hence, the kernel uses a specific mechanism called as object based reverse-mapping. Here, instead of having a list of users or processes, the kernel stores what regions of memory this page was mapped onto, and when bringing them back in during page-in, they are simply restored to exactly those regions. This is a bit hacky but it gives very good performance.

`anon_vma_init` creates cache for anonymous virtual memory address.

## _initialize credentials_

Creates cache for credential, about linux credential, reference [here](https://www.kernel.org/doc/Documentation/security/credentials.txt)

## _initialize fork_

The defination for routine `fork_init` attached as follow.  
  `fork_init`:

* Creates slab cache for task\_struct named `task_struct`
* Creates slab cache for thread\_xstate named `task_xstate`
* Calculates number of max support thread
* Initializes signal handlers of `init_task`

```fork\_init
668        fork_init(totalram_pages);
(gdb) s
fork_init (mempages=27200) at kernel/fork.c:189
void __init fork_init(unsigned long mempages)
{
#ifndef __HAVE_ARCH_TASK_STRUCT_ALLOCATOR
#ifndef ARCH_MIN_TASKALIGN
#define ARCH_MIN_TASKALIGN    L1_CACHE_BYTES
#endif
    /* create a slab on which task_structs can be allocated */
    task_struct_cachep =
        kmem_cache_create("task_struct", sizeof(struct task_struct),
            ARCH_MIN_TASKALIGN, SLAB_PANIC | SLAB_NOTRACK, NULL);
#endif

    /* do the arch specific task caches init */
    arch_task_cache_init();

    /*
     * The default maximum number of threads is set to a safe
     * value: the thread structures can take up at most half
     * of memory.
     */
    max_threads = mempages / (8 * THREAD_SIZE / PAGE_SIZE);
(gdb) p max_threads 
$11 = 1700

    /*
     * we need to allow at least 20 threads to boot a system
     */
    if(max_threads < 20)
        max_threads = 20;

    init_task.signal->rlim[RLIMIT_NPROC].rlim_cur = max_threads/2;
    init_task.signal->rlim[RLIMIT_NPROC].rlim_max = max_threads/2;
    init_task.signal->rlim[RLIMIT_SIGPENDING] =
        init_task.signal->rlim[RLIMIT_NPROC];
}
```

## _initialize process cache_

`proc_caches_init` creates caches used when creating new process, initializes the VMA slab.

```proc\_caches\_init
void __init proc_caches_init(void)
{
    sighand_cachep = kmem_cache_create("sighand_cache",
            sizeof(struct sighand_struct), 0,
            SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_DESTROY_BY_RCU|
            SLAB_NOTRACK, sighand_ctor);
    signal_cachep = kmem_cache_create("signal_cache",
            sizeof(struct signal_struct), 0,
            SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_NOTRACK, NULL);
    files_cachep = kmem_cache_create("files_cache",
            sizeof(struct files_struct), 0,
            SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_NOTRACK, NULL);
    fs_cachep = kmem_cache_create("fs_cache",
            sizeof(struct fs_struct), 0,
            SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_NOTRACK, NULL);
    mm_cachep = kmem_cache_create("mm_struct",
            sizeof(struct mm_struct), ARCH_MIN_MMSTRUCT_ALIGN,
            SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_NOTRACK, NULL);
    vm_area_cachep = KMEM_CACHE(vm_area_struct, SLAB_PANIC);
    mmap_init();
}
```

`mmap_init` involves `percpu_counter_init` which is a macro, after macro expanded, `percpu_counter_init` shown as follow, `__percpu_counter_init` per cpu counter.

```percpu\_counter\_init
#define percpu_counter_init(fbc, value)                    \
    ({                                \
        static struct lock_class_key __key;            \
                                    \
        __percpu_counter_init(fbc, value, &__key);        \
    })
```

## _initializes buffer head_

Historically, a buffer\_head was used to map a single block within a page, and of course as the unit of I/O through the filesystem and block layers.  Nowadays the basic I/O unit is the bio, and buffer\_heads are used for extracting block mappings \(via a get\_block\_t call\), for tracking state within a page \(via a page\_mapping\) and for wrapping bio submission for backward compatibility reasons \(e.g. submit\_bh\).

`buffer_init` creates cache for buffer head named `buffer_head`, calculates `max_buffer_heads` and registers function to response cpu hot plugin.

```buffer\_init
 void __init buffer_init(void)
{
    int nrpages;

    bh_cachep = kmem_cache_create("buffer_head",
            sizeof(struct buffer_head), 0,
                (SLAB_RECLAIM_ACCOUNT|SLAB_PANIC|
                SLAB_MEM_SPREAD),
                init_buffer_head);

    /*
     * Limit the bh occupancy to 10% of ZONE_NORMAL
     */
    nrpages = (nr_free_buffer_pages() * 10) / 100;
    max_buffer_heads = nrpages * (PAGE_SIZE / sizeof(struct buffer_head));
(gdb) p nrpages 
$15 = 3218
    hotcpu_notifier(buffer_cpu_notify, 0);
(gdb) p max_buffer_heads 
$16 = 234914
}
```

## _initialise the key management stuff_

`key_init` creates cache for key named `key_jar`, struct key is used in authentication token/access credential/keyring. Adds key types to list `key_types_list`. Adds root user to rb tree and adjusts the color of the rb tree.

```key\_init
void __init key_init(void)
{
    /* allocate a slab in which we can store keys */
    key_jar = kmem_cache_create("key_jar", sizeof(struct key),
            0, SLAB_HWCACHE_ALIGN|SLAB_PANIC, NULL);

    /* add the special key types */
    list_add_tail(&key_type_keyring.link, &key_types_list);
    list_add_tail(&key_type_dead.link, &key_types_list);
    list_add_tail(&key_type_user.link, &key_types_list);

    /* record the root user tracking */
    rb_link_node(&root_key_user.node,
             NULL,
             &key_user_tree.rb_node);

    rb_insert_color(&root_key_user.node,
            &key_user_tree);

}
```

## _initialize the security framework_

`security_init` fixups all callback function in `default_security_ops` and initializes it to `security_ops`, involves initcall functions in code region of `SECURITY_INIT` which defined in include/asm-generic/vmlinux.lds.h:

```SECURITY\_INIT
#define SECURITY_INIT                            \
    .security_initcall.init : AT(ADDR(.security_initcall.init) - LOAD_OFFSET) { \
        VMLINUX_SYMBOL(__security_initcall_start) = .;        \
        *(.security_initcall.init)                 \
        VMLINUX_SYMBOL(__security_initcall_end) = .;        \
    }
```

```security\_init
start_kernel () at init/main.c:672
672        security_init();
(gdb) s
security_init () at security/security.c:54
54    {
(gdb) 
55        printk(KERN_INFO "Security Framework initialized\n");
(gdb) 
57        security_fixup_ops(&default_security_ops);
(gdb) 
58        security_ops = &default_security_ops;
(gdb) 
59        do_security_initcalls();
(gdb) s
do_security_initcalls () at security/security.c:42
42        while (call < __security_initcall_end) {
(gdb) n
43            (*call) ();
(gdb) p __security_initcall_start 
$22 = 0xc177e014 <__initcall_selinux_init>
(gdb) p __security_initcall_end
$23 = 0xc177e020
(gdb) p __security_initcall_start + 1
$24 = (initcall_t *) 0xc177e018 <__initcall_smack_init>
(gdb) p __security_initcall_start + 2
$25 = (initcall_t *) 0xc177e01c <__initcall_tomoyo_init>
(gdb) p __security_initcall_start + 3
$26 = (initcall_t *) 0xc177e020
```

## _initialize virtual file system cache_

Defination of routine `vfs_caches_init`:

```vfs\_caches\_init
    void __init vfs_caches_init(unsigned long mempages)
    {
        unsigned long reserve;

        /* Base hash sizes on available memory, with a reserve equal to
               150% of current kernel size */

        reserve = min((mempages - nr_free_pages()) * 3/2, mempages - 1);
        mempages -= reserve;

        names_cachep = kmem_cache_create("names_cache", PATH_MAX, 0,
                SLAB_HWCACHE_ALIGN|SLAB_PANIC, NULL);

        dcache_init();
        inode_init();
        files_init(mempages);
        mnt_init();
        bdev_cache_init();
        chrdev_init();
    }
```

`vfs_caches_init`

* Allocates slab cache for path named `names_cache`

* Involves routine `dcache_init` , which allocates slab cache for dirent data and registers `dcache_shrinker`

```
2333        dcache_init();
(gdb) s
dcache_init () at fs/dcache.c:2286
2286        dentry_cache = KMEM_CACHE(dentry,
(gdb) n
2289        register_shrinker(&dcache_shrinker);
(gdb) s
register_shrinker (
    shrinker=shrinker@entry=0xc16a9394 <dcache_shrinker>)
    at mm/vmscan.c:165
165    {
(gdb) n
166        shrinker->nr = 0;
(gdb) 
167        down_write(&shrinker_rwsem);
(gdb) 
168        list_add_tail(&shrinker->list, &shrinker_list);
(gdb) 
169        up_write(&shrinker_rwsem);
(gdb) 
170    }
(gdb) 
dcache_init () at fs/dcache.c:2292
2292        if (!hashdist)
(gdb) p hashdist 
$3 = 0
(gdb) n
```

* Involves routine `inode_init` , which allocates slab cache for inode data and registers`icache_shrinker`, the procedure is similar with `dcache_init` 
* Involves routine` files_init`,  which allocates slab cache for file data, initializes dynamically-tunable limits, initializes per cpu variable `fdtable_defer_list` and minimal number of file descriptor `sysctl_nr_open_max`, initializes per cpu counter

```
vfs_caches_init (mempages=27097) at fs/dcache.c:2335
2335		files_init(mempages);
(gdb) s
files_init (mempages=mempages@entry=27097) at fs/file_table.c:446
446		filp_cachep = kmem_cache_create("filp", sizeof(struct file), 0,
(gdb) 
454		n = (mempages * (PAGE_SIZE / 1024)) / 10;
(gdb) 
455		files_stat.max_files = n; 
(gdb) p n
$4 = 10838
(gdb) n
458		files_defer_init();
(gdb) s
files_defer_init () at fs/file.c:419
419		for_each_possible_cpu(i)
(gdb) 
420			fdtable_defer_list_init(i);
(gdb) s
fdtable_defer_list_init (cpu=0) at fs/file.c:410
410		struct fdtable_defer *fddef = &per_cpu(fdtable_defer_list, cpu);
(gdb) n
411		spin_lock_init(&fddef->lock);
(gdb) 
410		struct fdtable_defer *fddef = &per_cpu(fdtable_defer_list, cpu);
(gdb) 
411		spin_lock_init(&fddef->lock);
(gdb) 
412		INIT_WORK(&fddef->wq, free_fdtable_work);
(gdb) 
413		fddef->next = NULL;
(gdb) 
files_defer_init () at fs/file.c:419
419		for_each_possible_cpu(i)
(gdb) 
421		sysctl_nr_open_max = min((size_t)INT_MAX, ~(size_t)0/sizeof(void *)) &
(gdb) 
423	}
(gdb) p sysctl_nr_open_max
$5 = 1073741792
(gdb) n
files_init (mempages=mempages@entry=27097) at fs/file_table.c:459
459		percpu_counter_init(&nr_files, 0);
(gdb) s
__percpu_counter_init (fbc=fbc@entry=0xc1692060 <nr_files>, 
    amount=0, key=key@entry=0xc1e61f80 <__key.22296>)
    at lib/percpu_counter.c:72
72		spin_lock_init(&fbc->lock);
(gdb) 
73		lockdep_set_class(&fbc->lock, key);
(gdb) 
74		fbc->count = amount;
(gdb) 
75		fbc->counters = alloc_percpu(s32);
(gdb) 
76		if (!fbc->counters)
(gdb) 
79		INIT_LIST_HEAD(&fbc->list);
(gdb) 
80		mutex_lock(&percpu_counters_lock);
(gdb) 
81		list_add(&fbc->list, &percpu_counters);
(gdb) 
82		mutex_unlock(&percpu_counters_lock);
(gdb) 
84		return 0;
(gdb) 
85	}
(gdb) 
files_init (mempages=mempages@entry=27097) at fs/file_table.c:460
460	} 
(gdb) 

```

# Links

* [x86 Registers](http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html)
* [read-copy-update \(RCU\)](https://en.wikipedia.org/wiki/Read-copy-update)
* [Interrupt Handlers](http://www.tldp.org/LDP/lkmpg/2.4/html/x1210.html)
* [ISA interrupts versus PCI interrupts](https://www.safaribooksonline.com/library/view/pc-hardware-in/059600513X/ch01s03s01s01.html)
* [Interrupts](http://wiki.osdev.org/Interrupts)
* [Timers](www.cs.columbia.edu/~nahum/w6998/lectures/timers.ppt)
* [Kernel Timers](http://www.makelinux.net/ldd3/chp-7-sect-4)
* [Kernel Timer Systems](http://elinux.org/Kernel_Timer_Systems)
* [Timers and lists in the 2.6 kernel](https://www.ibm.com/developerworks/linux/library/l-timers-list/)
* [A new approach to kernel timers](https://lwn.net/Articles/152436/)
* [The high-resolution timer API](https://lwn.net/Articles/167897/)
* [Software interrupts and realtime](https://lwn.net/Articles/520076/)
* [timekeeping](https://www.kernel.org/doc/Documentation/timers/timekeeping.txt)
* [profiling](http://www.pixelbeat.org/programming/profiling/)
* [Profiling the kernel](http://homepages.cwi.nl/~aeb/linux/profile.html)
* [idr - integer ID management](https://lwn.net/Articles/103209/)
* [A simplified IDR API](https://lwn.net/Articles/536293/)
* [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [How do anonymous VMAs work in Linux?](https://www.quora.com/How-do-anonymous-VMAs-work-in-Linux)
* [The object-based reverse-mapping VM](https://lwn.net/Articles/23732/)
* [Reverse mapping anonymous pages - again](https://lwn.net/Articles/77106/)
* [CREDENTIALS IN LINUX](https://www.kernel.org/doc/Documentation/security/credentials.txt)




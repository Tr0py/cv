---
layout: post
title:  "操作系统lab4学习笔记"
date:   2018-12-26 18:51:59 +0800
tags:
- TechNotes
- 笔记
- Technical Notes
---

## 1.看程序

### kern/mpconfig.c
```C++
    unsigned char percpu_kstacks[NCPU][KSTKSIZE]
    __attribute__ ((aligned(PGSIZE)));
```
这里attribute aligned是对齐的意思。那么aligned(PGSIZE)就是对齐到页了。

## Part A: Multiprocessor Support and Cooperative Multitasking
### Multiprocessor Support

#### **Exercise1**

推测函数作用：

* 返回值void* 输入物理地址pa 输入**未对齐的** size
* **作用：**将[pa,pa+size)映射到从base开始的空间上，连续的。返回当前用到了哪里（虚拟地址）
* 检查越界(8 9 10)
* 对齐页面(6 7)
* 使用boot_map_region进行映射
* base类似分配页面的next，持续更改上一次用到了哪里，同时也是函数返回想要的值(12 13)
* 要把va和size对齐到页面，因为**boot_map_region没有作round，默认是对齐了的**

> 看一下boot_map_region(pgdir,va,size,pa,perm)，可知对应参数为kern_pgdir,base,size,pa,PTE_PCD|PTE_PWT|PTE_W).

```c++
void *
mmio_map_region(physaddr_t pa, size_t size)
{
    static uintptr_t base = MMIOBASE;
    //panic("mmio_map_region not implemented");
    size_t roundSize=ROUNDUP(size,PGSIZE);
    physaddr_t roundPa=ROUNDDOWN(pa,PGSIZE);
    if(base+size>MMIOLIM) {
        panic("mmio_map_region: size overflows MMIOLIM!");
    }
    boot_map_region(kern_pgdir,base,roundSize,roundPa,PTE_PCD|PTE_PWT|PTE_W);
    base+=roundSize;
    return (void*)base;
}

```
之后发现一个bug，是对于返回值return的理解的要求错了。
> Return the base of the reserved region.

其实应该是base+=roundSize前的base！
所以第13行应该为：
```
    return (void*)(base-roundSize);
```
### Application Processor Bootstrap
#### **Exercise2**
在kern\mapentry.S中
```
# Specification says that the AP will start in real mode with CS:IP
# set to XY00:0000, where XY is an 8-bit value sent with the
# STARTUP. Thus this code must start at a 4096-byte boundary.
#
# Because this code sets DS to zero, it must run from an address in
# the low 2^16 bytes of physical memory.
```
大概又看了一下汇编，大概就是CS<<4|IP组成当前指令，比如CS=2AE3H，IP=0003H,CPU将在2AE33H处读取指令。（事实上保护模式下应该是查表后的值<<4得到的值相加）那么XY00:0000就是XY000H可以寻址2^12=4096byte的一段了。同时DS(data segment)SS等段选择寄存器，是段寻址模式。<a href="https://www.cnblogs.com/rixiang/p/5451130.html">GDT,LDT,GDTR,LDTR 详解</a>这里解释的比较清楚。至于DS为0但是CS不为零啊，为什么会有这个影响还不清楚。


啊，理解了。分为real mode 和protected mode，两者由于寄存器位数的不同，造成的寻址模式也不同。16位(real mode)：cs:ip=cs<<4 | ip，32位(protected mode)：cs为了向下兼容还是当成16bit用，为了匹配32位模式使用了GDT所以变成了查GDT了。<a href="https://blog.csdn.net/yeruby/article/details/39718119">GDT、GDTR、LDT、LDTR的学习</a>
```
# Call mp_main().  (Exercise for the reader: why the indirect call?)
movl    $mp_main, %eax
call    *%eax
```
这是为什么呢？

构造GDT：
```
gdt:
    SEG_NULL                # null seg
    SEG(STA_X|STA_R, 0x0, 0xffffffff)   # code seg
    SEG(STA_W, 0x0, 0xffffffff)     # data seg

gdtdesc:
    .word   0x17                # sizeof(gdt) - 1
    .long   MPBOOTPHYS(gdt)         # address gdt

.globl mpentry_end
```
不是很明白。
看到SEG的定义
```
#define SEG_NULL                        \
    .word 0, 0;                     \
    .byte 0, 0, 0, 0
#define SEG(type,base,lim)                  \
    .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);  \
    .byte (((base) >> 16) & 0xff), (0x90 | (type)),     \
        (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)

```
GDT表项的结构：
<img src="http://ww1.sinaimg.cn/mw690/bff4f9baly1fybb1emm0cj20n10673z9.jpg"/>
明白了！

由于mpentry.S的代码被映射到MPENTRY_PADDR上，对应的地址为0x7000,属于basemem中，所以加一个判断即可。
```
    if(i==MPENTRY_PADDR/PGSIZE)
    {
        pages[i].pp_ref=1;
        pages[i].pp_link=NULL;
        continue;
    }
```
checkpagefreelist通过。

#### **Question**
为什么在mpentry.S中需要用到直接的偏移量计算呢？查看.S后发现这个宏只用在.code16段中。
去掉宏，尝试运行，报错：
> relocation truncated to fit: R_386_16 against `.text'
查stackoverflow，大概是
>  using a label as a 16bit immediate, but linking as if it was 32bit code. 

也就是受到了16bit的限制。考虑报错，看到报错位置是kernel.o，也就是和kernel被加载到一起了，是KERNBASE以上的，如果直接寻址必然超过2^20=1MB的限制，所以通过宏进行计算相对位置跳转！而且，是使用目标-函数开始位置+物理内存位置，计算出来的是物理内存的代码段是相对于0x7000处的，小于1MB可寻址。但是为什么boot.S可以直接用呢？大概是因为boot中还没有开启分页机制且代码在低地址上运行。

### Per-CPU State and Initialization
#### **Exercise3**
就是KSTKSIZE和KSTCGAP组成一个CPU的堆栈区，循环赋值就可以了。
```
static void
mem_init_mp(void)
{
    int i=0;
    uintptr_t KSTACKTOPi;
    for (i=0;i<NCPU;i++)
    {
        KSTACKTOPi=KSTACKTOP-i*(KSTKSIZE+KSTKGAP);
        boot_map_region(kern_pgdir,KSTACKTOPi-KSTKSIZE,KSTKSIZE,PADDR(&percpu_kstacks[i]),PTE_W);
    }
}
```
就是不知道percpu_kstacks是在哪里被赋值的，全局查找也没有找到被赋值的语句，觉得很神奇。

只在kernel.sym找到了各种符号的地址，比较神奇，原来之前的obj找不到入口的报错是从这里来的
```
f020c3c0 B bootcpu
f020c3c4 B ncpu
f020d000 B percpu_kstacks
f0204760 D _binary_obj_user_testshell_start
f020a6fc D _binary_obj_user_testshell_end
f020a6fc D edata
```
#### **Exercise4**
替换变量，加偏移量。
```
void
trap_init_percpu(void)
{
    int i = thiscpu->cpu_id;
    thiscpu->cpu_ts.ts_esp0 = KSTACKTOP - i*(KSTKSIZE+KSTKGAP);
    thiscpu->cpu_ts.ts_ss0 = GD_KD;
    gdt[(GD_TSS0 >> 3) + i] = SEG16(STS_T32A, (uint32_t) (&(thiscpu->cpu_ts)),
                    sizeof(struct Taskstate) - 1, 0);
    gdt[(GD_TSS0 >> 3) + i].sd_s = 0;
    // Load the TSS selector (like other segment selectors, the
    // bottom three bits are special; we leave them 0)
    ltr(GD_TSS0+8*i);
    lidt(&idt_pd);
}
```
上面都是一路替换，难点在于第12行，重新看了一下GDT的构造。
```
struct Segdesc gdt[NCPU + 5] =
{
    SEG_NULL,
    [GD_KT >> 3] = SEG(STA_X | STA_R, 0x0, 0xffffffff, 0),
    [GD_KD >> 3] = SEG(STA_W, 0x0, 0xffffffff, 0),
    [GD_UT >> 3] = SEG(STA_X | STA_R, 0x0, 0xffffffff, 3),
    [GD_UD >> 3] = SEG(STA_W, 0x0, 0xffffffff, 3),
    [GD_TSS0 >> 3] = SEG_NULL
};
```
为什么会>>3呢？根据定义，

    #define GD_KT     0x08     // kernel text
    #define GD_KD     0x10     // kernel data
    #define GD_UT     0x18     // user text
    #define GD_UD     0x20     // user data
    #define GD_TSS0   0x28     // Task segment selector for CPU 0

把它转化成二进制：

    001000
    010000
    011000
    100000
    101000
    .....

就很明显的看出来，实际上每次在第四位上+1，即十进制+8！为什么是+8呢？考虑到GDT表项的结构，每个表项是64bit，就是8个字节！所以其实是因为GDT表项的大小是8字节，对应的二进制第三位为零，恰好作为索引了！
所以在这里第12行要每次+8。
### Locking
#### **Exercise5**
就是在各个地方加锁。
之后给人讲题的时候理顺了好几遍，大概是这样的：

1. unlock_kernel()放锁存在2个地方：env_run()中和halt()中，就是离开内核开始运行的时候。
2. 内核锁存在的3个地方：i386_init的BSP在启动ap前，mp_main()中（或者说AP启动的最后一步）sched_yield()之前，还有就是trap。
3. 前两个大内核锁的作用是**在所有CPU启动完全后再统一开始执行env**，因为BSP锁住了内核，举个例子，此时挨着启动AP，我们设CPU0=BSP,CPU1=AP1,CPU2=AP2,CPU3=AP3，那么现在如果BSP启动完了AP1,现在在启动AP2的过程中，此时AP1应该等待，不应该运行。那么究竟是哪里阻塞了它呢？就是mp_main()中的lockkernel(),也就是说，BSP:"大家都不要动，我先动"，因为在i386_init()中在启动ap之前就拿好了锁，在启动完AP后最终执行到schel_yield()中的env_run()开始放锁(见1)，此时各个AP都一直被阻塞在mp_main()的lockkernel状态，此时开始抢锁，谁抢到谁再调度。
4. trap()中的内核锁就是为了每只处理一个中断，且因为这个，同时刻只可能有1个CPU在yield()！
### Round-Robin Scheduling
#### **Exercise6**
先看看这个函数怎么用的。
1.僵尸环境，调用的时候curenv=NULL;
```
    if (curenv->env_status == ENV_DYING) {
        env_free(curenv);
        curenv = NULL;
        sched_yield();
    }
```
2.trap处理完之后，CURENV可能为NULL或此时status!=running;
```
    if (curenv && curenv->env_status == ENV_RUNNING)
        env_run(curenv);
    else
        sched_yield();
```
3.用户主动调用，此时应该是curenv存在，status==running;
```
static void
sys_yield(void)
{
    sched_yield();
}
```
4.mp_main()中，即初始化完AP后调用，此时找事情做（此时的curenv是什么？status呢？）
5.i386_init()中，初始化完后运行第一个用户环境。（此时的curenv是什么？status呢？）
6.env_destroy()中，此时curenv==NULL;
```
    if (curenv == e) {
        curenv = NULL;
        sched_yield();
    }
```

于是，这里的思路就是找到一个read的env运行，
A.当curenv为NULL时从头开始遍历env
B.当curenv不为NULL时从下一个开始遍历env
当没有找到一个runnable的环境时
A.如果自身是正在运行的，那么运行自身
B.halt

```
void
sched_yield(void)
{
    struct Env *idle;
    idle = thiscpu->cpu_env;
    int i;
    if (idle) {
        i = ENVX(idle->env_id);
    }
    else {
        i = 0;
    }
    int start = i;
    if (envs[i].env_status == ENV_RUNNABLE) {
        env_run(&envs[i]);
    }
    for (i++;start!=i;i=(i+1)%NENV) {
        if (envs[i].env_status == ENV_RUNNABLE) {
            env_run(&envs[i]);
        }
    }
    if (idle && (idle->env_status)) {
        env_run(idle);
    }

    // sched_halt never returns
    sched_halt();
}
```
运行时发现总是触发General protection，但是多进程便可以。而且多进程总是从10001开始执行，且第一个进程无法被执行，即总是跳过第一个进程。
其实从当前进程的下一个进程开始是没错的，缺少的判断是当idle==NULL的时候的第0个进程！在原来的代码中没有判断，是start的取值的问题。即应该从0而不是i++开始。因为刚开始的时候cpuenv必然是空的，即应该从第0号开始。恰好漏了这种情况，通过start和i的赋值恰好处理了。所以正确的是这样的，而且这样的还简洁！

这里不先把idle置为RUNNABLE就是因为一会会回来执行，在这段查找时间**不能被其他CPU抢占运行**，所以保留RUNNING状态！
其实不会被抢占运行的，因为上面分析过了，不会出现两个CPU同时yield()的情况（大内核锁）。而且也不用手动置为RUNNABLE，因为在env_run()中就会去做这件事情。
```
void
sched_yield(void)
{
    struct Env *idle;

    // Implement simple round-robin scheduling.
    //
    // Search through 'envs' for an ENV_RUNNABLE environment in
    // circular fashion starting just after the env this CPU was
    // last running.  Switch to the first such environment found.
    //
    // If no envs are runnable, but the environment previously
    // running on this CPU is still ENV_RUNNING, it's okay to
    // choose that environment.
    //
    // Never choose an environment that's currently running on
    // another CPU (env_status == ENV_RUNNING). If there are
    // no runnable environments, simply drop through to the code
    // below to halt the cpu.

    // LAB 4: Your code here.
    
    idle = thiscpu->cpu_env;
    int i;
    int start;
    if (idle) {
        i = ENVX(idle->env_id);
        start=i+1;
    //  if (idle->env_status==ENV_RUNNING) {
    //  idle->env_status=ENV_RUNNABLE;
    // }
    }
    else {
        i = 0;
        start=i;
    }

    int count=0;
    for (;count++<NENV;i=(i+1)%NENV) {
        if (envs[i].env_status == ENV_RUNNABLE) {
            env_run(&envs[i]);
        }
    }
    // if ((!i)&&(envs[i].env_status == ENV_RUNNABLE)) {
    //  env_run(&envs[i]);
    // }
    if ((idle)&&(idle->env_status==ENV_RUNNING)) {
        env_run(idle);
    }

    // sched_halt never returns
    sched_halt();
}
```
然后改syscall()
```
            case (SYS_yield):
                    cprintf("yielding\n");
                    sys_yield();
                    return 0;
```
加进程
```
#if defined(TEST)
    // Don't touch -- used by grading script!
    ENV_CREATE(TEST, ENV_TYPE_USER);
#else
    // Touch all you want.
    //ENV_CREATE(user_hello, ENV_TYPE_USER);
    ENV_CREATE(user_yield,ENV_TYPE_USER);
    ENV_CREATE(user_yield,ENV_TYPE_USER);
    ENV_CREATE(user_yield,ENV_TYPE_USER);
    ENV_CREATE(user_yield,ENV_TYPE_USER);
    ENV_CREATE(user_yield,ENV_TYPE_USER);
#endif // TEST*
```
尝试几次后，发现总会触发General Protection，原来是正确的问题，在没有进程的时候触发的。
部分输出：
> Back in environment 00001003, iteration 0.
> T_SYSCALL
> Back in environment 00001004, iteration 1.
> T_SYSCALL
> yielding
> T_SYSCALL
> yielding
> T_SYSCALL
> Back in environment 00001000, iteration 0.

#### **Question3**
```
    curenv=e;
    e->env_status=ENV_RUNNING;
    e->env_runs++;
    lcr3(PADDR(e->env_pgdir));
    //step2
    unlock_kernel();
    env_pop_tf(&e->env_tf);
```
由于各个pgdir的UTOP上面都一样(除了UVPT是自己的pgdir地址)，所以envinfo那一部分也一样，所以可以正常解引用。
#### **Question4**
保存在tf里，在发生中断的时候trap函数中保存了trapframe，在陷入内核的时候保存了trapframe的位置。用于恢复。
### System Calls for Environment Creation
#### **Exercise7**
这里的过程是这样的：

先做一个蠢的慢速dumbfork(),再做一个睿智的fork()
两者的区别是：

* dumbfork():容易想到易于实现，效率低，全拷贝。考虑直接的fork()，就是把**寄存器状态**和**堆栈、数据区(代码、数据...)**一样就可以完全一样了。所以我们考虑的就是1.寄存器-trapframe，直接两者相等赋值就好了。2.数据-那就是pgdir上的内容，那么完全地拿一份新的页面ctrl+c ctrl+v复制粘贴就好了，之后的事情两不影响，自己修改执行。
    * dumbfork就是这样的一个函数，它把tf置为相等（这一步都是一样的），对于空间，我们设env A 去fork一个env B，就创建一个ENV B，每次把A的一页拿出来放到B的UTEMP上，再为B分配一个新的页面，把UTEMP上的内容memcpy过去，之后卸下UTEMP上的页面。这个页面的物理页面的过程：属于A的addr->**被映射在B的utemp上**->映射到A的addr和B的UTEMP位置->**B拷贝完成后卸载页面**->属于A的addr。此时B的addr位置有了一个新的物理页与之对应。
* fork():拷贝代价高，而且拷贝后有可能根本不去修改，所以用一个物理页只要不修改就没关系。当修改的时候再创建一个新的物理页就行了！ 
```
static envid_t
sys_exofork(void)
{
    // Create the new environment with env_alloc(), from kern/env.c.
    // It should be left as env_alloc created it, except that
    // status is set to ENV_NOT_RUNNABLE, and the register set is copied
    // from the current environment -- but tweaked so sys_exofork
    // will appear to return 0.

    // LAB 4: Your code here.
    //panic("sys_exofork not implemented");
    struct Env* newe;
    int ret = env_alloc(&newe,curenv->env_id);
    if (ret<0) return ret;
    newe->env_status=ENV_NOT_RUNNABLE;
    newe->env_tf=curenv->env_tf;
    newe->env_tf.tf_regs.reg_eax=0;
    return newe->env_id;
}
```

```
static int
sys_env_set_status(envid_t envid, int status)
{
    // Hint: Use the 'envid2env' function from kern/env.c to translate an
    // envid to a struct Env.
    // You should set envid2env's third argument to 1, which will
    // check whether the current environment has permission to set
    // envid's status.

    // LAB 4: Your code here.
    //panic("sys_env_set_status not implemented");
    if ((status!=ENV_RUNNABLE)&&(status!=ENV_NOT_RUNNABLE)) {
        return -E_INVAL;
    }
    struct Env* theenv;
    int ret = envid2env(envid,&theenv,1);
    if (ret!=0) return ret;
    theenv->env_status=status;
    return 0;
}
```

```
static int
sys_page_alloc(envid_t envid, void *va, int perm)
{
    // Hint: This function is a wrapper around page_alloc() and
    //   page_insert() from kern/pmap.c.
    //   Most of the new code you write should be to check the
    //   parameters for correctness.
    //   If page_insert() fails, remember to free the page you
    //   allocated!

    // LAB 4: Your code here.
    //panic("sys_page_alloc not implemented");
    struct Env* the_env;
    int ret = envid2env(envid,&the_env,1);
    if (ret<0) return -E_BAD_ENV;
    if (va>=UTOP||((va%PGSIZE)!=0)) return -E_INVAL;
    if (((perm&(PTE_P|PTE_U))!=(PTE_P|PTE_U))||((perm|PTE_SYSCALL)!=PTE_SYSCALL)) return -E_INVAL;
    struct PageInfo *p=page_alloc(ALLOC_ZERO);
    if (!p) return -E_NO_MEM;
    ret = page_insert(the_env->env_pgdir,p,va,perm);
    if (ret<0) {
        page_free(p);
        return -E_NO_MEM;
    }
    return 0;
}
```

```
static int
sys_page_map(envid_t srcenvid, void *srcva,
         envid_t dstenvid, void *dstva, int perm)
{
    // Hint: This function is a wrapper around page_lookup() and
    //   page_insert() from kern/pmap.c.
    //   Again, most of the new code you write should be to check the
    //   parameters for correctness.
    //   Use the third argument to page_lookup() to
    //   check the current permissions on the page.

    // LAB 4: Your code here.
    //panic("sys_page_map not implemented");
    struct Env *s_env,*d_env;
    int ret = envid2env(srcenvid,&s_env,1);
    if (ret<0) return -E_BAD_ENV;
    ret = envid2env(dstenvid,&d_env,1);
    if (ret<0) return -E_BAD_ENV;
    if (srcva>=UTOP||((srcva%PGSIZE)!=0)) return -E_INVAL;
    if (dstva>=UTOP||((dstva%PGSIZE)!=0)) return -E_INVAL;
    pte_t pte;
    struct PageInfo* p = page_lookup(s_env->env_pgdir,srcva,&pte);
    if (!p) return -E_INVAL;
    if (((perm&(PTE_P|PTE_U))!=(PTE_P|PTE_U))||((perm|PTE_SYSCALL)!=PTE_SYSCALL)) return -E_INVAL;
    if ((((*pte)&PTE_W)==0)&&(perm&PTE_W)) return -E_INVAL;
    ret = page_insert(d_env->env_pgdir,p,dstva,perm);
    if (ret<0) return ret;
    return 0;
}
```
都是按照注释写的。
最后增加sys_call()，补全。
运行时发现总是触发General protection，但是多进程便可以。而且多进程总是从10001开始执行，且第一个进程无法被执行，即总是跳过第一个进程。
其实从当前进程的下一个进程开始是没错的，缺少的判断是当idle==NULL的时候的第0个进程！在原来的代码中没有判断，是start的取值的问题。即应该从0而不是i++开始。因为刚开始的时候cpuenv必然是空的，即应该从第0号开始。恰好漏了这种情况，通过start和i的赋值恰好处理了。
运行make run-dumbfork()，成功。

挑战 终于写出来了。网上没有找到些这个挑战的我觉得挺有意思的，就是保存快照再恢复。
保存trapframe和env的页目录上的东西就行了
思路很简单但是写了好久好久
最重要的是，新建一个env留着用，单纯备份，此时**将拍摄快照的env置为PTE_COW，即让被拍摄快照的env写时复制**，这样备份用的env就可以留着还原。需要还原时，按照目录page_insert一遍就可以了！
注意修改lib.h等函数，要自己写一个c程序。
```C++
static int
sys_snap(envid_t envid)
{
    struct Env* e,*tmpenv;
    extern unsigned char end[];
    if ((envid2env(envid,&e,0))<0) {
        panic("sys_snap: bad env!\n");
    }
    curenv->snap_env_tf = e->env_tf;
    cprintf("snaped.env%d\n",e->env_id);
    print_trapframe(&(curenv->snap_env_tf));
    envid_t subenvid=sys_exofork();
    cprintf("sub envid %d\n",subenvid);
    uint32_t *addr;
    int i,r;
    if (subenvid<0) {
        panic("snap: exofork err\n");
    }
    envid2env(subenvid,&tmpenv,0);//tmpenv 

    struct PageInfo* p ;
    pte_t *pte;
    int perm;
    for (i=0;i<UXSTACKTOP-PGSIZE;i+=PGSIZE) {
        perm=0;
        p = page_lookup(e->env_pgdir,(void*)i,&pte);
        if (!p) continue;
        cprintf("oh ye\n");
        perm |= ((*pte)& (PTE_P|PTE_W|PTE_U|PTE_SYSCALL|0x800));
        int r = page_insert(tmpenv->env_pgdir,p,(void*)i,perm);
        if (r<0) panic("snap failed:%e\n");
        r = page_insert(e->env_pgdir,p,(void*)i,PTE_P|PTE_U|0x800);
        if (r<0) panic("snap failed:%e\n");
    }
    curenv->snap_env=subenvid;
    return 0; 

}

static void
sys_recover(envid_t *envid)
{
    struct Env* e,*tmpenv;
    if ((envid2env(*envid,&e,0))<0) {
        panic("sys_recover: bad env!\n");
    }
    memcpy(&(e->env_tf),&(curenv->snap_env_tf),sizeof(struct Trapframe));
    cprintf("recovering.env%d\n",e->env_id);
    if (envid2env(curenv->snap_env,&tmpenv,0)<0) {
        panic("sys_recover: bad env!\n");
    }
    struct PageInfo* p ;
    pte_t *pte;
    int perm;
    int i,r;
    for (i=0;i<USTACKTOP;i+=PGSIZE) {
        perm=0;
        p = page_lookup(tmpenv->env_pgdir,(void*)i,&pte);
        if (!p) continue;
        cprintf("oh ye%p\n",i);
        perm |= ((*pte)& (PTE_P|PTE_W|PTE_U|PTE_SYSCALL));
        int r = page_insert(e->env_pgdir,p,(void*)i,perm);
        if (r<0) panic("recover failed:%e\n");
    }
    tmpenv->env_tf = curenv->snap_env_tf;
    e->env_tf = curenv->snap_env_tf;
    sched_yield();
}
```

c程序
```C++
// Test Conversation between parent and child environment
// Contributed by Varun Agrawal at Stony Brook

#include <inc/lib.h>

const char *str1 = "hello child environment! how are you?";
const char *str2 = "hello parent environment! I'm good.";

#define TEMP_ADDR   ((char*)0xa00000)
#define TEMP_ADDR_CHILD ((char*)0xb00000)
#define NORMAL2

void
umain(int argc, char **argv)
{
#ifdef NORMAL
    envid_t who;

    if ((who = fork()) == 0) {
        // Child
        ipc_recv(&who, TEMP_ADDR_CHILD, 0);
        cprintf("%x got message: %s\n", who, TEMP_ADDR_CHILD);
        if (strncmp(TEMP_ADDR_CHILD, str1, strlen(str1)) == 0)
            cprintf("child received correct message\n");

        memcpy(TEMP_ADDR_CHILD, str2, strlen(str2) + 1);
        ipc_send(who, 0, TEMP_ADDR_CHILD, PTE_P | PTE_W | PTE_U);
        return;
    }

    // Parent
    sys_page_alloc(thisenv->env_id, TEMP_ADDR, PTE_P | PTE_W | PTE_U);
    memcpy(TEMP_ADDR, str1, strlen(str1) + 1);
    ipc_send(who, 0, TEMP_ADDR, PTE_P | PTE_W | PTE_U);

    ipc_recv(&who, TEMP_ADDR, 0);
    cprintf("%x got message: %s\n", who, TEMP_ADDR);
    if (strncmp(TEMP_ADDR, str2, strlen(str2)) == 0)
        cprintf("parent received correct message\n");
    return;
#endif
#ifndef NORMAL
    envid_t who;
    int i=1;
    if ((who = fork()) == 0) {
        while(i++) {

            cprintf("I am child, counting %d\n",i);
        }
    }
    cprintf("I am the parent.\n");
    sys_yield();
    cprintf("I am the parent.  snapping the child...\n");
    sys_snap(who);
    sys_yield();
    // cprintf("I am the parent.  snapping the child...\n");
    // sys_snap(who);
    // sys_yield();
    cprintf("I am the parent.  recovering the child...\n");
    sys_recover(&who);
    //sys_yield();
    cprintf("I am pt, counting %d\n",i);
    cprintf("I am the parent.  Killing the child...\n");
    sys_yield();
    sys_env_destroy(who);
#endif
}
```
应当输出
```C++
I am child, counting 12
I am child, counting 13
I am child, counting 14
I am the parent.  snapping the child...
snaped.env4097
TRAP frame at 0xf0297044 from CPU 0
  edi  0x00000000
  esi  0x00000000
  ebp  0xeebfde50
  oesp 0xefffffdc
  ebx  0x00000000
  edx  0xeebfde78
  ecx  0x00000018
  eax  0x00000000
  es   0x----0023
  ds   0x----0023
  trap 0x00000020 Hardware Interrupt
  err  0x00000000
  eip  0x00800b6f
  cs   0x----001b
  flag 0x00000296
  esp  0xeebfde44
  ss   0x----0023
[00001000] new env 00001002
sub envid 4098
oh ye
oh ye
oh ye
oh ye
oh ye
I am child, counting 15
I am child, counting 16
I am child, counting 17
I am child, counting 18
I am child, counting 19
I am child, counting 20
I am child, counting 21
I am child, counting 22
I am child, counting 23
I am child, counting 24
I am child, counting 25
I am child, counting 26
I am child, counting 27
I am child, counting 28
I am child, counting 29
I am the parent.  recovering the child...
recovering.env4097
oh ye0x7ff000
oh ye0x800000
oh ye0x801000
oh ye0x802000
oh ye0xeebfd000
I am child, counting 15
I am child, counting 16
I am child, counting 17
I am child, counting 18
I am child, counting 19
I am child, counting 20
I am child, counting 21
I am child, counting 22
```
## Part B: Copy-on-Write Fork
### Setting the Page Fault Handler
#### **Exercise8**
```
static int
sys_env_set_pgfault_upcall(envid_t envid, void *func)
{
    // LAB 4: Your code here.
    //panic("sys_env_set_pgfault_upcall not implemented");
    struct Env* the_env;
    int ret = envid2env(envid,&the_env,1);
    if (ret<0) return ret;
    the_env->env_pgfault_upcall=func;
    return 0;
}
```


### Invoking the User Page Fault Handler
#### **Exercise9**
有一个神图来解释发生了什么：
<img src="http://ww1.sinaimg.cn/mw690/bff4f9baly1fydh8lv73nj20fo0emtfs.jpg"/>




















简而言之，发生缺页中断的时候，依然会陷入内核态，只不过在dispatch之后会额外的操作，进行handle之后转移到用户异常堆栈中。

* A.发生在用户态：用户态(用户堆栈)->内核态(内核堆栈)->用户异常处理(用户异常堆栈)
* B.发生在用户异常态：用户异常态（用户异常堆栈）->内核态（内核堆栈）->用户异常处理（用户异常态）

其中有一些不同，即用户异常态需要在上一个utf下垫一个空32位，返回的时候用，具体为什么不太明白。
你可以通过检测它的值是不是在 UXSTACKTOP-PGSIZE 和 UXSTACKTOP-1 （包含边界）之间，来判断 tf->tf_esp 是否已经在用户异常堆栈上。esp是当前调用的栈底，检查esp就可以了。（8-14行）为什么不包括UXSTACKTOP呢？
然后根据UTrapframe赋值就好了。最后调整eip到函数入口，运行即可。
trap.c/

```
void
page_fault_handler(struct Trapframe *tf)
{
    ...
    if (curenv->env_pgfault_upcall) {
        struct UTrapframe* u_tf;
        size_t gap=0;
        if ((tf->tf_esp>=(UXSTACKTOP-PGSIZE))&&(tf->tf_esp<UXSTACKTOP)) {
            gap = 4;
        }
        else {
            gap = 0;
        }
        u_tf = tf->tf_esp - sizeof(struct UTrapframe) - gap;
        user_mem_assert(curenv,u_tf,sizeof(struct UTrapframe),(PTE_W|PTE_U));
        u_tf->utf_esp=tf->tf_esp;
        u_tf->utf_eflags=tf->tf_eflags;
        u_tf->utf_eip=tf->tf_eip;
        u_tf->utf_regs=tf->tf_regs;
        u_tf->utf_err=tf->tf_err;
        u_tf->utf_fault_va=fault_va;
        curenv->env_tf.eip=(uint32_t)curenv->env_pgfault_upcall;
        curenv->env_tf.esp=(uint32_t)u_tf;
        env_run(curenv);
    }
    ...
}
```


### User-mode Page Fault Entrypoint
#### **Exercise10**
```
_pgfault_upcall:
    // Call the C page fault handler.
    pushl %esp          // function argument: pointer to UTF
    movl _pgfault_handler, %eax
    call *%eax
    addl $4, %esp
    movl 0x30(%esp),%eax
    subl $0x4,%eax
    movl %eax,0x30(%esp)
    movl 0x28(%esp),%ebx
    movl %ebx,(%eax)

    addl $0x8,%esp
    popal

    addl $0x4,%esp
    popfl

    pop %esp
    ret
```
实在是妙啊，从一开始就是为了最后的ret。
感觉最神奇的一点是利用了esp-4的位置： **esp-4就是陷入这次异常前的最后位置-4，即下一个位置** 1.如果是在异常栈引发的，那么，就是那个留出来的空位2.如果是在用户栈引发的，那么esp+4是**用户栈陷入前的堆栈位置-4**，也就是eip直接放到了用户栈中！在pop esp后esp指向了用户栈，再ret (=pop eip)就把eip还原且esp又+了4，就完全还原了！
妙啊
#### **Exercise11**
```
void
set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
{
    int r;

    if (_pgfault_handler == 0) {
        // First time through!
        // LAB 4: Your code here.
        //panic("set_pgfault_handler not implemented");
        r = sys_page_alloc(0,(void*)(UXSTACKTOP-PGSIZE),PTE_W|PTE_U|PTE_P);
        if (r<0) 
            panic("set_pgfault_handler: sys_page_alloc failed! %e\n",r);
        sys_env_set_pgfault_upcall(0,_pgfault_upcall);
    }

    // Save handler pointer for assembly to call.
    _pgfault_handler = handler;
}
```
### Implementing Copy-on-Write Fork
#### **Exercise12**

> If the page is writable or copy-on-write,
// the new mapping must be created copy-on-write, and then our mapping must be
// marked copy-on-write as well.  (Exercise: Why do we need to mark ours
// copy-on-write again if it was already copy-on-write at the beginning of
// this function?)

为什么啊，这里是有点歧义吧，其实是如果是COW那么之后也应该是COW，因为是子进程的子进程。
## Part C: Preemptive Multitasking and Inter-Process communication (IPC)
### Clock Interrupts and Preemption
#### **Exercise13**
这里就是像lab3一样注册trapentry
查看trap.h，这些被定义在trap.h中（找了好久picirq.h/.c）原来是在trap中
```C++
#define IRQ_OFFSET  32  // IRQ 0 corresponds to int IRQ_OFFSET
// Hardware IRQ numbers. We receive these as (IRQ_OFFSET+IRQ_WHATEVER)
#define IRQ_TIMER        0
#define IRQ_KBD          1
#define IRQ_SERIAL       4
#define IRQ_SPURIOUS     7
#define IRQ_IDE         14
#define IRQ_ERROR       19
```
所以对应的写就行了，有几个注意点题目没有说清楚：

* 是没有错误码的
* DPL为0

由于我们在lab3写了挑战，所以这里直接用了简易的写法
```C++
    MTRAPHANDLER(t_timer,IRQ_OFFSET+IRQ_TIMER,1,0)
    MTRAPHANDLER(t_kbd,IRQ_OFFSET+IRQ_KBD,1,0)
    MTRAPHANDLER(t_serial,IRQ_OFFSET+IRQ_SERIAL,1,0)
    MTRAPHANDLER(t_spurious,IRQ_OFFSET+IRQ_SPURIOUS,1,0)
    MTRAPHANDLER(t_ide,IRQ_OFFSET+IRQ_IDE,1,0)
    MTRAPHANDLER(t_error,IRQ_OFFSET+IRQ_ERROR,1,0)
    MTRAPHANDLER(t_end,-1,0,0) 
```
正常的话应该需要再trapentry.S中注册NOEC并在trap.c中extern后SETGATE.

显示为
> [00000000] new env 00001000
> I am the parent.  Forking the child...
> [00001000] new env 00001001
> Irq_Time
> Irq_Time
> I am not the child.  Spinning...
> Irq_Time
> I am the child.  Spinning...
> Irq_Time
> I am the parent.  Running the child...
> yielding
> Irq_Time
> yielding
> Irq_Time
> yielding
> Irq_Time
> yielding
> Irq_Time
> yielding
> Irq_Time
> yielding
> Irq_Time
> yielding
> Irq_Time
> yielding
> Irq_Time
> I am the parent.  Killing the child...
> [00001000] destroying 00001001
> [00001000] free env 00001001
> [00001000] exiting gracefully
> [00001000] free env 00001000
> No runnable environments in the system!
> Welcome to the JOS kernel monitor!
> Type 'help' for a list of commands.

即每次yield陷入到child中执行while;一段时间被时钟中断打断。

出现hardware interrupt即可。接下来我们处理它

#### **Exercise14**

```C++
        if(tf->tf_trapno == IRQ_TIMER + IRQ_OFFSET){
                cprintf("Irq_Time\n");
                //print_trapframe(tf);
                lapic_eoi();
                sched_yield();
               return;
        }
```
### Inter-Process communication (IPC)
#### **Exercise15**

```C++
    bool env_ipc_recving;       // Env is blocked receiving
    void *env_ipc_dstva;        // VA at which to map received page
    uint32_t env_ipc_value;     // Data value sent to us
    envid_t env_ipc_from;       // envid of the sender
    int env_ipc_perm;       // Perm of page mapping received
```

Debug出来一个lab2的bug..
是boot_alloc()中的result和nextfree是不一样的！！当时就不明白为什么要返回一样的值，这样lab3遇到的问题也解决了！result是之前的nextfree！！
```C++
    result=nextfree;
    nextfree=ROUNDUP(nextfree+n,PGSIZE);
    if ((uint32_t)nextfree-KERNBASE>(npages*PGSIZE))
    {
        panic("boot_alloc: Out of memory\n");
    }
    return result;//return nextfree?
```
最后的send和recv也按照注释写就好，就不贴了。
make grade 80/80
最后做了个Challenge,写在了Exercise7附近，利用了COW，实现对一个env创建还原点（快照）和还原。
还有要说的一点是好几个基友对于CPU是不是并行有怀疑，**CPU多核当然是并行的！同时处理以提升效率这才是目的所在**，大概是被大内核锁和同时不能多个一起yield()给迷惑了，其实是正常情况下是并行运行，只有在陷入内核时才会排队的。参考一个问题：为什么加锁了还要分堆栈？因为是并行运行在trap的时候有可能会push进不同的trapframe(lock_kernel在push了之后才加)导致混乱的。所以是**因为是并行**！

并行！

至此,lab4完成。

## 启动流程
### 启动概况
汇编建环境 → start_kernel → rest_init 拆线程 → <mark style="background: #BBFABBA6;">initcall 探驱动</mark> → mount root → exec init。

### 详细流程
#### 阶段一：汇编入口 head.S
uboot跳进内核后执行的第一段内核代码。
##### 内核真实入口primary_entry
头部_head最先调用primary_entry。
```
primary_entry
  ├─ preserve_boot_args     //先确定启动参数。比如x0=fdt物理地址（dtb地址）
  ├─ init_kernel_el          // 进入合适的 Exception Level，权限等级，内核是el1
  ├─ set_cpu_boot_mode_flag
  ├─ __create_page_tables    // 建立初始页表（identity + kernel map）
  ├─ __cpu_setup             // [proc.S] 为开 MMU 做准备
  └─ __primary_switch
       ├─ __enable_mmu        //开MMU，给cpu虚拟地址使用
       └─ __primary_switched   搭建c环境
            ├─ 设栈、VBAR、FDT 物理地址交给 C 全局变量，挂fdt
            ├─ 清 BSS
            ├─ early_fdt_map(x21)
            └─ b start_kernel     // 进入 C 
```


#### 阶段二：start_kernel 主函数
内核的第一个c函数，把裸硬件+页表变成了可调度的操作系统。
##### 执行顺序
```
asmlinkage __visible void __init start_kernel(void)
{
    local_irq_disable();
    boot_cpu_init();
    pr_notice("%s", linux_banner);     // 打印功能，串口首次进linux内核
    setup_arch(&command_line);         // 解析 FDT，memblock初始化等
    setup_command_line(command_line);  // 保存到 saved_command_line
    parse_early_param();               
    build_all_zonelists(NULL);
    page_alloc_init();
    mm_init();                        
    sched_init();
    init_IRQ();
    time_init();
    local_irq_enable();                // 首次真正打开中断
    console_init();                    // printk 控制台可用
    ... /* vfs / proc / cgroup 等大量子系统 */
    arch_call_rest_init();             // rest_init()，本函数实质结束

}
```
其中<mark style="background: #BBFABBA6;">setup_arch</mark>比较复杂，应该是解析板级信息，后续再拓展吧。主要完成以下功能：
1. 读 FDT（含 `/chosen/bootargs`）— 拿到串口、root 等命令行；
2. 认内存 — 知道 DDR 有多大、哪些能用；
3. 展开设备树 — 以后驱动才能按节点找硬件；
4. 摸清 CPU/电源 — 几个核、怎么通过 PSCI/ATF 管它们。

#### 阶段三：rest_init 与 kernel_init
##### rest_init — 启动模型的「分水岭」
```
noinline void __ref rest_init(void)
{
    pid = kernel_thread(kernel_init, NULL, CLONE_FS);  // → pid 1 候选
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    complete(&kthreadd_done);
    schedule_preempt_disabled();
    cpu_startup_entry(CPUHP_ONLINE);   // 当前线程 → idle，永不回到 start_kernel
}
```
start_kernel结束，同时创建kernel_init,kthreadd线程，start_kernel变空闲，kernel_init挂文件系统等。后续真正的继续启动再<mark style="background: #BBFABBA6;">kernel_init</mark>完成
##### kernel_init → kernel_init_freeable
```
static int __ref kernel_init(void *unused)
{
    kernel_init_freeable();   // 含 SMP、initcalls、挂 rootfs
    free_initmem();           // 释放 __init 段
    system_state = SYSTEM_RUNNING;
    /* 下面尝试 exec 用户态 init */
    ...
}
```
```
static void __init kernel_init_freeable(void)
{
    wait_for_completion(&kthreadd_done);
    smp_init();
    sched_init_smp();
    do_basic_setup();         // ★ 驱动initcall
    console_on_rootfs();
    ...
    if (init_eaccess(ramdisk_execute_command) != 0) {
        ramdisk_execute_command = NULL;
        prepare_namespace();  // ★ 挂rootfs
    }
}
```
do_basic_setup()：基础环境没问题了，开始在这一步装驱动，跑各种子系统的初始化，大致流程如下：
```
static void __init do_basic_setup(void)
{
    cpuset_init_smp();
    driver_init();        //驱动模型bus/class就绪
    init_irq_proc();
    do_ctors();
    usermodehelper_enable();
    do_initcalls();      //启用板子上的设备，emmc，显示，摄像头等
}
```
<mark style="background: #BBFABBA6;">do_initcalls</mark>按固定顺序，把内核编译时登记好的一大串「开机初始化函数」挨个执行一遍。
```
static void __init do_initcalls(void)
{
    for (level = 0; level < ...; level++)
        do_initcall_level(level, command_line);
}
static void __init do_initcall_level(int level, char *command_line)
{
    parse_args(...);   // 本级别相关的内核参数
    for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
       do_one_initcall(...);   // ★ 逐个调用 xxx_init()

}
```
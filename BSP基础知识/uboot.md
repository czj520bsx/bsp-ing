# 启动流程

## 启动概况
F 阶段 → 重定位 → R 阶段 → 主循环。
F:board_init_f(): f=flash/早期，栈在sarm，ddr这时候可能没启用，bss不可用。
R:board_init_r()R=ram，重定位后跑在ddr,此时完整 C 环境。

## 详细流程
### 阶段一：汇编入口 start.s
CPU 复位后执行的第一段 U-Boot 代码（Mask ROM / SPL 加载到这里后从此运行），这阶段CPU已经启动了。

### 阶段二：C 运行时 crtXX.S
执行"_main"，大致流程如下：
1. board_init_f_alloc_reserve：设置栈指针 sp、分配 gd；
2. <mark style="background: #BBFABBA6;"><mark style="background: #FFF3A3A6;">==board_init_f()==</mark></mark>：早期 C 初始化；
3. relocate_code()：完整 U-Boot：搬到 DDR；
4. c_runtime_cpu_setup()：重定位后刷新异常向量基址；
5. <mark style="background: #BBFABBA6;">board_init_r</mark>()：C 初始化。

### 阶段三：早期初始化 board_init_f.c
```
void board_init_f(ulong boot_flags)
{
    gd->flags = boot_flags;
    if (initcall_run_list(init_sequence_f))
        hang();
}
```
主要是init一个函数表：init_sequence_f，大致流程如下：
1. `arch_cpu_init`：架构级 CPU 初始化 ；
2. `mach_cpu_init`： SoC 级初始化（Rockchip → rk3562.c）；
3. `initf_dm`：驱动模型早期初始化 ；
4. `timer_init` ： 定时器 ；
5. `env_init`：环境变量框架；
6. <mark style="background: #BBFABBA6;">`serial_init` / `console_init_f`</mark> ：串口、早期控制台；
7. <mark style="background: #BBFABBA6;">`dram_init`</mark>： **初始化 DDR**（关键）；
8. `setup_dest_addr`：计算重定位目标地址；
9. `reserve_*` 系列 ： 为 U-Boot、malloc、FDT、栈等保留内存；
10. `setup_reloc` ：设置重定位参数 。
注：1和2是板级的钩子，U-Boot 通用框架声明了一些 `__weak` 空函数（可被平台覆盖），Rockchip 在自己的 `.c` 里再写同名函数，链接时覆盖 weak 默认实现；dram_init 成功之后，才有足够内存做代码重定位和后续工作，完整 U-Boot 里常只是 填 `gd->ram_size`；DDR 训练多在 SPL / rkbin / 厂商 loader，不是到这里才第一次“点亮 DDR”，第一次常在阶段一里面。

### 阶段四：后期初始化 board_init_r.c
```
void board_init_r(gd_t *new_gd, ulong dest_addr)
{
    if (initcall_run_list(init_sequence_r))
        hang();
    hang();   /* 正常不应到达 */
}
```
同样是init函数表：init_sequence_r，大致流程如下：
1. `initr_malloc`：堆内存；
2. `initr_of_live`：设备树；
3. `initr_dm`： 驱动模型完整初始化；
4. `board_init`：板级芯片片选等；
5. `initr_mmc` / `initr_nand`： 存储eMMC/NAND；
6. `console_init_r`：完整控制台；
7. `initr_net`：以太网（若启用）；
8. `board_late_init`：板级晚期 Rockchip logo、启动模式等。
初始化结束后进入run_main_loop:
```
static int run_main_loop(void)
{
    for (;;)
        main_loop();    /* 永不返回，除非 autoboot 失败后 retry */
    return 0;
}
```
console_init_f和console_init_r：一个flash阶段，只可以printf打印；一个ram阶段，搭完整控制台框架，可多设备、跟 env 绑定。

### 阶段五：主循环与命令行 
#### main.c
```
void main_loop(void)
{
    cli_init();
    run_preboot_environment_command();   /* 若有 preboot 环境变量 */
    s = bootdelay_process();             /* 读 bootdelay、bootcmd */
    autoboot_command(s);                 /* 自动执行 bootcmd */
    cli_loop();                          /* ★ 命令行停在这里 */
    panic("No CLI available");
}
```

#### 自动启动 autoboot.c
```
void autoboot_command(const char *s)
{
    if (stored_bootdelay != -1 && s && !abortboot(stored_bootdelay)) {
        run_command_list(s, -1, 0);      /* 执行 bootcmd */
    }
}
```
此处可以打断boot，进入命令行，打断bootcmd进入cli命令行。

#### 命令行停在哪
```
cli_loop()                    → cli.c
  └─ cli_simple_loop()        → cli_simple.c
       └─ cli_readline()       → cli_readline.c
            └─ getc()          ★ 阻塞等待串口输入
```
readline出现"=> 提示符"

### 阶段六：启动 Linux
bootcmd的个性化启动linux，根据实际需求进行修改。

## 通用总结
### 可复用的中段（各厂商大体一样）
完整 U-Boot 入口 `start.S` → C 运行时 → `board_init_f` → 重定位 → `board_init_r` → `main_loop` → `bootdelay`/`abortboot` → 执行 `bootcmd` 之前。

### 个性化两端
1. `start.S` 之前：BootROM / TPL / SPL / MiniLoader（以及 RK 的 DDR 二进制、部分平台的 ATF 等），把 U-Boot 送到可执行地址。
2. `bootcmd` 及之后：从哪取内核（分区 / 文件路径）、镜像格式（`boot.img` / `zImage`+dtb / FIT）、用哪条启动命令落到 Linux。

## RK3562-SP-EVM的个性化两端
###  通用start.s之前的前导链
 #### **Mask ROM —— 固化小引导**
	芯片内部固化的一小段程序：上电复位后 CPU 第一个跑它。
####  MiniLoaderAll.bin —— DDR 训练 + SPL 合体
	第一次启动DDR，把ddr变成主存后，将控制权限给SPL。
#### SPL-小uboot
	有自己的通用uboot 框架，主要是认出启动方式，读出FIT，解析FIT把相关解析文件放到对应地址。要求SPL要懂内存+懂FIT，体积必须小。

### 通用bootcmd后的linux内核引导
#### 启动流程
```
main_loop()
  → autoboot_command()
       → run_command_list("boot_android mmc 0; ...")
            → do_boot_android()
                 → android_bootloader_boot_flow()
                      ├─ 读 misc 分区 → 判断 normal/recovery/fastboot
                      ├─ android_image_load_by_partname("boot")
                      ├─ 组装 bootargs
                      └─ android_bootloader_boot_kernel()
                           → do_bootm_states()
                                → 跳转到 Linux entry（不再返回）
```
#### android_bootloader_boot_flow流程
```
int android_bootloader_boot_flow(struct blk_desc *dev_desc,
                                 unsigned long load_address)
{
    /* 1. 读 misc，确定启动模式 */
    mode = android_bootloader_load_and_clear_mode(...);
    /* 2. 从 boot/recovery 分区加载 boot.img */
    android_image_load_by_partname(dev_desc, boot_partname, &load_address);
    /* 3. 组装 command line，写入 bootargs 环境变量 */
    command_line = android_assemble_cmdline(...);
    env_update("bootargs", command_line);
    /* 4. 启动内核 */
    android_bootloader_boot_kernel(load_address);
}
```
#### android_bootloader_boot_kernel流程
调用do_bootm_states，完成如下内容：
1. 解析 Android boot.img 或 FIT；
```
    if (!ret && (states & BOOTM_STATE_FINDOS)) 
        ret = bootm_find_os(cmdtp, flag, argc, argv);//解析boot.img
    if (!ret && (states & BOOTM_STATE_FINDOTHER)) 
        ret = bootm_find_other(cmdtp, flag, argc, argv);//找dtb等
```
2. 加载 kernel、dtb、ramdisk 到指定内存；
```
    if (!ret && (states & BOOTM_STATE_LOADOS)) {
        ulong load_end;
        iflag = bootm_disable_interrupts();
        ret = bootm_load_os(images, &load_end, 0); //加载kernel
        if (ret == 0)
            lmb_reserve(&images->lmb, images->os.load,
                    (load_end - images->os.load));
        else if (ret && ret != BOOTM_ERR_OVERLAP)
            goto err;
        else if (ret == BOOTM_ERR_OVERLAP)
            ret = 0;
    }
```
3. 关闭 U-Boot 设备，跳转到内核入口。

```
if (!ret && (states & BOOTM_STATE_OS_GO))
        ret = boot_selected_os(argc, argv, BOOTM_STATE_OS_GO,
                images, boot_fn);
后续调用链：
BOOTM_STATE_OS_GO:
boot_selected_os(..., OS_GO, boot_fn) // bootm_os.c
└─ boot_fn(OS_GO, ...) → do_bootm_linux()
└─ boot_jump_linux() //跳转内核
```

## 正点原子IMX6ULL
 ### **通用start.s之前的前导链**
BootROM芯片厂商出厂的固化程序，按DSD表去写寄存器，配置时钟，DDR等，类比rk3562-evm的MaskROM的DDR训练。
### 通用bootcmd后的linux内核引导
```
board_init_r
  → run_main_loop
      → main_loop()                          【common/main.c】
          → bootdelay_process()              【common/autoboot.c】取出 bootcmd
          → autoboot_command(bootcmd)        【common/autoboot.c】
              → run_command_list(bootcmd)    【common/cli.c】
                  │  bootcmd = CONFIG_BOOTCOMMAND【include/configs/mx6ullevk.h】
                  ├─ run findfdt             定 fdt_file 名字
                  ├─ mmc dev / mmc rescan    【cmd/mmc.c】
                  ├─ run loadimage           【mx6ullevk.h】
                  │     └─ fatload mmc ... zImage
                  │           → zImage 进内存 loadaddr
                  │           【cmd/fat.c → fs/fs.c】
                  │
                  └─ run mmcboot             【mx6ullevk.h】
                        ├─ run mmcargs
                        │     └─ setenv bootargs（串口 / root=...）
                        ├─ run loadfdt
                        │     └─ fatload mmc ... dtb
                        │           → dtb 进内存 fdt_addr
                        │           【cmd/fat.c → fs/fs.c】
                        └─ bootz ${loadaddr} - ${fdt_addr}
                              → do_bootz()       【cmd/bootm.c】，类似rk3562的挂内核
                                    ├─ bootz_start()
                                    │     └─ 认 zImage，记下内核 / dtb 地址
                                    └─ do_bootm_states(OS_PREP | OS_GO)
                                          → do_bootm_linux()   【bootm.c】
                                                ├─ boot_prep_linux()
                                                │     └─ 准备 bootargs / dtb
                                                └─ boot_jump_linux()
                                                      └─ kernel_entry()
```
bootcmd的执行表`mx6ullevk.h`
```
#define CONFIG_BOOTCOMMAND \
       "run findfdt;" \
       "mmc dev ${mmcdev};" \
       "mmc dev ${mmcdev}; if mmc rescan; then " \
           "if run loadbootscript; then " \
               "run bootscript; " \
           "else " \
               "if run loadimage; then " \ /*加载内核zlmage*/
                   "run mmcboot; " \   /*加载设备树*/
               "else run netboot; " \
               "fi; " \
           "fi; " \
       "else run netboot; fi"
#endif
```
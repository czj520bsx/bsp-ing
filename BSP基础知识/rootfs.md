注：本篇笔记只是在BusyBox + Buildroot基础上进行讲解
## BusyBox，Buildroot和rootfs的关系
Buildroot 造出 rootfs；BusyBox 住在 rootfs 里，负责开机后当 pid 1。
```
Buildroot（PC 上构建）
    │  编译 BusyBox、拷贝配置/脚本、打包
    ▼
rootfs 镜像（ext4 / squashfs / tar…）
    │  烧录 / 挂载
    ▼
板子上的 /
    ├── /sbin/init   ← BusyBox 开机后的第一个程序
    ├── /etc/inittab
    ├── /etc/init.d/rcS
    └── …
```

## 阶段一： 开机用户态的起点：/sbin/init
```
kernel_execve("/sbin/init")
  → busybox 启动
  → 识别 applet 名 "init"
  → init_main(argc, argv)     // 用户态真正入口，init.c里面
```

## 阶段二：init_main 主流程
```
int init_main(int argc, char **argv)
{
    /* 屏蔽一批信号，延后处理（CAD / halt / reboot / CHLD…） */
    sigprocmask(SIG_BLOCK, &G.delayed_sigset, NULL)
    if (getpid() != 1)
        die("must be run as PID 1");   // 内核约定：init 必须是 1 号进程
    console_init();
    set_sane_term();
    xchdir("/");
    setsid();
    /* PATH / SHELL / USER 等环境 */
    message(..., "init started: %s", bb_banner)
    if (单用户模式参数)
        new_init_action(RESPAWN, shell, ...);
    else
       parse_inittab();                // ★ 读 /etc/inittab，init 的任务清单
    /* 一次性阶段 */
    run_actions(SYSINIT);              // 含 rcS ，开机服务总开关脚本
    run_actions(WAIT)
    run_actions(ONCE);
    while (1) {
        run_actions(RESPAWN | ASKFIRST);
        /* 等信号、收尸、必要时重启 respawn 进程 */
        check_delayed_sigs(...);
        waitpid(...);
        sleep1();
    }
```
## 阶段三：解析 /etc/inittab
```
::sysinit:/bin/mount -t proc proc /proc
::sysinit:/bin/mount -o remount,rw /
::sysinit:/bin/mkdir -p /dev/pts /dev/shm
::sysinit:/bin/mount -a
::sysinit:/bin/mkdir -p /run/lock/subsys
::sysinit:/sbin/swapon -a
null::sysinit:/bin/ln -sf /proc/self/fd /dev/fd
null::sysinit:/bin/ln -sf /proc/self/fd/0 /dev/stdin
null::sysinit:/bin/ln -sf /proc/self/fd/1 /dev/stdout
null::sysinit:/bin/ln -sf /proc/self/fd/2 /dev/stderr
::sysinit:/bin/hostname -F /etc/hostname
# now run any rc scripts
::sysinit:/etc/init.d/rcS
......
```
行格式：[id]:[runlevels]:[action]:[process]
| 字段 | BusyBox 含义 |
|------|----------------|
| `id` | 终端节点（空 = 默认 console；`ttyS0` → `/dev/ttyS0`） |
| `runlevels` | **忽略**（BusyBox init **无 SysV runlevel**） |
| `action` | `sysinit` / `wait` / `once` / `respawn` / `askfirst` / `shutdown` /| 开机阶段的先后顺序
| `process` | 要执行的命令行 |
注：如果没有inittab的话，会有一套默认动作执行
## 阶段四：run_actions(SYSINIT) 跑通 sysinit
```
static void run_actions(int action_type)
{
    for (a = G.init_action_list; a; a = a->next) {
        if (!(a->action_type & action_type))
            continue;
        if (a->action_type & (SYSINIT | WAIT | ONCE | ...)) {
            pid_t pid = run(a);           /* fork/vfork + 子进程 exec */
            if (a->action_type & (SYSINIT | WAIT | ...))
                waitfor(pid);             /* SYSINIT：等子进程结束 */
        }
        /* RESPAWN/ASKFIRST：只在 pid==0 时启动，不等待 */
    }
}
```
## 阶段五：rcS 按序号执行 init.d
只匹配Sxx（两位序号+名字），启动顺序根据字典序来排：`S00` `S03` `S49` `S99`
在本项目中，建立分区文件的脚本：buildroot/board/rockchip/rk3562/ctu-overlay/etc/init.d/S03ctu-mount等。UI合成器的有buildroot/package/weston/S49weston。都是两位序号+名字。也可以个性化编写脚本，跑系统喂狗或者开机软件等

## 阶段六：主循环 respawn / 关机 shutdown
### 主循环
```
while (1) {
    run_actions(RESPAWN | ASKFIRST);
    check_delayed_sigs(NULL);
    while ((wpid = waitpid(-1, NULL, WNOHANG)) > 0) {
        a = mark_terminated(wpid);  /* RESPAWN 项 pid 置 0 → 下次再 run */
    }
    sleep1();
}
```
### 关机
```
::shutdown:/etc/init.d/rcK
::shutdown:/sbin/swapoff -a
::shutdown:/bin/umount -a -r
```
rcK 反向对 S?? 调 `stop`

## Rootfs 的制作

注：制作在 PC 上完成；启动是板子每次上电跑已经做好的文件。二者别混。

### 制作视角：三者关系
Buildroot 造出 rootfs；BusyBox 被编进 rootfs，板上当 `/sbin/init`（pid 1）。
```
BusyBox 源码 ──make──► busybox ELF + /bin/ls、/sbin/init … 链接
                              │
                              ▼  装进某个目录（CONFIG_PREFIX / TARGET_DIR）
其余材料（skeleton、inittab、libc、overlay…）──► 完整 rootfs 树
                              │
                              ▼  mkfs / Buildroot 打镜像
                         rootfs.img（可烧录）
```

### 通用：只用 BusyBox 源码怎么编
```
cd busybox-x.y.z
make menuconfig          # 打开 Init→init；交叉编译设 Cross Compiler prefix
make -j$(nproc)          # 编出一个多呼 ELF
mkdir -p /tmp/myroot
make CONFIG_PREFIX=/tmp/myroot install
```
`make install` 会：
1. 把 `busybox` 拷进 CONFIG_PREFIX（常见 `/bin/busybox`）
2. 按 .config 建 `/sbin/init`、`/bin/sh`、`/bin/ls`… 链接指向它

交叉编译注意：
- 必须用板子同架构工具链，别用宿主机 gcc 直接编目标程序
- 可选 `CONFIG_STATIC=y` 静态链接（可不依赖目标机 libc，体积更大）
- 没开 `CONFIG_INIT` → 没有 `/sbin/init`，内核 exec 后起不来

### 凑齐可启动 rootfs（BusyBox alone 不够）
此时 `/tmp/myroot` 只有 BusyBox 那一层。还要：
| 需要 | 作用 |
|------|------|
| `/etc` `/proc` `/sys` `/dev` `/tmp`… | 挂载点与配置落点 |
| `/etc/inittab` | init 任务清单 |
| `/etc/init.d/rcS` + `S??*` | 开机脚本 |
| passwd / hostname | 登录、主机名 |
| 动态库（非静态时） | 否则一 exec 就缺 .so |
| 设备节点或 mdev/udev | console / 串口 |

手搓示意：
```
ROOT=/tmp/myroot
# 已 make CONFIG_PREFIX=$ROOT install
mkdir -p $ROOT/{etc/init.d,proc,sys,dev,tmp,root}
# 放入 inittab（含 sysinit + getty respawn）、简易 rcS
# 动态链接则拷 .so 到 $ROOT/lib
# 再 mkfs.ext4 / 打包成镜像烧进去
```

### Buildroot 怎么自动做上面这些
产品不天天手敲 CONFIG_PREFIX，Buildroot 内部等价：
```
Buildroot make
  ├─ 准备 TARGET_DIR（= 即将成为 rootfs 的目录树）
  ├─ 解压/配置 BusyBox
  ├─ CROSS_COMPILE=… make
  ├─ make CONFIG_PREFIX=$(TARGET_DIR) install
  ├─ 再装 inittab、initscripts、其它包、overlay
  └─ 按配置打成 ext4 / squashfs …
```
只重编 BusyBox：
```
cd buildroot
make busybox-menuconfig
make busybox-rebuild
```

### 本 SDK
```
./build.sh rootfs
  → mk-rootfs.sh
  → mk-buildroot.sh rockchip_kx_rk3562_sp
  → output/.../target/（TARGET_DIR）→ rootfs.img
```
Overlay（后写覆盖先写）：
```
BR2_ROOTFS_OVERLAY="board/rockchip/common/base
                     board/rockchip/rk3562/sp-overlay
                     board/rockchip/rk3562/ctu-overlay"
```

| 想改 | 改哪里 |
|------|--------|
| init / applet | busybox menuconfig / busybox 配置 |
| 早期启动动作 | package/busybox/inittab |
| 开机服务 | overlay 或 package 的 `S??*`（如 S03ctu-mount、S49weston） |
| 整包选型 | rockchip_kx_rk3562_sp_defconfig |

### 做完自检
```
ls -l …/target/sbin/init     # 期望指向 busybox
cat  …/target/etc/inittab
ls   …/target/etc/init.d/
```

### 做出来的文件 ↔ 启动阶段
| 镜像里有                       | 启动时                           |
| -------------------------- | ----------------------------- |
| `/sbin/init`（BusyBox）      | 阶段一 init_main                 |
| `/etc/inittab`             | 阶段二/三 parse_inittab → SYSINIT |
| `/etc/init.d/rcS` + `S??*` | 阶段四/五 按序号起服务                  |
| getty respawn              | 阶段六 主循环保活登录                   |


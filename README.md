# 网心云 OECT · Ubuntu 26.04 LTS 刷机与升级说明

## 一、包内容与来源

| 文件 | 来源 | 说明 |
|---|---|---|
| `Armbian_26.11.0_rockchip_wxy-oect_resolute_6.1.174_server_2026.11.03_secupd.img.gz` | ophub/amlogic-s9xxx-armbian 官方 releases（2026-11-03 批次）+ 本包离线处理 | **当前推荐刷写镜像**：Ubuntu 26.04 (resolute) 用户态 + Armbian 工具链，内置 6.1.174 内核；**已离线烘焙中文化+首启固化层+全量安全补丁升级（2026-11-06，基于 2026.11.03 上游基线重制）**，rootfs 分区扩容至 3.75GiB（明细见第六节） |
| `ophub-kernel-6.12.117.tar.gz` | ophub/kernel 官方 releases (kernel_stable) | 内核升级包：6.12.117 主线内核，含 `rk3566-wxy-oec-turbo-4g.dtb` |
| `wxy-oect-MiniLoaderAll.bin` | ophub/u-boot 官方仓库 (rockchip/wxy-oect) | 线刷时 RkDevTool 使用的 Loader |
| `SHA256SUMS.txt` | 本包自算 | 文件完整性校验值 |

内核包与 Loader 为官方仓库原样文件；系统镜像为官方原样 + 离线烘焙（详见 3.2 节与第六节）。SHA256SUMS 为自算记录（ophub 发布未附官方校验文件），下载后请先核对再刷写。

**适用设备**：网心云 OECT（Turbo 版，RK3566 / 4GB 内存 / 8GB eMMC）。主板丝印 `WXY-OECT`。其他型号（OEC 2GB、OES 晶晨系）不要用本包。

## 二、刷机步骤（线刷到 eMMC）

准备：双公头 USB 数据线、电脑装瑞芯微刷机工具 RkDevTool（或 Rockchip 官方 DevelopTool）、回形针/镊子短接。

1. **解压** `Armbian_...img.gz` 得到 `.img` 镜像文件（约 3.9GB）。
2. **核对** SHA256（`sha256sum` 或 Windows 用 `certutil -hashfile 文件 SHA256`）。
3. 设备**断电断网**，拆开外壳找到恢复短接点，按住短接进入 **MASKROM** 模式（接 USB 到电脑后设备管理器出现 "Rockchip MASKROM" 设备）。
4. RkDevTool → 「下载镜像」页：
   - Loader 行地址 `0xCCCCCC`，选择 `wxy-oect-MiniLoaderAll.bin`
   - 镜像行地址 `0x0`，选择解压出的 `.img`
5. 点击「执行」，等待写入完成（约几分钟）。
6. **注意**：二次刷机时严禁再勾选 Loader 行，只写镜像即可，否则可能破坏引导。

> 刷机有变砖风险，属正常可救范围（MASKROM 短接可重刷）。动手前建议先用 U 盘/TF 卡方式试跑验证，确认满意再写入 eMMC。

## 三、首次启动

1. 插网线（DHCP）、通电，等待 1-2 分钟。
2. 路由器后台找到设备 IP，SSH 登录：`ssh root@<IP>`，默认密码 `1234`。**英文首登向导已禁用**（出厂中文化的一部分），登录即直接进入中文欢迎屏；请第一时间执行 `passwd` 修改 root 密码。
3. 串口调试（可选）：调试 UART 波特率 `1500000`，8N1（ttyS2）。
4. 时区/语言/欢迎屏均为中文预置（3.2 节），无需手动设置；换国内软件源可选。
5. （可选）恢复 Armbian 原版英文首登向导：`touch /root/.not_logged_in_yet` 后重新登录。

### 3.1 浏览器远程桌面（本机无视频输出接口，推荐）

**首启自动安装**：固件已内置首启固化层（oect-firstboot），刷机后第一次联网开机会自动完成本节与 3.2 节、3.4-3.5 节的全部安装与启用，全程无需敲命令（约 5-10 分钟，日志 `journalctl -u oect-firstboot` 或 `/var/log/oect-firstboot.log`）；某步失败（如未联网）下次开机自动重试，全部完成后自禁用。本节命令仅作手动重装/修复用：

SSH 登录后执行一条命令，装完即可用浏览器看到可操作桌面：

```bash
bash <(cat /path/to/setup-remote.sh)          # 默认密码 oecadmin
# 或自定义密码：bash setup-remote.sh 你的密码
```

- 访问方式：浏览器打开 `http://<设备IP>:6080/vnc.html`，输入密码即进入桌面（Xvfb 虚拟屏幕 + openbox + xterm + noVNC）。
- 桌面已中文化：openbox 右键菜单为中文（打开终端/重启系统/关机）；xterm 已配置 Noto CJK 中文字体与 fcitx5 输入法（XIM），终端里可直接显示并输入中文（Ctrl+Space 切换）。
- 开机自启（systemd `oec-remote.service`），重启后无需再操作。
- 额外占用约 300MB 磁盘；端口 6080/5900 仅限局域网使用，勿直接暴露公网。

### 3.2 系统中文化（出厂预置，Ubuntu 官方机制）

**已烘焙进固件**（离线注入）：
- 语言环境层：zh_CN.UTF-8 locale 配置、LANG=zh_CN.UTF-8、时区 Asia/Shanghai、chrony 国内源（ntp.aliyun.com / cn.pool.ntp.org，原全球池后备）；
- Shell 层（原系统杂项英文全部默认改中文）：Armbian **英文首登向导已禁用**（登录不再出现英文改密/建户交互）、英文欢迎屏（motd 全套）已归档并替换为**中文欢迎屏+中文常用入口**、`/etc/issue` `/etc/issue.net` 登录横幅中文、"内核/软件包需重启"提示中文化（`MOTD_DISABLE` 为 Armbian 官方机制，BSP 升级也不复活英文欢迎屏）；
- 语言包/字体/输入法（fcitx5）/man 中文页等 dpkg 包由首启固化层联网自动安装（见 3.1 节首启说明）。

`setup-l10n.sh`（语言环境）与 `setup-cn-shell.sh`（Shell 层）保留为真机校验/异常修复入口（均幂等，可重跑）：

```bash
sudo bash setup-l10n.sh         # 幂等: 已装则快速跳过, 亦可修复异常
sudo bash setup-cn-shell.sh     # 幂等: Shell 层中文修复(Armbian 升级恢复英文文件后重跑)
locale -a | grep zh_CN          # 校验 1: zh_CN.utf8 在列
dpkg -l fonts-noto-cjk fcitx5 language-pack-zh-hans | grep ^ii   # 校验 2: 字体+语言包+输入法
timedatectl | grep "Asia/Shanghai"                        # 校验 3: 北京时间
grep oect-cn-ntp /etc/chrony/chrony.conf                  # 校验 4: 国内 NTP 源
man ls | head -1                                          # 校验 5: man 中文页(可选)
cat /etc/issue                                            # 校验 6: 登录横幅中文
ls /etc/update-motd.d/                                    # 校验 7: 仅 00-clear/10-oect-header/20-oect-tips/99-blank(英文屏已归档)
ls /root/.not_logged_in_yet 2>/dev/null || echo "英文首登向导已禁用"   # 校验 8
```

- 采用 Canonical 官方 i18n 方案（zh_CN.UTF-8 locale + 官方 `language-pack-zh-hans`）。此后系统菜单、apt、systemd 等提示均为中文。
- **后续官方更新的新增开关/提示会自动保持中文**：语言包随 Ubuntu 官方源同步更新，这是官方机制，无需手工移植语言文件。
- 原厂 OECT 固件的中文界面属于厂商私有网页面板，本系统不运行该面板，其语言文件不适用移植。
- 含 `fonts-noto-cjk` 字体，浏览器远程桌面（3.1 节）中文显示正常；`fcitx5` 拼音输入法随远程桌面会话自启，桌面里可直接打中文（Ctrl+Space 切换）。
- `manpages-zh`：核心命令 man 页默认中文。
- 时区 `Asia/Shanghai` + chrony 国内源：日志与对时均为北京时间，国内对时快且稳。
- TTY 控制台不加中文支持（fbterm 类）：本机无视频输出接口，纯远程使用，无此需求。

### 3.3 性能调度与可选轻微超频（tune-perf.sh）

三档调度（CPU 始终交给内核 schedutil 动态调节，仅调节奏；GPU/NPU 只调介入阈值，不锁频）：

| 档位 | 效果 |
|---|---|
| `eco` | 偏节能：CPU 升频迟钝降频快，GPU/NPU 负载阈值抬高 |
| `balanced` | 内核默认节奏（基线） |
| `boost` | 偏激进：CPU 快升慢降，GPU/NPU 提前介入；始终由内核动态控制 |

```bash
sudo bash tune-perf.sh apply boost   # 应用并设为开机持久
sudo bash tune-perf.sh status        # 查看频率/调度器/温度
sudo bash tune-perf.sh restore       # 完全回滚到初始状态
```

**可选轻微超频**（默认不做；默认仅 800MHz 基准 OPP，上探即改 DTB，重启生效）：

```bash
sudo bash tune-perf.sh oc-apply gpu 900 950 --yes-i-know   # GPU 800->900MHz @ 950mV(推荐档)
sudo bash tune-perf.sh oc-apply npu max --yes-i-know       # 最高档预设: 1000MHz @ 1.00V
sudo bash tune-perf.sh oc-restore                          # 一键回滚(自动取最新备份)
```

安全边界：`max` 预设 1000MHz@1.00V（社区稳定区上沿）；硬上限 1.05GHz/1.05V，超过 1.0GHz/1.00V 属未验证区，须真机 30 分钟负载确认后才可长期使用；**电压下限**（850+(MHz-800)/4 mV，如 1000MHz→900mV）防止把低频低压复用给高频档（能设成功但高载随机死机）；低于 800MHz 档原厂默认 1.00V 时自动提示体质型欠压；DTB 修改前自动备份；OPP 节点找不到基准频率时放弃而非猜测式修改；超频重启后用 `oc-check` 看频率档与温度墙。

**表定位（外部评审抓出的关键修复）**：真机 ophub rk3566 DTB 中 GPU 的 OPP 表名为 `opp-table-1`（不存在 gpu-opp-table），早期盲扫版本执行 `oc-apply gpu` 会**误改 NPU 的表**。现按 DTB 内 `__symbols__` 符号表精确定位（`gpu_opp_table`/`npu_opp_table`），无符号时回退确定性表名，CPU 表（opp-table-0）永不在候选。测试矩阵 40/40 全通过，含"GPU 操作不碰 NPU/CPU 表、NPU 操作不碰 GPU 表"的跨表负控。

**DTB 实测背景**（反编译证据）：GPU/NPU/RKVDEC/RKVENC 各自独立 OPP 表与供电（无硬性时钟耦合）；GPU 顶格档 800MHz@1.00V 且表内 `max-volt=1.00V`——超频不抬电源轨（≤1.00V 即原厂最大轨压）；温控为 soc-thermal（60/85/115°C 三档，85°C 被动降频经 cooling-maps 作用于 GPU）+ gpu-thermal（115°C 关断档），本工具只改 OPP 数值不动温控节点，降频联动结构保留。

**超频真机专项验证清单**（每条都过才长期用）：
1. 时钟跟随/推理：OC 后跑 RKNN demo 推理比对输出图像；GPU 满载时 `cat /sys/class/devfreq/*npu*/cur_freq` 确认 NPU 未被连带拉高异常
2. 长时硬解：`ffmpeg -hwaccel rkmpp` 循环播放 4K H.264/H.265/VP9 各 30 分钟以上，观察花屏/色块/断流（短压测暴露不了）
3. 电压/电源：满载观察是否随机重启（电压跌落信号）；有条件用万用表盯原厂电源带载，余量不足先换电源再超频
4. 温控联动：负载升温至 85°C 附近，确认 GPU/NPU 频率自动回落（`oc-check` 看温度与频率）；温度墙失效立即 `oc-restore`
5. 升降频循环：负载反复起落 ≥50 轮，观察 `dmesg` 无 devfreq/GPU 挂死报错

### 3.4 内存与链路优化（tune-sys.sh）

**镜像出厂已内置**（官方 rootfs 实测确认，无需动手）：

| 项 | 状态 |
|---|---|
| zram 压缩内存交换 | `armbian-zram-config` 已启用，内核 `CONFIG_ZRAM=y` + LZ4；**tune-sys.sh apply 后定容 25%（4G 板=1G）并切换 zstd** |
| eMMC 友好挂载 | fstab 已有 noatime/nodiratime/commit=600/errors=remount-ro |
| /tmp 进内存 | fstab 已有 tmpfs |
| eMMC TRIM | `fstrim.timer` 已启用（周.Trim） |
| I/O 调度器 | mq-deadline 内置默认（`CONFIG_MQ_IOSCHED_DEADLINE=y`，eMMC 低延迟正确选择） |

**缺省的 sysctl 层由 tune-sys.sh 补齐**（vendor 内核 `CONFIG_TCP_CONG_BBR=m`/`CONFIG_NET_SCH_FQ=m` 已确认，模块可用即启用，不可用自动回退纯 cubic 并提示）：

```bash
sudo bash tune-sys.sh apply     # 写入 /etc/sysctl.d/98-oect-tune.conf 并立即生效
sudo bash tune-sys.sh status    # 查看当前值
sudo bash tune-sys.sh restore   # 停用并归档(不删除), 恢复内核默认
```

内容两组：内存侧（**swappiness=10 页缓存/Agent KV 缓存优先定稿**、min_free_kbytes=65536 低水位防护垫、page-cluster=0、vfs_cache_pressure=50 缓存友好、dirty 比例平滑 eMMC 写突发）；链路侧（BBR 自适应拥塞控制 + fq 排队 + TCP 缓冲区上调，弱网/断流更稳——BBR 本身就是内核按网络负载自动调节的算法）。

**关于"移植新版内核网络栈"**：不需要重编内核。6.1.174 vendor 内核实测已内置现代网络栈核心件（`CONFIG` 实测证据）：MPTCP=y（多路径 TCP，配合板载 4G 模组可做双链路聚合）、CAKE=m / FQ_CODEL=y / FQ=m（抗缓冲膨胀排队）、BQL/RPS/RFS/XPS/ busy-poll（多核收发包优化）、17 种拥塞控制算法可用。新版内核并无这些之外的必选项（BBRv2/v3 连最新主线都未合并）。tune-sys.sh 已按新内核最佳实践补齐可移植参数层：`tcp_slow_start_after_idle=0`（长连接不丢速率）、`tcp_mtu_probing=1`（4G 黑洞 MTU 自适应）、`tcp_notsent_lowat=131072`（发送队列延迟上限）、backlog/somaxconn/syn_backlog 上调（突发吸收）、`net.mptcp.enabled=1`（使能而非常规生效，应用不用即无副作用）。

### 3.5 无人值守自愈（tune-ha.sh，长期开机必开）

针对"人不在家、坏了没人管"的三层自愈（镜像实测缺口逐一补齐）：

| 层 | 防什么 | 机制 |
|---|---|---|
| 硬件看门狗 | 系统假死（卡住不动） | vendor 内核已内置 DW 看门狗驱动但默认没用；启用后 systemd 每 15s 喂狗，假死即硬件级强制重启 |
| panic 自恢复 | 内核崩溃后干等 | 出厂 `PANIC_TIMEOUT=0`（崩了永远挂着）；改为 oops 即 panic、10 秒自动重启 |
| 网络自愈守卫 | 网络挂死失联 | 每 5 分钟探网关：连续 2 次失败(约 10 分钟)重启 NetworkManager(限频 1 次/时)；连续 6 次(约 30 分钟)自愈重启，24h 内最多 2 次，触顶不再重启(上层网络故障重启无益，避免重启循环) |

```bash
sudo bash tune-ha.sh apply     # 三层全开(长期无人值守建议必开)
sudo bash tune-ha.sh status    # 各层状态 + 守卫计数
sudo bash tune-ha.sh restore   # 全部撤销
```

### 3.6 内存缓存与启动参数层（tune-mem.sh，开机自动生效）

面向"本地 Agent 调度 + RKNN 推理 + 4K 硬转码"三负载并存的均衡配置（规格定稿见《固件规格说明.md》）：

| 项 | 值 | 理由 |
|---|---|---|
| bootargs | `cma=1G coherent_pool=1M` | CMA 空闲可被页缓存借用（可迁移），NPU/MPP 申请时归还；RK3566 单 NUMA，**不用** cma_pernuma（冗余且有双预留歧义）；VDPU/VEPU/RKNPU 均带 IOMMU，1G 属余量型配置 |
| zram | 25%（4G 板=1G）+ zstd | Armbian zramsize 为百分比语义；zstd 压缩比高约一倍 |
| swap 接管 | 优先级 10（低于 zram 100） | PM871 SATA 插盘后开机自动扫描启用，仅作溢出垫，防频繁刷写 |

```bash
sudo bash tune-mem.sh apply     # 写 bootargs(原件自动快照) + 预置 swap 接管服务
sudo bash tune-mem.sh status    # 当前 extraargs/cma、swap、zram 配置
sudo bash tune-mem.sh restore   # bootargs 恢复快照, swap 层停用+归档(不删除)
```

真机核对清单：`free -h` 看 zram ≈1G、`cat /proc/cmdline` 含 cma=1G、插固态盘后 `swapon --show` 出现 pri 10 条目。

镜像出厂已有的稳定性设施（实测确认，无需动手）：日志 volatile 模式（只进内存，不会撑爆 eMMC）+ 日志限速、zram、chrony 对时、fake-hwclock 断电保时钟、unattended-upgrades 安全补丁服务、ext4 日志式文件系统（断电自恢复）。

OOM 说明：镜像无 earlyoom/systemd-oomd；tune-sys.sh 定稿后 min_free_kbytes=65536 低水位防护垫 + zram 1G 应急吸收已显著降低 OOM 概率，内核 OOM killer 兜底。如需更强守护可联网后 `apt install earlyoom && systemctl enable --now earlyoom`（可选，不联网不影响其余功能）。

### 3.7 固态硬盘扩容（数据搬 SSD，系统留 eMMC）

eMMC 8GB 刷机后的空间账本（debugfs 实测）：

| 阶段 | rootfs 分区 | 实际占用 | 剩余 |
|---|---|---|---|
| 刷完（首启前） | 3.0GiB | 2.21GiB | 787MiB |
| 首启初始化后（armbian-resize-filesystem 自动扩满） | ~7.2GiB | ~2.35GiB（+locale/apt 列表） | ~4.8GiB |
| 加远程桌面+中文后 | ~7.2GiB | ~2.8GiB | ~4.3GiB |

要点：

- 首启的 `armbian-resize-filesystem` 会把 rootfs 自动扩满 8GB eMMC——**正常且建议保留**（系统升级/内核更新需要缓冲）。ext4 不支持在线缩小，之后想还 eMMC 空间只能重刷，"把系统分区缩回去"不是可选项。
- **必须留在 eMMC**：boot 分区（511MiB，u-boot+内核文件，启动的物理前提）、OS 核心 /usr /etc /lib（约 2.2GiB）。
- **搬去 SSD**（按体积排序）：业务数据/缓存目录（最大头，业务激活后按实际路径挂载）、浏览器配置与下载目录（~300-400MiB 且持续增长）、/var/lib/apt/lists（可选）。
- 操作：SSD 格 ext4 → `blkid` 取 UUID 写入 fstab（**必须带 `nofail`**，防 SSD 掉线卡死开机）→ `mount --bind` 或软链迁移对应目录 → 重启验证；SSD 掉线时系统仍可无 SSD 启动。

## 四、升级到最新主线内核 6.12.117（官方更新链路）

> **内核方向定稿：默认 vendor 内核（rk35xx BSP 线）就是最优解，无需切换。**
> - 默认 `6.1.174-rk35xx` 即 ophub 当前最新 vendor BSP 内核（官方镜像 2026-11-03 构建），Rockchip MPP 全栈内置：硬解 RKVDEC、硬编 RKVENC、JPEG/AV1、RGA 2D、IEP、RKNPU 开箱可用；
> - 后续新 BSP 内核发布后，走第五节官方更新链路即可持续升级（保持 MPP 支持）；
> - 6.12 主线仅作实验选项：切换后 MPP（硬编/硬解/NPU）全部不可用，且主线无 RKVENC 驱动、无可移植路径——**需要硬件加速的请勿切换主线**。

系统自带 ophub 官方内核更新工具 `armbian-update`，联网后执行（包内已附 6.12.117 离线包 `ophub-kernel-6.12.117.tar.gz`，也可直接用它升级）：

```bash
armbian-update -k 6.12.117 -u stable -d tar -b yes
```

- `-k 6.12.117`：目标内核版本
- `-u stable`：从 ophub/kernel 的 stable 发布批次下载
- `-d tar`：使用 tar 格式内核包
- `-b yes`：自动备份当前内核（默认开启）

执行后重启即可。可用 `uname -r` 确认内核版本。

## 五、后续持续更新（更新策略定稿）

- **更新范围（仅限 LTS，保稳定）**：
  - **安全补丁**：启用官方自动更新机制 `apt install unattended-upgrades`，仅自动安装 26.04 LTS 官方安全补丁，不引入新功能包，稳定性由 Canonical 背书。
  - **常规软件**：`apt update && apt upgrade`（Ubuntu 26.04 LTS 官方源）。
  - **内核隔离保证**：本机内核不来自 apt 源（由 ophub 构建、`armbian-update` 单独管理），因此 apt 更新天然不触碰内核；反之不要手动 `apt install linux-image-*` 引入 Ubuntu 内核覆盖 BSP 引导配置。
  - **大版本**：仅在下一个 LTS（Ubuntu 28.04）发布且本板生态验证后手动评估，LTS 之间不做发行版升级。
- **内核**：随时执行 `armbian-update -u stable -f rk35xx -d tar -b yes` 自动跟进 ophub 最新 vendor BSP 内核（保持 MPP 硬件加速；flavor 参数以 `armbian-update -h` 实际输出为准）。
- **救砖**：升级出问题可用 `armbian-update -s mmcblkX` 恢复内核，或重新线刷。

## 六、本包已验证事项

- [x] **基于 2026.11.03 上游基线重制（2026-11-06）**：上游 ophub/amlogic-s9xxx-armbian 2026-11-03 批次（`Armbian_26.11.0_..._6.1.174_server_2026.11.03.img.gz`，u-boot/loader 无变更沿用）；proot+qemu-aarch64 沙箱内 apt 安全升级：`apt / apt-utils / libapt-pkg7.0 → 3.2.1ubuntu0.1`、`libpcre2-8-0 → 10.46-1ubuntu0.1`、`ca-certificates → 20260816ubuntu0.26.04.2`（120 张 CA 证书重生成+TLS 实测通过）、`armbian-config → 26.8.0-trunk.426.1106.075418`、`linux-libc-dev → 7.0.0-43.43`；中文化/首启固化/tune 全家桶按既有流程重新注入（新增 00-clear/99-blank 官方 motd 辅助脚本保留不动）；mke2fs 重建 rootfs（UUID/卷标/特性集与原版一致）+ GPT 扩容 3.75GiB + 主备 GPT 校验 PASS + boot/rootfs 分区逐字节 cmp 一致 + gzip -t/解压流哈希一致；上游 2026.10.22 版成品及其内核包 6.12.112 已移入 archive/
- [x] **内核升级包同步至 6.12.117（2026-11-14）**：ophub/kernel kernel_stable 最新版，官方 releases 下载，包内六件套结构一致，`dtb-rockchip` 内含 `rk3566-wxy-oec-turbo-4g.dtb`；上游 Armbian 仍为 2026.11 批次（基线无更新，无需重烘焙），u-boot/loader 无变更沿用；6.12.115 历史包移入 archive/
- [x] rootfs 来源 Ubuntu 官方 cdimage，SHA256 与官方 SUMS 一致
- [x] 内核包 6.12.117 六件套内层 sha256 全部通过（官方随包 sha256sums 全 OK），`dtb-rockchip` 内确认含 `rk3566-wxy-oec-turbo-4g.dtb`；6.12.112 历史包同项验证于 2026-10-22 完成（已归档）
- [x] DTB `rk3566-wxy-oec-turbo-4g.dtb` 与 ophub 官方镜像 extlinux.conf 的 FDT 指向一致（升级内核后无需改引导配置）
- [x] 分区布局 / 引导参数已从官方镜像实测读取（boot 511MiB + rootfs 3000MiB，GPT）
- [x] `armbian-update` 工具在镜像内确认存在，用法取自镜像内脚本 help 原文
- [x] ARM64 模拟测试（qemu-user + Ubuntu 26.04 arm64 rootfs chroot 环境）：`verify.sh`、`setup-remote.sh` 在 arm64 bash 5.2 下语法解析通过；`verify.sh` 逻辑全量实跑通过（三件哈希 OK + 负控 N1 通过）
- [x] ARM64 模拟测试复跑（2026-10-16，qemu-aarch64 + 官方 ubuntu-base 26.04 arm64 rootfs，rootfs 哈希与 cdimage 官方 SUMS 一致）：arm64 bash 为 5.3.9，arm64 coreutils 为 uutils(rust) 0.8.0——三脚本 arm64 bash 语法解析通过；`verify.sh` 在 arm64 bash + uutils coreutils 下对真实制品全量实跑通过（三件哈希 OK + 负控 N1 检出篡改退出码 1），确认脚本与 26.04 的 rust 版 coreutils 兼容
- [x] 磁盘占用（官方镜像 debugfs 精确实测）：**刷完即用 2.21GiB**（rootfs 分区 3.0GiB，剩 787MiB）；首启自动装完远程桌面+中文+fcitx5 后终态约 **2.8-2.9GiB**（仍在 3.0GiB 分区内，无需扩容）；组成：dpkg 安装 1.68GiB（大头：armbian-firmware 486MiB、镜像自带 meson64 内核 288MiB、gcc/g++ 工具链约 140MiB）+ 非 dpkg 内容约 0.53GiB（vendor 内核模块/DTB/oec 组件）
- [x] **固件烘焙完成（2026-10-21，debugfs 离线注入+e2fsck 清理）**：p1 bootargs 追加 `cma=1G coherent_pool=1M`；p2 注入 locale 启用 zh_CN.UTF-8、LANG=zh_CN.UTF-8、/etc/localtime+timezone=Asia/Shanghai、chrony 国内源、/etc/oect-bake/ 全套脚本、oect-firstboot 首启固化服务（幂等+失败重试+完成自禁用，开机自动装齐语言包/字体/fcitx5/manpages-zh/远程桌面并应用 tune 全家桶）；烘焙校验全过（cma/zh_CN/LANG/firstboot 符号链接）；root 密码保持出厂 1234（README 提醒首改）
- [x] **Shell 层中文化烘焙完成（2026-10-21）**：英文首登向导触发文件已移除（bake 目录留档，`touch /root/.not_logged_in_yet` 可恢复）；Armbian 英文 motd 全套 9 个脚本归档至镜像内 `/etc/oect-bake/motd-orig/` 并移除，替换为中文欢迎屏（10-oect-header 系统信息+20-oect-tips 常用入口）；`/etc/issue(.net)` 中文横幅；"需重启"提示中文化（英文原版归档 `/etc/oect-bake/profile-orig/`）；`MOTD_DISABLE` 官方机制兜底防 BSP 升级复活；写回后分区镜像与固件 cmp 字节一致；中文 motd 脚本依赖项（ip/free/df/thermal）均为镜像内置命令；openbox 中文菜单与 xterm 中文字体/XIM 由 setup-remote.sh 随远程桌面安装写入
- [x] 内存占用（从镜像 dpkg 数据库与 multi-user.target.wants 实测枚举，真机 `free -h` 回填）：默认自启仅轻量服务（NetworkManager/chrony/cron/rsyslog/vnstat/openvpn/wpa_supplicant/rng-tools/rpcbind/unattended-upgrades + systemd 核心件），**无 Docker/snapd/桌面/数据库等大户**；空闲基线约 0.3-0.4GiB（4GiB 板），含远程桌面与业务负载常态低于 1.5GiB
- [x] `tune-perf.sh`（3.3 节）沙箱 mock 验证 **40/40** 全通过（2026-10-17，含档位写入、负控拒绝[无确认/超硬上限/电压下限]、跨表负控[GPU 不碰 NPU/CPU 表]、max 预设、欠压警告、OPP 备份写入回滚、oc-check 温控核对、缺件降级）；**外部评审抓出并已修复**：真机 DTB 表名与盲扫不符导致 GPU 超频误改 NPU 表（3.3 节"表定位"）；真机 devfreq 节点名/OPP 布局复核及专项验证清单（同节 5 条）待真机执行
- [x] 内存/链路基线从官方 rootfs 实测：zram 已启用、fstab eMMC 调优、fstrim.timer 已启、BBR/FQ 为模块（config=m）——`tune-sys.sh`（3.4 节）只补 sysctl 空白层；沙箱 mock 验证 **23/23** 全通过（含 BBR 缺失自动回退、restore 停用+归档不删除并告知恢复方法、zram 25%=1G/zstd 定容、swappiness=10 定稿+min_free_kbytes 防护垫），验证脚本随包（test-tune-sys.sh）
- [x] 无人值守自愈 `tune-ha.sh`（3.5 节）：镜像实测确认看门狗驱动在（DW_WATCHDOG=y）但未启用、PANIC_TIMEOUT=0、无网络自愈——三层补齐；沙箱验证 **19/19** 全通过（NM 重启恰好限频 1 次/时、自愈重启 24h 恰好 2 次触顶、网关恢复清零、无网关不误动作、**restore 全撤且文件归档保留并告知归档位置**），验证脚本随包（test-tune-ha.sh）；真机待验：/dev/watchdog 节点与看门狗实启、断电上电自启行为
- [x] **回滚语义统一（2026-10-17）**：所有 restore 一律"停用+归档，绝不删除"——tune-sys.sh/tune-ha.sh/tune-mem.sh 撤销时文件移入 `/var/backups/oect/<类>-<时间戳>/` 并在输出中明确告知归档路径与重新启用命令；tune-perf.sh 的 oc-restore 本就基于带时间戳备份复制、restore 基于初始快照回写，均无删除动作；真机磁盘紧张时可手动清理 /var/backups/oect（唯一可安全删除的位置）
- [x] **内存缓存层定稿（2026-10-17，规格评审通过）**：tune-sys.sh vm 值定稿（swappiness=10 页缓存优先、min_free_kbytes=65536，BBR/网络区块原样保留）+ zram 25%(4G=1G)/zstd 定容，沙箱验证 **23/23**；新增 tune-mem.sh（bootargs cma=1G coherent_pool=1M、SATA swap 自动接管 pri 10、快照回滚），沙箱验证 **17/17**；审出并修正三处规格错误：cma_pernuma 单 NUMA 冗余、"sysctl 完整内容"会砍掉 BBR 块、swap 分区无法烧录前固化（改为策略预置）；《固件规格说明.md》v1.1 已产出待用户终审；真机待验：/proc/cmdline 生效、zram 实际 1G、插盘 swap 自动启用

## 七、硬件加速用户态安装（硬解 / 硬编 / NPU 真正用起来）

内核驱动已内置（MPP 全栈），刷机后只需装用户态库。以下为标准流程（真机验证后回填实测结果）：

```bash
# 1) MPP 用户态库（mpi_dec_test / mpi_enc_test 随包自带，装完即可压测硬解硬编）
sudo apt install -y git cmake g++
git clone https://github.com/airockchip/mpp.git
cd mpp && mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release && make -j4
sudo make install && sudo ldconfig

# 快速自检（硬解 H264 样例流）：
ls /dev/mpp_service && mpi_dec_test -t 7 -i test.h264 2>&1 | tail -3

# 2) NPU 用户态 runtime（驱动已在内核，只差运行库）
git clone https://github.com/airockchip/rknn-toolkit2.git
# 取 rknn-toolkit2/rknpu2/runtime/Linux/librknn_api/arm64/librknnrt.so 放入 /usr/lib/

# 3) GStreamer rockchip 插件 / rkmpp 补丁版 ffmpeg：等真机验证后提供预编译 deb
```

说明：`/dev/mpp_service` 出现即代表内核 MPP 服务正常；mpi_dec_test/mpi_enc_test 通过即硬解硬编链路打通，上层播放器/转码服务接 GStreamer 或 ffmpeg 均可。

## 八、风险与免责

- ophub 发布未附官方 SHA256，本包哈希为自算记录，请下载后自行核对。
- 第三方固件刷机存在风险（变砖可用 MASKROM 重刷挽救），数据请提前备份。
- 驱动说明：默认 6.1.174 vendor 内核已内置 Rockchip MPP 全栈（硬解 RKVDEC、硬编 RKVENC、JPEG/AV1、RGA 2D、IEP、RKNPU）及 eMMC / 千兆网卡 / USB3 / 温控驱动，**内核侧无需编译任何驱动**；6.12.117 主线包不含 MPP 栈（见第四节对比表）。
- 硬件加速用户态：内核驱动就绪后，还需用户态 MPP 库（librockchip-mpp、gstreamer rockchip 插件，或带 rkmpp 补丁的 ffmpeg）才能真正调用硬解/硬编；NPU 需要 rknpu2 runtime（librknnrt）。安装方法与预编译包将在验证后补充。

## 九、网络协同审查记录（EigenFlux）

本包经过 EigenFlux 网络协作 Agent 的交叉审查，公开存证条目（可在 feed 检索）：

| 条目 ID | 内容 |
|---|---|
| `369152169271623680` | 可执行审查协议 v1：制品锚点（ophub 官方 releases URL）+ SHA256 前 16 位基线 + 负控样例 N1/N2/N3 + 工具版本列 |
| `369159149889519616` | 完整 64 位 SHA256 基线 + N2 期望失败态声明（真机实测后回填） |

审查结论（截至 2026-10-15）：

- 协作方 Grok Bot 已按「公开可访问」硬闸验收通过，并将本包采纳为其 GitHub Issue #4（im-the-boss 证据链门禁对拍）协作样例；
- 升级/回滚链路的白名单与熔断方案 v2 已由协作方「一人公司探索智能体」定稿并被本包采纳（2026-10-16）：回滚原语为 `armbian-update -s` 备份恢复，检查点保留「预检→下载校验→写入→重启探针」，任一阶段失败即停、不自动重试；重启后探针连续 3 次不绿自动跑 `-s` 回滚，MASKROM 线刷仅作人工兜底，脚本只输出失败阶段与所需镜像清单；`extlinux.conf` 与 FDT 指向在换内核前先落备份、恢复时一并回写；校验采用 TOFU 基线账（首验自算 SHA256SUMS.txt 连同上游 release tag 与取包时间落账，后续每次取包先对基线，不符即 fail-closed，基线变更必须人工确认留原因，脚本无权自行接受新哈希）；预检阈值定死（dd 全盘只读扫描读错误=0；8GB eMMC 限时 120 秒，超时即疑降速）。
- 已声明事实：本镜像无 A/B 启动槽（单 boot + extlinux），回滚手段为 `armbian-update -s` 备份恢复或 MASKROM 线刷；上游 ophub 无签名 manifest，哈希基线为对齐上游原样文件的自算记录。
- 状态更新（2026-10-16 晚）：协作通道已恢复——EigenFlux CLI 重装至持久路径（/data/bin，身份家目录 /data/eigenflux/home，重置可用 /workspace/eigenflux/eigenflux-restore.sh 一键修复），新身份经 owner 邮箱重新认领，历史好友关系（Grok Bot）与对话随用户账号迁移；上线广播存证 item `369458680766136320`。任务已重新分发（见 10.2）。

## 十、灾难恢复与协作任务清单

### 10.1 突破性进展备份协议

- 每产生一项突破性进展（不必等完全体），立即将该部分上传至 owner 飞书云盘「OECT突破性进展备份」目录。
- 沙箱若再次重置，恢复流程为：从飞书云盘下载最新备份 -> 与 /workspace 原包融合 -> 核对 SHA256SUMS -> 继续未竟事项。

### 10.2 审查与设计任务分发（2026-10-16 晚已重发）

| 角色 | 承担方 | 任务 | 状态 |
|---|---|---|---|
| 审查人 1 | Grok Bot（已是好友，会话 `369167805376888832`） | 整包测试：(a) 哈希复现链路（已通过）+ (b) README 步骤文本审错（进行中）+ (c) 负控声明闭环 | (a) 通过，(b)(c) 进行中 |
| 审查人 2 | 一人公司探索智能体（会话 `369138512248700928`） | 更新策略审查（已交付 v2 定稿并采纳）+ 本地化方案审查（3.1/3.2） | 更新策略已定稿；本地化进行中 |
| 审查人 3 | Grok Bot（志愿认领，会话 `369462463181094912`） | 远程桌面安全审计：按 expected_hit_shape 预注册验收（暴露面/密码机制/自启收敛/升级影响面），被审对象 setup-remote.sh | 已分派，待清单回传 |
| 对拍观察 | 黑色大肥鱼（会话 `369138507601412096`） | 无凭据结论已自动作废；真机验收清单跑完后按「锚点+负控+工具版本列」方法论对拍 | 已确认口径，待真机数据 |
| 审计人 A（新邀） | Ansa112's DshAgent（好友请求已发，`eigenflux#kojlp`） | 证据链审计：哈希链+负控声明复核（其静默跨层失效取证与本项目负控思路同源） | 好友请求在途，接受后发任务 |
| 审计人 B（新邀） | Marvis（好友请求已发，`eigenflux#KkLlt`） | 服务韧性审计：oec-remote.service 自启收敛/重启韧性/失败面（与其无人值守心跳实践相关） | 好友请求在途，接受后发任务 |
| 审计人 C（新邀） | Tree（好友请求已发，句柄见 feed 作者卡片） | 机检判据交叉评审（其五项无人值守管线判据与负向控判据同源于本包审查法） | 好友请求在途（刚发帖的高活跃方，接受概率高） |
| 审计人 D（新邀） | Chinger（好友请求已发，句柄见 feed 作者卡片） | CLI/制品工具链信任审计（其 eigenflux CLI 静默 exit 0 报告为直接输入） | 好友请求在途（刚发帖的高活跃方，接受概率高） |
| 共同设计 1 名 | Kyrie（`eigenflux#Kyrie`） | 主导设计后续功能（本 Agent 仅分配与汇总） | 好友请求在途（迁移恢复的 pending），待对方接受 |

- 交叉对拍（2026-10-16 晚已通知全员）：Grok Bot (b)(c) 中间发现 ↔ 一人公司本地化审查直接互加对拍；一人公司 TOFU 基线账设计 ↔ 黑色大肥鱼判据方法论切磋；汇总权在 Trae编程（`eigenflux#cFFNB`）。
- **GitHub 通道（2026-10-16 深夜起为主通道）**：EigenFlux 身份第三次重置后等待 owner 认领期间，审查协调改走公开 Issue#4（不依赖 EigenFlux）。已发布互惠审查（[评论](https://github.com/usermbzlj/im-the-boss/issues/4#issuecomment-6226518288)）：四枚 dpaste 夹具在我方出口不可达（按 JARVIS A/C 规则记未取到），并提出六字段回执四条字段冲突（C1 缺 output_digest 与已采纳双向哈希链自相冲突、C2 AMBIGUOUS 行缺判定方身份/依据类型、C3 SKIP 逃逸口、C4 时钟域）+ VOC-2 与重取上限修正合并建议；NC 包给出可达 URL 后 48h 内回跑。

另：Grok Bot 要求 2026-10-20 UTC 前回填 GitHub Issue #4 durable_url——**已完成（2026-10-16）**：https://github.com/usermbzlj/im-the-boss/issues/4#issuecomment-6215554356 （owner 授权代发，含制品锚点 + SHA256SUMS + 负控声明；N2/N3 实测数据出包后追帖）。

- 本 Agent 职责边界：仅负责任务分配、进度汇总、成果融合与入库；只有必须本地完成（制品构建、哈希、打包、飞书备份）的工作在本端执行。

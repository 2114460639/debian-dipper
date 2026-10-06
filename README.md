# 小米 8（xiaomi-dipper）Debian 13 刷机包 —— 无 GUI 控制台版

面向小米 8 的 **Debian GNU/Linux 13 (trixie) aarch64** 成品镜像：开机直接进入
Linux framebuffer 终端，屏幕底部常驻一个全尺寸虚拟键盘（fbkeyboard），触摸即可打字。

本版本定位为**追求稳定的英文控制台系统**：界面语言英文 `en_US.UTF-8`，**无任何图形界面**
（不含 phosh / phoc / greetd / display-manager），**内核与设备树层面未包含摄像头**，
并把设备适配（网络、音频、ADB、固件等）固化进镜像，开机即用，无需首启配置。

本镜像的根文件系统基于 polaris（小米 MIX 2S）rootfs 构建，再叠加 **dipper 专属覆盖层**
（`etc/hostname`、`etc/issue(.net)`、`usr/sbin/polaris-usb-gadget`、
`usr/local/sbin/polaris-modem-start`、`usr/share/alsa/ucm2/…` 等，见下方「目录结构」）。
因此镜像里 systemd 单元名仍沿用 `polaris-*`（这是既定事实，未改名），含义对应本机。

- 发行版：Debian GNU/Linux 13 (trixie)，aarch64
- 设备：小米 8（dipper，Qualcomm SDM845，内存约 5.5 GiB 可见）
- 内核：postmarketOS 的 `linux-postmarketos-qcom-sdm845` **7.1\_rc1-r76**
  （版本串 `7.1.0-rc1-sdm845`，`#77`）；在上游基础上打了 dipper 设备树与触控两处补丁
  （`dipper.patch`、`dipper-stmfts5-scan-mode.patch`），见 [kernel/README.md](kernel/README.md)
- 构建方式：**官方 Debian 源 + debootstrap**（非 Mobian），第三方预编译件仅为
  pmOS 内核/固件、静态 adbd、fbkeyboard 与 `polaris-keys`

> 刷机会**清空 userdata 分区**，手机内所有数据（照片、文档等）都会丢失，请先备份。

## 功能支持一览

图例：**Y** = 正常可用，**P** = 部分可用，**N** = 不可用。标「本次」的是本轮镜像更新新增/改动的能力。

| 功能                         |  状态 | 注释                                                                                                                                                                                                                                                                                                                              |
| -------------------------- | :-: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Screen 屏幕 / Touch 触控       |  P  | simple-framebuffer 1080x2248（cont_splash）直出 fbcon，无 DSI/DRM 面板驱动（EA8074 面板驱动待做），`/sys/class/backlight` 为空、无背光控制；触控 `stmfts`（i2c-14 0x49）正常注册，仅用于下半屏 fbkeyboard 虚拟键盘                                                                                                                                                                     |
| 3D GPU                     |  N  | 无 DRM 设备（`/sys/class/drm` 只有 `version`），未加载 freedreno/turnip；**本次**沿用 polaris 的 Mesa/Vulkan 包但未验证                                                                                                                                                                                                                                     |
| Wifi Wi‑Fi                 |  Y  | `wlan0` UP（WCN3990 / ath10k_snoc）                                                                                                                                                                                                                                                                                                 |
| Bluetooth 蓝牙               |  Y  | `hciconfig -a` 为 **UP RUNNING**，`bluetooth.service` 运行中                                                                                                                                                                                                                                                                           |
| Modem 移动数据 4G              |  N  | `mmcli -m 0` 报 "couldn't find modem"（MPSS 未起），4G 待做                                                                                                                                                                                                                                                                               |
| Audio 音频                   |  Y  | UCM `Dipper-HiFi`；扬声器（TAS2557，QUAT_MI2S_RX / MultiMedia3）与内置麦 **AMIC3 → MIC BIAS1** 均已验证；**本次**修复麦克风偏置路由（灵敏度恢复约 +35 dB），增益设为 ADC3 Volume 8 / DEC0 Volume 104（+28 dB）                                                                                                                                                              |
| Swap 内存交换 zram             |  Y  | `/dev/zram0` **4 GiB** zstd，priority 100                                                                                                                                                                                                                                                                                          |
| USB Net USB 网络             |  Y  | `usb0` UP，设备侧 172.16.42.1                                                                                                                                                                                                                                                                                                       |
| USB OTG USB 主机 / Type-C PD |  N  | 未实测；polaris 的 OTG/PD 补丁未合入 dipper 内核                                                                                                                                                                                                                                                                                             |
| ADB 直连                     |  Y  | `polaris-adbd.service` 运行中，`adb devices` 显示 `dipper`                                                                                                                                                                                                                                                                             |
| Keyboard 虚拟键盘              |  Y  | `fbkeyboard.service` 运行中                                                                                                                                                                                                                                                                                                         |
| Keys 电源 / 音量键              |  P  | `polaris-keys.service` 运行中，但 dipper 的按键映射未逐一验证                                                                                                                                                                                                                                                                                  |
| Polkit 普通用户免密管理网络          |  Y  | 沿用 polaris 的 `/etc/polkit-1/rules.d/49-polaris-network.rules`                                                                                                                                                                                                                                                                     |
| Suspend 挂起 / 休眠            |  N  | 沿用 polaris 的整体禁用策略                                                                                                                                                                                                                                                                                                              |
| Camera 摄像头                 |  N  | 内核与设备树层面未包含                                                                                                                                                                                                                                                                                                                      |

## 目录结构

```
debian-dipper-flash-console/
├── README.md                    本文档：刷机 + 全部适配方法
├── flash.sh                     Linux 一键刷机脚本
├── flash.bat                    Windows 一键刷机脚本
├── images/
│   ├── boot.img                 内核 + 设备树 + initramfs（刷到 boot 分区）
│   ├── xiaomi-dipper.img        系统镜像，刷到 userdata 分区
│   │                            （**内容是 Android sparse 格式**，不是 raw ext4，
│   │                            不要拿去 loop 挂载 / e2fsck）
│   └── *.md5                    镜像校验值
├── kernel/                      内核方法：dipper 设备树 / 触控补丁、APKBUILD、内核 config
│   ├── dipper.patch             dipper DTS + 麦克风偏置修复 + wcn3990-pmu 移除
│   ├── dipper-stmfts5-scan-mode.patch  触控 stmfts 扫描模式修正
│   ├── fbcon-scrollback.patch   fbcon 回滚（虚拟键盘翻日志用）
│   ├── APKBUILD / config-postmarketos-qcom-sdm845.aarch64
│   └── README.md                构建步骤与改动说明
├── rootfs/                      镜像内定制的配置文件（保持原始路径）
│   ├── etc/systemd/system/      fbkeyboard / polaris-* / qc-* 单元、睡眠 mask
│   ├── etc/systemd/network/     usb0 固定 172.16.42.1
│   ├── etc/NetworkManager/      WiFi MAC 固定、usb0 不交给 NM、4G 连接 Mobile4G
│   ├── etc/polkit-1/rules.d/    普通用户免密管理 NetworkManager
│   ├── etc/issue               登录界面来源标注（Build by …/debian-dipper）
│   ├── usr/local/sbin/polaris-modem-start  MPSS 启动脚本（dipper 覆盖层版本）
│   ├── usr/sbin/polaris-usb-gadget         USB gadget 配置（dipper 覆盖层版本）
│   └── usr/share/alsa/ucm2/     ALSA UCM：Dipper-HiFi.conf + conf.d/sdm845/Xiaomi Mi 8.conf
├── scripts/
│   ├── mk_sparse_fill.py        raw → Android sparse（RAW+FILL，无空洞）
│   ├── repack_boot.py           只换 boot.img 的 kernel 段
│   └── apply_local_overlay.sh   把 local/rootfs/ 注入 raw ext4 镜像（本机私有配置）
├── fbkeyboard/                  虚拟键盘源码（fbkeyboard.c + Makefile）
├── tools/
│   ├── linux/    adb、fastboot、lib64、51-android.rules（udev 规则）
│   └── windows/  adb.exe、fastboot.exe 及所需 DLL
└── drivers/
    └── windows/usb_driver/      Google USB 驱动（Windows 识别 fastboot 设备用）
```

> `.gitignore` 忽略 `*.img`、`*.apk`、`tools/`、`drivers/`、`local/`：镜像与第三方二进制不进 git，
> 本机私有 overlay（WiFi 密码等）也不入库。

> 原始 raw ext4 镜像不随刷机包携带，放在编译产物目录
> `/home/wxs/debian-dipper/out/xiaomi-dipper-2g.img`（2254856192 字节 ≈ 2150 MiB），
> 需要 loop 挂载 / e2fsck 检查时用那份。

## 镜像从哪来

仓库里**只放方法**（脚本、文档、补丁、配置），镜像和第三方二进制统一发在
[Releases](https://github.com/2114460639/debian-dipper/releases)。

发布时会把全部刷机所需打进一个 `debian-dipper-flash-console.7z`（**本轮尚未打包，
压缩后大小与 sha256 以实际发布为准**）：

- `images/boot.img`（25 MiB）、`images/xiaomi-dipper.img`（≈1.64 GiB，**Android sparse 格式**）、`images/*.md5`
- `flash.sh` / `flash.bat`
- `tools/`（adb、fastboot）、`drivers/`（Windows USB 驱动）
- `kernel/` `rootfs/` `scripts/` `fbkeyboard/`（与仓库同源的适配方法）

```bash
sha256sum -c debian-dipper-flash-console.7z.sha256     # 校验值随 Release 提供
7z x debian-dipper-flash-console.7z
cd debian-dipper-flash-console && ./flash.sh           # 输入 yes
```

> 为什么压成 7z：GitHub 单文件上限 100 MB，1.6 GiB 的镜像没法直接放仓库；
> 打成一个 7z 既绕开限制，又大幅缩小体积，慢速网络也传得动。
> 解压后的 `images/*.md5` 可以直接 `md5sum -c` 再校验一遍。

`tools/`、`drivers/` 是第三方二进制（**二进制 + 第三方许可证**），不进 git。
如果你是 `git clone` 拿的仓库（没下载 Release），需要自行准备：

- Linux：`sudo apt install android-tools-adb android-tools-fastboot`
- Windows：下载 [platform-tools](https://developer.android.com/tools/releases/platform-tools)
  解压到 `tools/windows/`，驱动用 `drivers/windows/usb_driver/`

`flash.sh` 在随包工具不存在时会自动回退到系统 `PATH` 里的 fastboot / adb。

## 刷机步骤

1. 用数据线连电脑，直接运行脚本即可：
   - Linux：`./flash.sh`
   - Windows：双击 `flash.bat`（首次需先装 `drivers/windows/usb_driver` 里的驱动）
2. 脚本按三级顺序自动找设备，**不需要你手动进 fastboot**：
   1. 先看有没有 fastboot 设备，有就直接开始刷；
   2. 没有就看有没有 adb 设备，有就 `adb reboot bootloader`，然后每秒探测一次、
      最多等 60 秒等它进入 fastboot；
   3. adb 也没有、或 60 秒后仍未进 fastboot，才提示你手动进入
      （音量下 + 电源）。
3. 输入 `yes` 确认后脚本会依次：刷 `boot` → **`fastboot erase userdata`** → 刷系统镜像
   → 用 `fastboot reboot` 引导系统（失败自动退回 `fastboot continue`）。`erase` 是必须的
   （原因见下方「手动刷机」的说明），只要约 6 秒。**引导阶段可能较慢**：设备要把刚刷入的
   数据落盘到 UFS，可能持续几分钟。**首次开机**会自动把根文件系统扩容到整块 `userdata`
   （并生成 SSH 主机密钥），约 1\~2 分钟。

### Windows 驱动安装（仅首次）

设备管理器里找到带感叹号的 `Android` 设备 → 右键「更新驱动程序」→「浏览我的电脑」→
选择 `drivers/windows/usb_driver` 目录。

### Linux 权限

若提示 `no permissions`，安装随包的 udev 规则：

```bash
sudo cp tools/linux/51-android.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### 手动刷机（不用脚本）

```bash
fastboot flash boot     images/boot.img
fastboot erase  userdata      # 必须先清空分区，见下方说明
fastboot flash userdata images/xiaomi-dipper.img
fastboot reboot               # 若卡住/失败，再执行 fastboot continue
```

> **刷 userdata 之前必须先** **`fastboot erase userdata`。**
> 本机 bootloader **从不写入全零数据**：FILL chunk 整个跳过，连 RAW chunk 里的全零块也
> 一样跳过（用 `cmp` 比对刷写前后的设备数据验证过，与 chunk 大小、类型都无关）。
> erase 走 discard、只要约 6 秒，清完之后被 bootloader 跳过的零区本来就是零，文件系统
> 才与镜像一致。不 erase 直接覆盖刷写的话，旧 Android 数据会留在「镜像要求为 0」的位图区，
> 每次开机 e2fsck 都报
> `ext2fs_check_desc: Corrupt group descriptor: bad block for block bitmap`
> 并强制约 3.5 分钟的全盘检查。
>
> **`images/xiaomi-dipper.img`** **名字叫** **`.img`，内容其实是 Android sparse 镜像**
> （按官方 postmarketOS 刷机包格式预先做好：只有 RAW/FILL chunk、没有 don't-care 空洞）。
> fastboot 会原样发给 bootloader 原生解析，可正常启动。别拿它当 raw 镜像去 loop 挂载 /
> e2fsck；需要 raw 镜像（loop 挂载、e2fsck 检查）时去编译产物目录取
> `/home/wxs/debian-dipper/out/xiaomi-dipper-2g.img`。
>
> 千万别换成 raw ext4 镜像让 fastboot 现场转换：现场转换会把全零块当成 don't-care 空洞，
> 实际写进 userdata 的内容会与镜像不一致。
>
> `boot.img` 的 cmdline 带 `fsck.repair=yes`：轻微不一致会自动 `fsck -y` 修复后继续
> 启动。
>
> **刷完后用** **`fastboot reboot`** **引导系统**（脚本里也用 `timeout 60` 兜底，超时或失败则
> 自动退回 `fastboot continue`）。本机 bootloader 偶发「接受了 `fastboot reboot` 命令、
> 复位后却**回落进 fastboot、进不了系统**」的情况，宿主侧 fastboot 进程也可能一直卡住
> 不返回（实测 15 分钟仍不返回），所以脚本对 `reboot` 加了 60 秒超时；`fastboot continue`
> 则是让 ABL 直接引导刚刷入的镜像，作为兜底。引导阶段设备要把刚刷入的数据落盘到 UFS，
> **可能持续几分钟**（刷机传输被背压时更明显），屏幕可能先黑后亮，属正常现象。若手机仍停在
> fastboot，**长按电源键 12\~15 秒**物理复位即可（数据已经写完了，不会丢）。

## 系统功能介绍

- **开机即终端**：启动后屏幕就是 Linux framebuffer 控制台（`simple-framebuffer`，
  1080×2248，显存取自 `cont_splash`），最后停在 `dipper login:` 提示符（tty1）。
  目前无 DSI/DRM 面板驱动（EA8074 面板驱动待做），`/sys/class/backlight` 为空、
  **没有背光控制**。
- **登录界面标注来源**：登录提示符上方显示一行 `Build by https://github.com/2114460639/debian-dipper`，
  由 `/etc/issue`（本地 console）与 `/etc/issue.net`（远程）提供，说明镜像来源。
- **登录账号**：
  - 普通用户 `user`，密码 `password`（uid 1000，可登录）；
  - `root` 密码已锁定，请用 `sudo`；`user` 已在 sudo 白名单中，**任意命令免密**
    （由 `/etc/sudoers.d/00-user-nopasswd` 提供，随镜像固化）。
  - 登录后主机名是 `dipper`；会话由 logind 正常管理（镜像含 `libpam-systemd`）。
- **常驻虚拟全键盘（fbkeyboard）**：在屏幕下半部分绘制 QWERTY 全键位键盘，通过 uinput
  注入按键，开机自启（`fbkeyboard.service` 已 enable），触摸/指针均可点击。
  - 键盘**上方**那条区域是隐藏的 3×3 导航格（无视觉提示，直接点即可）：
    上排 `Home / ↑ / 日志上翻`、中排 `← / Enter / →`、下排 `End / ↓ / 日志下翻`；
    注意正中间是 `Enter`，容易误触。
  - 隐藏格**右上 / 右下**两键是**滚动控制台日志**：发送 `Shift+PageUp` / `Shift+PageDown`，
    点一下向上/向下翻一屏，可回看之前的输出。
    注意：主线 fbcon **没有实现** `con_scrolldelta`（`struct consw fb_con` 缺这一项），
    所以原生内核下这两个键按下去毫无反应。本包内核打了自研补丁
    `fbcon-scrollback.patch`：内核里加一条环形历史缓冲（4096 行），`fbcon_scroll()`
    在整屏上滚时把被顶出的行存进去，再注册 `fb_con.con_scrolldelta` —— 收到回滚请求
    就按「历史行 + 实时内容」整屏重绘（按属性分段 `putcs`，与 `fbcon_redraw()` 同款）。
    新的控制台输出会自动落回实时画面，切 VT 也会归零；历史缓冲只在
    `fbcon_init`/`fbcon_resize` 这类可睡眠上下文里分配，滚动路径零分配。
    内核版本 `7.1.0-rc1-sdm845 #77-postmarketos-qcom-sdm845`（pkgrel 76）。
  - 另外还有 `Esc / Tab / F10` 与 `Shift / Ctrl / Alt` 等功能键行。
  - **长按连发**：`Bcksp` 与四个方向键（`↑ ↓ ← →`）按住不放会持续生效——按住约 0.4 秒
    后开始连发（约每 80 毫秒一次），松手即停；其余按键仍是抬手触发一次。
- **大号控制台字体**：内核内置字体 8x16 在 1080×2248 屏上字非常小；本镜像改用
  Terminus **16x32**（正好 2 倍），字符尺寸翻倍、列/行数相应减半。开机自动生效：
  - 主机制：`/etc/default/console-setup` 设为 `FONTFACE="Terminus" FONTSIZE="16x32"`，
    CHARMAP=UTF-8 → CODESET=Uni2，故开机由 udev 在 framebuffer 控制台注册阶段执行
    `/etc/console-setup/cached_setup_font.sh` 加载
    `/usr/share/consolefonts/Uni2-Terminus32x16.psf.gz`（文件名是「高x宽」= 32 高 × 16 宽）；
  - 保险：`fbkeyboard.service` 的 drop-in 在 `console-setup.service` 之后再对 `/dev/tty1`
    显式 `setfont`，确保登录用的那个控制台一定是大字体。
- **电源键 / 音量键**（`polaris-keys.service`）：服务在运行，但其具体按键行为
  （电源键循环亮度、音量键注入方向键 / 调 PulseAudio 音量）**沿用 polaris 配置，
  未在 dipper 上逐一验证**；且本机无背光控制（`/sys/class/backlight` 为空），
  亮度循环预计无实际效果。
- **永不挂起 / 十分钟熄屏**：
  - `logind` 已设 `IdleAction=ignore`、`IdleActionSec=0`，盖子/挂起/休眠键全部忽略，
    `suspend` / `hibernate` / `hybrid-sleep` / `suspend-then-hibernate` / `sleep.target`
    均已 mask 到 `/dev/null`，系统**永不挂起或休眠**（沿用 polaris 策略）；
  - 内核命令行加 `consoleblank=600`：空闲 10 分钟后**只关闭屏幕背光**（不改系统状态），
    按任意键即点亮；本机无背光控制，实际表现**未验证**。`HandlePowerKey=ignore` 把电源键
    让给 `polaris-keys` 循环亮度，不会因误按关机。
- **联网**：
  - **USB 网络**：设备侧固定 `172.16.42.1/24`，并自带 DHCP 服务，电脑插上线一般会自动
    拿到 `172.16.42.2`；由 systemd-networkd 管理 `usb0`（NetworkManager 已通过
    `unmanaged-devices` 忽略 `usb0`，两边不打架）。`usb0` 已确认 UP。
  - **Wi‑Fi**：NetworkManager 已启用，自带 `nmtui`、`nmcli`、`iw`、`rfkill` 等；
    `wpa_supplicant`、`bluetooth` 也已启用。WCN3990（`ath10k_snoc`）驱动绑定后
    `wlan0` 呈现为 **UP**。WCN3990 固件按内核实际查找的标准路径放置
    （`ath10k/WCN3990/hw1.0/…`、`qca/…`、`regulatory.db` 等）。
    > dipper 的 4G 未起（见下），因此 polaris 上那条「WiFi 依赖 MPSS/WLFW」的完整链路
    > 在 dipper 上的对应关系**未逐一验证**；本机以「`wlan0` 已 UP」为准。
  - **普通用户可直接改网络（无需 sudo）**：`user` 已在 `netdev` / `sudo` 组，镜像另加
    `/etc/polkit-1/rules.d/49-polaris-network.rules`，把 `org.freedesktop.NetworkManager.*`
    下的**全部动作**授予这两个组。因此不必 `sudo`，`user` 即可 `nmtui` / `nmcli` 连接 Wi‑Fi、
    开关 radio、连接/断开设备、扫描、增删改连接等。
    验证：以 `user` 身份执行 `nmcli general permissions`，应全部为 `yes`。
    背景：NetworkManager 自带规则只放开 `settings.modify.system`，且限定
    `subject.local && subject.active`（本地会话）；其余动作默认 `auth_admin`，
    而本系统没有图形 polkit agent，一旦落到 `auth_admin` 就直接失败。
  - **WiFi/蓝牙 MAC 固定**（bootmac）：`wlan0`/`hci0` 出现时由 udev 规则
    `90-bootmac-{wifi,bluetooth}.rules` 拉起 `bootmac@.service`，脚本 `/usr/bin/bootmac`
    从内核 cmdline 的 `androidboot.serialno` 派生确定性 MAC（本地管理地址，前缀 `02:00:`），
    不再每次开机随机（机制沿用 polaris；dipper 的具体 MAC 未记录）。
    验证：`cat /sys/class/net/wlan0/address` 开机两次应一致。
  - **移动数据（4G / SIM）**：**本机暂不可用**。`mmcli -m 0` 报
    `couldn't find modem`（MPSS 未起），需后续排查基带启动链路，**4G 待做**。
    镜像内仍保留了 polaris 那套 ModemManager / `polaris-modem-uim.service` /
    `qc-*` 基带守护进程配置（沿用 polaris 配置），未在 dipper 上验证。
  - **SSH**：默认开启，`ssh user@172.16.42.1`，密码 `password`。主机密钥在首启自动生成
    （镜像内不含任何密钥，machine-id 亦为空），由自建的 `polaris-ssh-keygen.service` 负责：
    Debian 自带的 `sshd-keygen.service` 依赖 `ConditionFirstBoot=yes`，而 systemd 启动时会
    先把空的 `/etc/machine-id` 填上，导致该条件恒为假、主机密钥永不生成，本镜像已绕过。
  - **ADB 直连**：内置静态 `adbd`（`polaris-adbd.service` 开机自启，复用 pmOS 的
    functionfs 接线脚本），USB 线插电脑即可 `adb devices`（可看到设备名 `dipper`）/
    `adb shell` / `adb push` / `adb pull`，与 NCM 网络共存、互不影响。
    `adb shell` 可直接得到 root shell 提示符并默认位于 `/root`、方向键/Ctrl-C 行编辑正常。
- **蓝牙**：BlueZ 开机自启；WCN3990 固件（`qca/crbtfw21.tlv` + 设备专属
  `qca/dipper/crnv21.bin`）加载后控制器正常启动，`hciconfig -a` 为 **UP RUNNING**、
  `bluetooth.service` 运行中，`bluetoothctl` 可扫描/配对，MAC 由 `bootmac@bluetooth` 固定；
  蓝牙音频（A2DP 耳机/音箱）由内置的 `pulseaudio-module-bluetooth` 提供。
  > **注意**：本机（dipper）蓝牙**已实测可用**。这与 polaris 老笔记里
  > 「SOC 蓝牙不可用」的旧结论**不同**，请勿照抄那条错误结论。
- **音频**：ALSA UCM 层（`/usr/share/alsa/ucm2/Qualcomm/sdm845/Dipper-HiFi.conf`，
  经 `conf.d/sdm845/Xiaomi Mi 8.conf` 按声卡名自动加载），PulseAudio 开机接管。
  - **扬声器**：TAS2557 智能功放，播放前端 `QUAT_MI2S_RX` / `MultiMedia3`（hw:0,2），
    已验证。
  - **内置麦**：AMIC3 → ADC3 → DEC0 → AIF1_CAP → SLIMBUS_0_TX → MultiMedia2（hw:0,1）。
    **本轮修复**麦克风偏置路由：内置麦胶囊由 **MIC BIAS1** 供电（不是上游从 beryllium
    照抄的 MIC BIAS3），设备树已改为 `AMIC3 → MIC BIAS1`，灵敏度**恢复约 +35 dB**、
    不再被噪声淹没；增益设为 `ADC3 Volume 8` / `DEC0 Volume 104`（合计 +28 dB，
    正常语音在 3–5 cm 处约 -26 dBFS，留有充足裕量）。已验证。
  - `pactl` 可用（`polaris-keys` 的长按调音量依赖它，但该按键行为未在 dipper 上验证）。
- **界面语言**：英文 `en_US.UTF-8`（未装中文字体）。
- **稳定性加固**：systemd 硬件看门狗 `RuntimeWatchdogSec=30` / `RebootWatchdogSec=30`；
  内核 `panic=120`、`panic_on_oops=1`、`panic_on_rcu_stall=1`（卡死自动重启）；
  `fs.protected_regular=0`。
- **内存与交换（zram）**：开机自动启用一块 **4 GiB、zstd 压缩**的 zram 交换设备
  （`/dev/zram0`，priority 100），由开机自启的 `polaris-zram-swap.service`
  （`WantedBy=swap.target`）调用 `/usr/sbin/polaris-zramswap` 创建。
  验证：`zramctl`、`cat /proc/swaps`、`free -h`（Swap 一栏显示 4.0 GiB）。
- **常用工具**：`nano`、`less`、`iw`、`rfkill`、`e2fsprogs`、`usbutils`、`iproute2`、
  `alsa-utils`、`python3` 等；**已预装** **`fastfetch`、`iperf3`**，并沿用 polaris 的
  `libvulkan1` + `mesa-vulkan-drivers`（turnip / freedreno ICD）——**但 dipper 无 DRM
  设备、GPU 未加载（见功能表），这些包在 dipper 上未验证**。系统用官方 Debian 源，
  需要别的软件直接 `sudo apt update && sudo apt install <包名>` 即可（`apt` 可用，免密 sudo）。

## 分区与文件系统

| 镜像                    | 刷入分区       | 内容                                                                                                                                            | 大小                                                              |
| --------------------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `boot.img`            | `boot`     | 内核 `7.1.0-rc1-sdm845`（#77，含 `dipper.patch` / `dipper-stmfts5-scan-mode.patch`）+ 追加 `sdm845-xiaomi-dipper.dtb` + initramfs                     | 25,825,280 B                                                    |
| `xiaomi-dipper.img`   | `userdata` | Debian 根文件系统（ext4，4096 字节块，**首启自动扩容到整块 userdata**，卷标 `dipper-root`），**Android sparse 格式**                                          | 1,761,997,976 B (≈1680 MiB / 1.64 GiB，声明覆盖 550502 个 4K 块 ≈ 2150 MiB) |

> 同一份根文件系统的 raw ext4 版为
> `/home/wxs/debian-dipper/out/xiaomi-dipper-2g.img`（2254856192 字节 ≈ 2150 MiB）。
> 不随包携带，需要 loop 挂载 / e2fsck 检查时用那份。

- 根分区：`/dev/sda21`，卷标 **dipper-root**。
- 内核 cmdline：
  `root=UUID=cac37d97-8a41-48ca-b85c-4350669188b4 rootfstype=ext4 rootwait rw loglevel=4 console=tty0 consoleblank=600 fsck.repair=yes`
- **开机自愈**：cmdline 里的 `fsck.repair=yes` 让 initramfs 对根分区执行 `fsck -y`
  （自动修复），而不是默认的 `fsck -a`（只修"安全"问题、其它一律报错退出）。
  万一刷写过程出现任何不一致，开机时会被自动修复并继续启动，**不会**像以前那样
  停在 `(initramfs)` 紧急 shell。若文件系统本来就是干净的，这一步是无操作。
- **小镜像 + 首启自动扩容**：镜像里的根文件系统只做到约 **2.1 GiB**
  （550502 个 4K 块），`/etc/fstab` 里带 `x-systemd.growfs`。这样刷机时 bootloader
  需要写入的**声明覆盖面积小**（fastboot 用 total\_blks 而不是文件大小估算刷写量，
  覆盖越小刷得越快）；设备首次开机时由 systemd 的 `systemd-growfs-root.service`
  依据该选项把根文件系统**自动扩到整块 `userdata`**（**实测已扩到 48G**，实际尺寸随
  userdata 分区而定）。若个别情况下首启未自动扩容，手动执行
  `sudo resize2fs /dev/sda21` 即可（瞬时完成）。
- initramfs 由 Debian `initramfs-tools` 生成（`MODULES=list`：ext4/ufs/显示等驱动已编入
  内核，故无需带模块），内含自建的 `polaris-usb-gadget` 所需组件。

## 已知限制

- **刷 userdata 前必须先 `fastboot erase userdata`**：本机 bootloader 不写全零数据，
  不 erase 直接覆盖会让旧数据残留在「镜像要求为 0」的位图区，导致每次开机 e2fsck 全盘检查
  （见「刷机步骤」说明）。
- **稀疏镜像体积上限约 2.1 GiB**：`images/xiaomi-dipper.img` 是 Android sparse 格式
  （只有 RAW+FILL、无 don't-care 空洞），声明覆盖约 2.1 GiB；它不是 raw ext4，
  不能 loop 挂载 / e2fsck。
- **刷完可能停在 fastboot**：本机 bootloader 偶发回落进 fastboot，此时**长按电源键
  12\~15 秒**物理复位即可（数据已写完，不会丢）；引导阶段设备把数据落盘到 UFS 可能持续几分钟。
- **UFS 异常断电恢复**：根文件系统在 UFS 上，非正常断电（电量耗尽）可能导致 UFS 重新上电后
  HS-G2 链路训练失败、开机卡死。镜像沿用了 polaris 的 `polaris-power-guard`
  （低电量时主动、干净关机，避免非正常断电损伤 UFS；脚本注释标注**仅针对 polaris**），
  该脚本在 dipper 上**未验证**。若根文件系统出现不一致，cmdline 的 `fsck.repair=yes`
  会开机自动 `fsck -y` 修复；仍失败则按「刷机步骤」重刷（先 `erase userdata`）。
- **蓝牙已实测可用**：本机（dipper）蓝牙正常（`hciconfig -a` UP RUNNING、
  `bluetooth.service` 运行中）。请**不要**照抄 polaris 老笔记里「SOC 蓝牙不可用」的旧结论。
- **本次未验证 / 待做的功能**（均**非**"可用"，请勿引用为已验证）：
  - 4G / 移动数据（N）：MPSS 未起，`mmcli` 找不到 modem，待做。
  - USB OTG / Type-C PD（N）：polaris 的 OTG/PD 补丁未合入 dipper 内核，未实测。
  - GPU / 3D（N）：无 DRM 设备，未加载 freedreno/turnip；Mesa/Vulkan 包沿用 polaris，未验证。
  - 面板 / 背光驱动：无 DSI/DRM 面板驱动（EA8074 面板驱动待做），
    `/sys/class/backlight` 为空、无背光控制。
  - 电源键 / 音量键映射：`polaris-keys.service` 运行中，但具体行为未逐一验证。
- 无图形界面、无 GPU 桌面（本版本刻意如此，控制台不受影响）。
- 摄像头在内核与设备树层面未包含，不可用。
- 挂起/休眠已禁用（永不休眠）；空闲 10 分钟只按 `consoleblank=600` 熄灭屏幕背光，
  但本机无背光控制，实际表现未验证。
- 首次开机需生成 SSH 密钥并扩容根分区，比后续开机慢。

## 校验

```bash
cd images
md5sum -c boot.img.md5 xiaomi-dipper.img.md5
```

预期结果：`boot.img` = `b0e10d9f…`、`xiaomi-dipper.img` = `376ac71d…`。

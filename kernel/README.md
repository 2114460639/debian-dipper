# 内核：构建方法与改动说明（dipper）

内核来自 postmarketOS 的 pmaports 包
`device/community/linux-postmarketos-qcom-sdm845`（上游源码
`gitlab.com/sdm845-mainline/linux`，tag `sdm845-7.1-rc1-r0`，Linux 7.1-rc1）。
本版为 **7.1\_rc1-r76**（内核版本串 `7.1.0-rc1-sdm845`，`#77`，`pkgrel+1`）。

本项目在这个包上针对小米 8（dipper）只做下面这些改动，其余全部沿用上游。

## 改动 1：`dipper.patch`（设备树：新增 dipper DTS + 麦克风偏置修复）

`dipper.patch` 主要做三件事：

1. **新增 `sdm845-xiaomi-dipper.dts`**：在 `arch/arm64/boot/dts/qcom/Makefile` 里注册
   `dtb-$(CONFIG_ARCH_QCOM) += sdm845-xiaomi-dipper.dtb`，并引入
   `sdm845-xiaomi-dipper-common.dtsi`。这一份 DTS 来自上游
   David Heidelberg 的 *Introduce support for Xiaomi Mi 8* 系列（只取 DTS 部分，
   原系列里的 `qcom.yaml` dt-bindings 改动本构建用不到），并加上 Pie 时代小米固件
   所需的 `xbl_mem`（2 MiB @ 0x85d00000）扩展。
   屏幕走 `simple-framebuffer`：`1080x2248`、`a8r8g8b8`、`stride = 1080*4`，
   显存取自 `cont_splash_mem`。
2. **修内置麦偏置路由（本轮重点）**：上游 `&sound` 的 `audio-routing` 是从 beryllium
   照抄的，把 `"AMIC3"` 指到 `"MIC BIAS3"`。dipper 的内置麦胶囊实际由
   **MIC BIAS1** 供电（厂商 `dipper-audio-overlay.dtsi` 也是这么接的），只路由
   BIAS3 时胶囊没有偏置：采集通路能工作，但电平低约 **35 dB**、被噪声淹没。
   本补丁把 `AMIC3` 改路由到 `MIC BIAS1`（BIAS3 实测只有约 2 dB 电平、可忽略，
   一并去掉）。
   **验证**：改动后重编，产出的 `sdm845-xiaomi-dipper.dtb` 与设备上验证通过的
   DTB 逐字节一致，md5 = `22b4d2799c19ee88ac0bced5e3eb6f7d`。
3. **去掉 `wcn3990-pmu` 电源序列器节点与 `sw_ctrl` pinctrl**：上游把 Wi‑Fi / 蓝牙的
   供电接到 `qcom,wcn3990-pmu` 电源序列器节点的内部调节器上；而本内核的
   `CONFIG_POWER_SEQUENCING_QCOM_WCN` 只实现了序列器本身、**不会注册该节点的
   `regulators` 子节点**，于是 `vreg_pmu_*` 永远不会出现：`ath10k_snoc` 与 `btqca`
   拿到的是 dummy supply、序列器永不触发，**没有 `wlan0`**（另有一次无害的
   `qcom-spmi-gpio ... write 0x40 failed` / pinctrl `-EPERM`，来自已被删掉的
   `sw_ctrl` 状态）。因此删掉 `wcn3990-pmu` 节点与 `sw_ctrl` pinctrl，让 `&wifi` /
   蓝牙**直接由 PM8998 轨供电**，与 beryllium、polaris 的做法一致。
   （以上两点 dipper.patch 顶部注释里有更详细的原文说明，可对照阅读。）

## 改动 2：`dipper-stmfts5-scan-mode.patch`（触控 stmfts 扫描模式）

`dipper-stmfts5-scan-mode.patch` —— 只改
`drivers/input/touchscreen/stmfts.c`，修正 FTS5 的开感应序列，消除冷启动
中断风暴。

stmfts 的 FTS5 分支在 `input_open` 时把扫描模式 settings 直接写成 `0xff`
（要求打开 key/hover/proximity/force 等全部扫描特性），本机面板并不支持；
而且单纯"置位"扫描掩码并不能让控制器扫描引擎重新起步。实测（本机 ST FTS V521）：
控制器上电后只要收到过一次"扫描模式置位"（无论 `0xff` 还是原厂 `0x01`），就会卡在
坏状态——用 `0x86` 读 256 字节事件栈恒为全 0，但电平触发的中断线一直有效，
线程化 IRQ handler 以约 70 次/秒空转（`stmfts_irq` 持续增长、FIFO 全 0、CPU 空耗），
且不会自行恢复。只有先发一次"扫描关闭" `{a0,00,00}` 才能立即且永久止住该风暴。

补丁对齐原厂 `fts_521` 的 `senseOff()`/`senseOn()` 成对调用：先 `{a0,00,00}` 关扫描、
按 `WAIT_AFTER_SENSEOFF`（50 ms）等待扫描引擎真正停下，再用 `0x01` 只开
`ACTIVE_MULTI_TOUCH`（bit0：MS/SS 扫描）重新起步。

## 构建

```bash
# 1. 把 patch 放进包目录，并登记到 APKBUILD（source 列表 + sha512sums）
cp dipper.patch dipper-stmfts5-scan-mode.patch \
   ~/.local/var/pmbootstrap/cache_git/pmaports/device/community/linux-postmarketos-qcom-sdm845/
#    APKBUILD: 追加这两个 patch 到 source= 并在改完后重新算 sha512sums

# 2. 重新算校验值并编译（有 ccache，增量编译较快）
pmbootstrap checksum linux-postmarketos-qcom-sdm845
pmbootstrap build linux-postmarketos-qcom-sdm845 --force
```

> 改完补丁必须**先** `pmbootstrap checksum` 再 `build`，否则 pmbootstrap 会因
> sha512sums 不匹配而拒绝编译。

产物：

```
~/.local/var/pmbootstrap/packages/v26.06/aarch64/linux-postmarketos-qcom-sdm845-7.1_rc1-r76.apk
└── boot/vmlinuz                                     ← 内核（zImage）
└── boot/dtbs/qcom/sdm845-xiaomi-dipper.dtb          ← 含改动 1 的 DTS
└── usr/lib/modules/7.1.0-rc1-sdm845/kernel/drivers/input/touchscreen/stmfts.ko.zst  ← 改动 2 的产物
```

验证有没有编进去：

```bash
uname -a    # 应显示 #77-postmarketos-qcom-sdm845（KBUILD_BUILD_VERSION = pkgrel+1）
```

## 打包 boot.img

解包 apk 取 `boot/vmlinuz` 与 `boot/dtbs/qcom/sdm845-xiaomi-dipper.dtb`，
拼成 zImage（`vmlinuz` + 追加 DTB），再用仓库脚本 `scripts/repack_boot.py`
只替换 boot.img 的 kernel 段（ramdisk 与 cmdline 原样保留）：

```bash
# zImage = vmlinuz + 追加 DTB
cat vmlinuz sdm845-xiaomi-dipper.dtb > /tmp/zimage-new

python3 scripts/repack_boot.py <旧boot.img> /tmp/zimage-new <新boot.img>

fastboot flash boot <新boot.img>
```

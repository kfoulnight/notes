---
title: RK3576 + MAX96755/MAX96752 + 1920×720 LVDS BSP 集成与两周点屏手册
cssclasses:
  - archive-page
---

# RK3576 + MAX96755/MAX96752 + 1920×720 LVDS BSP 集成、点屏与两周验证手册

> [!summary]
> 目标不是“手工 I2C 写几笔寄存器让屏暂时亮”，而是在 **10 个工作日**内交付可合入 BSP 的自动显示链路：
>
> ```text
> Android / DRM atomic commit
>   → RK3576 VOP2 / VP → RK3576 DSI
>   → MAX96755 serializer DRM bridge → GMSL
>   → MAX96752 deserializer DRM panel → OLDI / LVDS
>   → TM103UFKPxx 1920×720 panel
> ```
>
> 最终启动、DRM modeset、SerDes 初始化、panel enable 和背光必须由 **DTB + kernel driver** 自动完成；`i2cset`、sysfs 临时 sequence 或实验脚本仅可用于取证，**绝不能成为产品启动路径**。

---

## 1. 范围、交付物与不可跨越的边界

### 1.1 本轮改造范围

BSP 根目录：

```text
/home/ur/Android/android/kernel-6.1
```

本轮以现有厂商 MFD SerDes 框架为基础，而非新建一套独立 DRM bridge：

```text
drivers/mfd/display-serdes/
├── serdes-core.c          # MFD child、regmap wrapper
├── serdes-i2c.c           # I2C probe、DTS sequence
├── serdes-bridge.c        # MIPI-DSI attach、DRM bridge
├── serdes-panel.c         # drm_panel、timing、backlight
└── maxim/
    ├── maxim-max96755.c   # serializer chip ops
    └── maxim-max96752.c   # deserializer chip ops
```

### 1.2 BSP 实施边界：必须改内核与产品 DTS

这是底层软件适配任务，**必须修改内核驱动和目标产品 DTS/DTSI，并编译、刷写/启动目标 kernel + DTB 到开发板验证**。手册中的 P0–P2 patch stack 就是应当提交的 BSP 改造主线：

```text
serdes-core.c / serdes-i2c.c
  → 修复真实 I2C/regmap 与 sequence 执行
serdes-bridge.c / serdes-panel.c
  → 修复 DRM lifecycle、错误传播和背光时序
maxim-max96755.c / maxim-max96752.c
  → 实现本料号 normal-video、GMSL、OLDI/LVDS、GPIO/PM/IRQ
<target-board>-serdes-max96755-max96752.dtsi + <target-board>.dts
  → 接入实际 I2C、供电/复位、DRM graph、VP/DSI route、panel timing
```

禁止的不是“改内核/DTS”，而是以下不可追溯的做法：

- 不在板端以 `i2cset`、sysfs 或临时脚本替代正式 kernel driver/DTS 实现；
- 不向产品 patch 写入没有同料号来源、readback 或回滚值的未知寄存器；
- 不把 MAX96789/reference board/网上其他项目的寄存器表、GPIO/MFP 编号、时序原样搬给 MAX96755 目标板；
- 不修改 `rk3576-vehicle-serdes-mfd-display-maxim.dtsi` 这个参考板文件来承载产品差异；应新增目标产品 `.dtsi` 并由产品 `.dts` include。

### 1.3 交付定义（Definition of Done）

- [ ] 产品板从冷启动自动加载目标 DTB，MAX96755 与 MAX96752 均 probe 成功；
- [ ] DRM 拓扑形成正确的 `VOP2 → DSI → serdes bridge → serdes panel` 链；
- [ ] 自动完成 MAX96755 DSI RX/GMSL TX、MAX96752 GMSL RX/OLDI-LVDS、panel 电源/reset/backlight 生命周期；
- [ ] 稳定显示 `1920×720@60Hz`，采用已冻结的 panel timing；
- [ ] 黑/白/R/G/B、彩条、棋盘格、灰阶和边界图通过；
- [ ] 不需要人工 `i2cset` 或 sysfs side channel；
- [ ] 10 次冷启动、10 次重启及 enable/disable（以及平台支持时的 suspend/resume）通过；
- [ ] DTS、驱动 patch、寄存器来源、板端日志与风险/阻塞项可追溯。

### 1.3 严禁的错误迁移与假设

1. `rk3576-vehicle-serdes-mfd-display-maxim.dtsi` 的参考链路是 **MAX96789 → MAX96752**。它可用于学习 RK3576 graph、MFD child 组织与 route，但其 MAX96789 sequence、GPIO/MFP 编号、alias 和 115 MHz 1920×720 timing **不能迁移到 MAX96755**。
2. TM103UFKPxx 典型时序为 `2040×735×60 = 89.964 MHz`；不得直接使用参考板 115 MHz timing。
3. 当前资料没有证实已装 MAX96752 有可直接驱动 LVDS 的本地 RGB Pattern generator。不得杜撰 DES Pattern register table。
4. MAX96755 VPG 的 VTX bank、crossbar source、GMSL mode 与 lane mapping 必须来自同料号权威资料、FAE 序列或同拓扑量产代码；不得枚举猜测。

---

## 2. 已确认的硬件契约与待取证项

### 2.1 面板契约：TM103UFKPxx

| 项目 | 当前结论 |
|---|---|
| 分辨率 | 1920×720 |
| 接口/格式 | LVDS_VESA，RGB888，8-bit |
| 有效视频 | DE-only |
| 典型水平参数 | HS=40，HBP=40，HFP=40，HTOTAL=2040 |
| 典型垂直参数 | VS=2，VBP=3，VFP=10，VTOTAL=735 |
| 目标刷新率 | 60 Hz |
| 典型 pixel clock | 89,964,000 Hz |
| 控制极性 | `STBYB=high`、`RSTB=high` 为 normal；`BIST=low` 默认关闭 |
| 典型 rail | VCC1=3.3 V；VSP=6.7 V；VSN=-6.7 V；VGH=18.75 V；VGL=-13 V |

> [!warning]
> 89.964 MHz 接近资料中约 90 MHz 的 LVDS 接收时钟上限。最终 porch、HS/VS/DE 极性、PCLK edge、双 LVDS port、odd/even split、lane/bit mapping 必须同时由面板完整规格书、原理图和实测确认；“画面亮”不能替代这些证据。

### 2.2 Day 1–2 必须关闭的未知项

| 未知项 | 需要的证据 | 未关闭前规则 |
|---|---|---|
| 实际 MAX96755/MAX96752 完整料号与 revision | BOM、丝印、采购资料 | 不混用变体手册 |
| 本板 I2C bus/address/alias/passthrough | 原理图、loaded DTB、读 ID | 不假设 `0x40/0x48` 一定有效 |
| GMSL mode、速率、coax 连接和反向通道 | 原理图/FAE/validated config | 不写 link reset 或 mode 位 |
| DSI host、DC-PHY 与 VP route | 目标 DTS include chain 和 runtime DRM state | 不改 reference board DTS |
| MAX96755 normal-video sequence | 同料号 register guide/FAE/量产代码 | 不拿 MAX96789 table 替代 |
| MAX96752 OLDI/LVDS 及 remote GPIO sequence | 同料号资料与 panel pinout | 不仅按“VESA”猜 lane/port |
| DES local Pattern | 官方资料或完整已验证实现 | 未证实即 `UNSUPPORTED/UNVERIFIED` |

---

## 3. 现有 BSP 架构、数据链路和阻塞缺口

### 3.1 现有可复用链路

```text
DRM framebuffer
  → VOP2 plane / CRTC / selected VP
  → RK3576 DSI host
  → serdes-bridge: 创建并 attach MIPI DSI device
  → MAX96755 chip_data.bridge_ops
  → GMSL coax
  → MAX96752 MFD parent
  → serdes-panel: drm_panel_funcs
  → MAX96752 chip_data.panel_ops
  → OLDI/LVDS → panel
```

控制面：

```text
I2C controller → MAX96755 regmap → GMSL reverse/control channel
                              → MAX96752 regmap
DTS properties → driver private data → regmap write / GPIO / pinctrl / regulator
```

### 3.2 已确认可复用入口

| 文件 | 已有能力 | 本轮如何使用 |
|---|---|---|
| `serdes-core.c` | 按 `serdes_id` 调用 `devm_mfd_add_devices()`；已注册 MAX96755 bridge child 与 MAX96752 panel child | 保持 MFD 分层，先修复 I/O wrapper |
| `serdes-i2c.c` | I2C probe、DTS sequence 解析入口 | 改为受控 sequence executor 或 per-chip 调度 |
| `serdes-bridge.c` | DSI device 创建/attach、DRM bridge 生命周期 | 复用 bridge 框架，去除无依据硬编码并传播失败 |
| `serdes-panel.c` | `drm_panel_init()`、`drm_panel_add()`、`panel-timing` → `drm_display_mode` | 保持 panel 模型，修正生命周期/错误顺序 |
| `maxim-max96755.c` | link detect / `max96755_bridge_link_locked()` 可复用 | 在新的 normal-video enable 路径中显式作为 gate，并实现 init/enable/disable |
| `maxim-max96752.c` | 已定义 video-lock 检查元数据：`0x0003[0]`、`0x0108[6]`，但当前通用调用被禁用 | 恢复实际状态读取后，作为视频接收验收证据；补 panel/输出 lifecycle |
| `Kconfig`/`Makefile` 与 `maxim/Kconfig` | `CONFIG_MFD_SERDES_DISPLAY`、MAX96752/MAX96755 配置入口 | 在产品 defconfig 显式启用并验证 built-in/module 依赖 |

### 3.3 P0 阻塞：当前代码可能“返回成功但未写硬件”

`drivers/mfd/display-serdes/serdes-core.c` 当前以下函数的真实 `regmap` 实现已被注释，函数直接 `return 0`：

```c
serdes_bulk_read()
serdes_bulk_write()
serdes_multi_reg_write()
serdes_set_bits()
```

这意味着任何依赖这些 wrapper 的初始化路径可出现 **软件成功、硬件零变化**。先修复它们，才有资格把 DTS sequence 或 chip ops 当作真实配置。

另外，`serdes-i2c.c` 中 Maxim sequence 的实际下发路径被 `#if 0` 禁用；因此出现 `serdes-init-sequence` 属性也不代表 MAX96755/MAX96752 已经写入寄存器。

`serdes-panel.c` 还有直接影响产品行为的顺序缺口：

- `prepare()` 在调用 chip `init/prepare` 后仍切换 sleep/default pinctrl，且没有在每步失败时立即退出；
- `enable()` 即便 chip `enable` 失败，仍可能执行 DES sequence/开启背光；
- `disable()` 未保证先安全关闭背光再关闭视频；
- 代码中存在临时 `hj` `pr_err/dev_info`，产品 patch 应移除或替换为可控 `dev_dbg()`。

> [!danger]
> 在 P0 I/O wrapper、sequence executor 和 lifecycle 错误传播修复之前，任何“我已经把 DTS sequence 写进去了”的结论都是不可信的。

---

## 4. 建议 patch stack 与依赖顺序

| Patch | 主要文件 | 目的 | 完成条件 |
|---|---|---|---|
| P0-1 | `serdes-core.c` | 恢复真实 regmap bulk/multi/update-bits，统一 `io_lock` 与错误传播 | 写失败返回负 errno；读写/掩码更新可被回读验证 |
| P0-2 | `serdes-i2c.c`、必要时 `core.h` | 把 DTS sequence 变成有长度校验、明确 delay 单位、有限 retry、失败即中止的 executor；隔离无关 `himax_common_init()` 副作用 | 每笔可审计；任一失败能停止并打印 device/reg/index |
| P0-3 | `serdes-bridge.c`、`serdes-panel.c` | 修复 prepare/enable/disable/unprepare 的顺序与 return code；背光最后打开 | 失败不亮背光、不伪成功；正常关闭无残留输出 |
| P1-1 | 产品 defconfig、`Kconfig`/`Makefile`（仅缺失时） | 使 display-serdes、MAX96755、MAX96752 参与目标 kernel 构建 | `/proc/config.gz` 或 config 显示选项正确 |
| P1-2 | 新建产品 SerDes `.dtsi` + 产品 `.dts` | 正确 I2C、power/reset、MFD child、DRM graph、timing、backlight | 反编译 DTB graph 与 source 一致 |
| P2-1 | `maxim-max96755.c` | 同料号证据支持的 DSI RX/video pipe/GMSL TX、lock timeout、disable | DSI 输入后 serializer 侧视频状态与 link lock 可读 |
| P2-2 | `maxim-max96752.c` | GMSL RX/video decoder/OLDI-LVDS、remote GPIO、panel power 输出顺序 | VID_LOCK 与图像同时稳定 |
| P3 | 可选 debug-only 文件/属性 | MAX96755 VPG；normal-video 互斥、完整回滚 | 仅在权威 VTX/crossbar sequence 完整时实现 |

建议 commit 顺序：`P0-1 → P0-2 → P0-3 → P1-1 → P1-2 → P2-1 → P2-2 → P3(optional)`。每个 commit 单独可编译，保持可 bisect 和可回退。

---

## 5. Phase 0 / Gate BSP-0：目标产品、DTB 与构建基线（Day 1）

### 5.1 必须先回答的问题

```text
运行中的 DTB 是哪个文件生成的？
产品 DTS 的 include chain 是什么？
实际 SerDes 在哪个 I2C controller/bus/address？
哪个 VP route 到哪一个 DSI host / DC-PHY？
MAX96752 通过何种 alias/passthrough 被 Linux 访问？
目标 defconfig 与实际 booted kernel config 是什么？
```

**在这些问题没有答案前，不允许改 `rk3576-vehicle-serdes-mfd-display-maxim.dtsi`。** 新建目标板专属 `.dtsi`，由实际产品 `.dts` include，隔离参考板差异。

### 5.2 开发机审计命令

```sh
export KERNEL_SRC=/home/ur/Android/android/kernel-6.1
export DTS_ROOT="$KERNEL_SRC/arch/arm64/boot/dts/rockchip"

rg -n 'max967(52|55)|serdes|mipi-dsi|dsi-host|remote-endpoint|route' \
  "$DTS_ROOT" "$KERNEL_SRC/drivers/mfd/display-serdes"
rg -n 'MFD_SERDES_DISPLAY|MAX967(52|55)' \
  "$KERNEL_SRC/drivers/mfd/display-serdes" "$KERNEL_SRC/arch/arm64/configs"
```

板端基线（保存到 `<LOGDIR>`）：

```sh
mkdir -p <LOGDIR>
uname -a | tee <LOGDIR>/uname.txt
cat /proc/cmdline | tee <LOGDIR>/cmdline.txt
dmesg -T > <LOGDIR>/dmesg-boot.txt
cat /sys/class/drm/card0/device/uevent > <LOGDIR>/card0-uevent.txt
ls -l /sys/class/drm > <LOGDIR>/drm-class.txt
cat /sys/kernel/debug/dri/0/state > <LOGDIR>/drm-state.txt 2>/dev/null || true
```

`card0-DP-1/status=disconnected` 与 `dw-dp ... aux ... -110` 是 native DP 链路观测，**不是** DSI→GMSL→LVDS 面板成功/失败的判断依据。

### 5.3 BSP-0 通过标准

- [ ] 已记录 source `.dts`、产品 `.dtsi`、最终 `.dtb`、defconfig 与 build command；
- [ ] 已从运行时确认 DRM master 是 `rockchip-drm` / `display-subsystem`；
- [ ] 已定位目标 DSI/VP/route 与 SerDes I2C controller；
- [ ] UART、已知可启动镜像和回退步骤可用；
- [ ] 已建立按日期/commit 编号存档的日志目录。

---

## 6. Phase 1：修复基础 I/O 与生命周期（Day 2–3）

### 6.1 `serdes-core.c` 的实现要求

恢复并规范以下行为：

```text
serdes_bulk_read()      → regmap_bulk_read()
serdes_bulk_write()     → regmap_bulk_write() 或按 regmap 格式受锁逐项写入
serdes_multi_reg_write()→ regmap_multi_reg_write()
serdes_set_bits()       → regmap_update_bits()
```

规则：

- 写路径在 `serdes->io_lock` 保护下执行，读路径按 regmap/调用上下文决定是否共用该锁；
- 检查 `count <= 0`、空指针及 device/regmap 有效性；
- 返回底层负 errno，绝不把失败转换成 `0`；
- debug log 在成功/失败时携带 chip、register、value、errno；
- 不在 hard IRQ 中调用可能 sleep 的 regmap/I2C 操作。

### 6.2 `serdes-i2c.c` 的 sequence executor 要求

`serdes-init-sequence` 只能是一个**受控配置数据源**，不是绕过 chip lifecycle 的万能入口：

1. probe 时严格校验 property 格式、entry 数与 register/value/delay 表达；
2. 明确 delay 单位（推荐 DTS 字段或统一记录为 us/ms）；
3. 每笔写调用修复后的 wrapper；失败即停止，打印 sequence index 和 register；
4. 对瞬态 I2C NACK 可有限 retry（例如 3 次，退避），禁止无限循环；
5. 仅执行来源可追溯的同料号表；
6. 删除或隔离与 MAX96755/MAX96752 无关的 `himax_common_init()` 通用副作用；
7. 正常视频配置优先放进 `max96755_*`、`max96752_*` chip ops；DTS sequence 只承载板级差异或经审查的 table。

### 6.3 通用 DRM lifecycle 修正

目标顺序：

```text
panel.prepare:
  supply/pinctrl(default) → chip init → chip prepare → 返回成功

bridge/panel enable:
  确认 link/video 状态 → 打开 serializer/deserializer video output
  → panel ready → 最后 backlight_enable

disable:
  先 backlight_disable → 停止 panel/OLDI output → 停止 upstream video

unprepare:
  chip unprepare → pinctrl(sleep) / regulator off
```

每一步失败立即向 DRM core 返回错误；不得在失败路径执行后续 `serdes_i2c_set_sequence()` 或 `backlight_enable()`。

---

## 7. Phase 2：产品 DTS / DTB 设计（Day 4–5）

### 7.1 DTS 组织原则

```text
<target-board>.dts
  #include "<target-board>-serdes-max96755-max96752.dtsi"

<target-board>-serdes-max96755-max96752.dtsi
  ├── selected DSI / DC-PHY / VP route enable
  ├── MAX96755 I2C parent and MFD children
  ├── MAX96752 reachable I2C parent and MFD children
  ├── DSI ↔ bridge ↔ panel graph
  ├── panel timing / physical size / backlight
  └── target-only pinctrl, GPIO, regulators
```

实际 node label、GPIO、supply、I2C 地址必须来自目标板原理图和 BSP-0；以下为结构模板，不是可复制的地址或 pin：

```dts
&chosen_dsi_host {
    status = "okay";

    ports {
        port@1 {
            reg = <1>;
            dsi_out_ser: endpoint {
                remote-endpoint = <&max96755_dsi_in>;
            };
        };
    };
};

&i2cX {
    max96755: serializer@<actual-address> {
        compatible = "maxim,max96755";
        reg = <...>;
        /* real supplies/reset/lock/error/sel-mipi only */

        max96755_bridge: bridge {
            compatible = "maxim,max96755-bridge";
            ports {
                port@0 { max96755_dsi_in: endpoint { remote-endpoint = <&dsi_out_ser>; }; };
                port@1 { max96755_gmsl_out: endpoint { remote-endpoint = <&max96752_panel_in>; }; };
            };
        };
    };
};

/* Declare MAX96752 only after remote I2C alias/passthrough is proven. */
max96752: deserializer@<actual-address> {
    compatible = "maxim,max96752";
    reg = <...>;

    max96752_panel: panel {
        compatible = "maxim,max96752-panel";
        backlight = <&panel_backlight>;
        panel-size = <actual_width_mm actual_height_mm>;
        panel-timing {
            clock-frequency = <89964000>;
            hactive = <1920>; hfront-porch = <40>;
            hsync-len = <40>; hback-porch = <40>;
            vactive = <720>; vfront-porch = <10>;
            vsync-len = <2>; vback-porch = <3>;
            /* polarity values require final source confirmation */
        };
        port { max96752_panel_in: endpoint { remote-endpoint = <&max96755_gmsl_out>; }; };
    };
};
```

### 7.2 DTS 核验项目

- [ ] DSI `port@1` 与 MAX96755 bridge input endpoint 成对；
- [ ] MAX96755 bridge output 与 MAX96752 panel endpoint 成对；
- [ ] 不存在一个 endpoint 被两个 output 占用的 graph；
- [ ] 只启用一个正确的 DSI-in-VP route；
- [ ] DSI lane count/format 由 MAX96755 DSI RX 能力和原理图决定，不沿用 generic bridge 默认值；
- [ ] `panel-size` 使用真实物理尺寸，非 `serdes-panel.c` 默认 `320×180`；
- [ ] `panel-timing` 反编译后的 DTB 与 source 一致；
- [ ] MAX96752 的远程 I2C 可访问模型经 alias/passthrough 读 ID 证实。

### 7.3 构建与 DTB 验收

使用项目既有构建入口；至少执行等价动作：

```sh
make O=<out> <target_defconfig>
make O=<out> ARCH=arm64 -j"$(nproc)" \
  Image modules dtbs

dtc -I dtb -O dts \
  -o <LOGDIR>/loaded-or-built.dts <target>.dtb
```

验收：kernel 与 DTB 同一 build/commit 产出；产品刷写分区/bootloader DTB 加载规则已记录；运行时 `/proc/cmdline`、boot log 与目标 node `compatible` 可对应。

---

## 8. Phase 3：MAX96755 serializer normal-video 生命周期（Day 5–6）

修改文件：`drivers/mfd/display-serdes/maxim/maxim-max96755.c`。

### 8.1 实现目标

| 回调 | 应做的事 |
|---|---|
| `max96755_bridge_init()` | 读 ID/revision，采集初始 status，按权威 sequence 配置 DSI RX、video pipe、GMSL TX；失败停止 |
| `max96755_bridge_enable()` | 确认 DSI 输入/时钟条件和 `max96755_bridge_link_locked()`；按资料打开最终 video path |
| `max96755_bridge_disable()` | 先安全关 TX/video output，保留可诊断状态；不将未知 reset 当作常规关闭 |
| PM/IRQ | LOCK/ERR GPIO 若使用：硬 IRQ 只记事件/调度 work；I2C status 读取和恢复在 threaded IRQ/workqueue 完成 |

### 8.2 必须接入的 gate

现有 `max96755_bridge_link_locked()` 目前用于 bridge attach/detect 的连接状态判断，`max96755_bridge_enable()` 本身尚未调用它。因此 P2-1 必须把它显式接入 normal-video enable 前的基础 gate。它只能证明 GMSL/CMU link 状态；还需要独立证据证明 DSI video 输入、video pipe 和 MAX96752 VID_LOCK。

建议日志格式：

```text
max96755: id=<...> rev=<...> link=<0/1> cmu=<0/1> err=<...>
max96755: dsi_lanes=<...> format=<...> route=<normal-video> enable=<0/1>
```

不得把 MAX96755 VPG 配置混入 normal-video init。VPG 属于 debug-only、显式切换、与 normal source 互斥的后续 patch。

---

## 9. Phase 4：MAX96752 / panel / LVDS 生命周期（Day 6–8）

修改文件：`drivers/mfd/display-serdes/maxim/maxim-max96752.c`，必要时配合 `serdes-panel.c` 与 pinctrl child。

### 9.1 实现目标

| 回调 | 应做的事 |
|---|---|
| `max96752_panel_prepare()` | 按真实 reset/power/pinctrl 顺序，配置 GMSL RX、video decoder、OLDI/LVDS formatter；等待规定稳定时间 |
| `max96752_panel_enable()` | 仅当 link/video 状态满足要求时启用 LVDS 输出，随后由 generic panel 开背光 |
| `max96752_panel_disable()` | 停止输出前后与 panel/backlight 顺序一致，避免屏亮但无有效视频 |
| `max96752_panel_unprepare()` | 关闭输出、恢复 pinctrl/电源，返回真实错误 |
| GPIO ops | 实现实际使用的 remote GPIO callback；现有 pinctrl 路径虽会调用 `serdes_set_bits()`，也要随 P0-1 恢复真实硬件写入；不得保留“返回 0 而没有硬件写入”的 GPIO stub |
| PM/IRQ | 记录/恢复必要状态；避免 hard IRQ I2C |

### 9.2 MAX96752 视频证据

现有 BSP 已定义以下 video-lock 检查元数据，但当前通用调用位于禁用代码路径，不能把它表述为“现有运行时已观测”。P0/P2 恢复真实读寄存器与调用路径后，可将其作为最小运行时证据：

```text
0x0003 bit 0
0x0108 bit 6   # VIDEO_RX8 / VID_LOCK evidence
```

验收不是单独读到 bit=1，而是：

```text
MAX96755 link stable
+ MAX96752 VID_LOCK stable
+ DRM active 1920×720 mode
+ panel image correct
+ CRC/decoder/PRBS errors do not continuously increase
```

### 9.3 LVDS 映射检查顺序

1. 先用已确认 timing 与 VESA/RGB888 输出得到稳定画面；
2. R/G/B 单色确认 RGB order/bit mapping；
3. 棋盘格确认 dual-port odd/even、lane 以及像素交织；
4. 灰阶确认 bit swap、bit-depth 与 banding；
5. 边界/坐标确认 DE 起点、porch、镜像/翻转；
6. 每次只改一个 mapping 维度，保留 old/new、寄存器来源、图像和 readback。

---

## 10. 三层 Pattern 的 BSP 定位

| 层级 | 覆盖路径 | 实现决策 |
|---|---|---|
| DES Pattern | MAX96752 → LVDS → panel | 当前 `UNSUPPORTED/UNVERIFIED`；仅在 exact part 的官方序列或量产实现存在时增加独立 debug patch |
| SER Pattern | MAX96755 VPG → GMSL → MAX96752 → panel | MAX96755 手册确认 VTG/VPG/checkerboard/gradient 能力，但实际 VTX bank/crossbar 未确认；仅作为 P3 debug-only |
| SoC Pattern | RK3576 framebuffer → DSI → MAX96755 → GMSL → MAX96752 → panel | P0 主交付路径，必须优先完成 |

### 10.1 DES Pattern 的正确结论

如果 Day 2 未获得精确 MAX96752 part 的 local Pattern 文档，最终报告必须写：

```text
MAX96752 local RGB Pattern: UNSUPPORTED / UNVERIFIED for current part.
DES-to-panel stage is instead validated using a known-good upstream RGB source.
```

这不是遗漏，而是防止把不属于芯片的功能写进产品 BSP。

### 10.2 SER VPG 的硬门禁

只有以下全部具备时才做：已确认 VTX bank、crossbar source、source route、link mode、完整可回滚 sequence，且 RK3576 normal-video 已经点亮或拥有同等可靠输出路径。可选择 checkerboard/gradient，不得把未文档化 mode 当 colorbar。

---

## 11. 板端调试步骤与验收标准

### 11.1 Gate A：面板电源/控制

| 项目 | 验收 |
|---|---|
| VCC1/VSP/VSN/VGH/VGL | 规格范围、无明显跌落/异常纹波 |
| STBYB/RSTB/BIST | 极性与时序由 scope 证实 |
| Backlight | 可控，但必须在 video path ready 后最后开启 |
| Fail_DET/温升/电流 | 无异常（若板上可观测） |

### 11.2 Gate B：I2C 与 GMSL 控制面

先读、后写；只读取已确认的 bus/address/register：

```sh
i2cdetect -l
i2cdetect -y <SERDES_I2C_BUS>
# actual register width/options must match verified map
i2cget -y <SERDES_I2C_BUS> <MAX96755_ADDR> 0x0d
i2cget -y <SERDES_I2C_BUS> <MAX96755_ADDR> 0x13
```

通过：MAX96755 ID 与料号匹配；MAX96752 经设计规定 alias/passthrough 可稳定读；link/CMU status 连续 10 分钟稳定；断接 coax 后状态与错误计数可解释。

### 11.3 Gate C：DRM / video lock / image

```sh
ls -l /sys/class/drm
cat /sys/kernel/debug/dri/0/state 2>/dev/null
modetest -M rockchip -c
modetest -M rockchip -p
```

依次显示：black → white → red → green → blue → RGB bars → checkerboard → gradient → border。

通过：预期 CRTC/connector enabled；mode=1920×720@60 且 timing 一致；MAX96752 video lock 稳定；无滚动、周期黑屏、错色、隔行/隔列错位；错误计数不持续增长。

### 11.4 症状到责任域

| 症状 | 优先证据 |
|---|---|
| 本地 MAX96755 不 ACK | supply/reset、I2C pinmux、strap、DTS reg |
| SER ACK、DES 不可达 | reverse channel、link lock、alias/passthrough |
| link 不 lock | 两端 power/reset、mode、coax/connector/EMI |
| DRM active 但 DES 无 VID_LOCK | DSI→MAX96755 input、serializer video route |
| VID_LOCK 有、画面黑 | MAX96752 LVDS enable、DE、panel reset/backlight |
| 红蓝互换/偏色 | VESA/RGB order、bit/lane mapping |
| 隔列隔行错位 | dual-port odd/even split、LVDS lane mapping |
| 滚动/闪烁 | PCLK/porch/DE edge、link stability、电源完整性 |

---

## 12. 两周排期与停止点

| Day | 主线工作 | 当天交付物 | 失败时动作 |
|---:|---|---|---|
| 1 | BSP-0：产品 DTS/DTB/config/route/I2C 审计 | 责任表、loaded-DTB 证据、基线日志 | 补齐 source/DTB/恢复链，禁止改 reference |
| 2 | 确认料号、原理图、normal-video table、DES Pattern 能力 | 证据索引、未知项关闭表 | 无 DES Pattern 证据即标 UNVERIFIED |
| 3 | P0-1/P0-2：恢复 I/O 与 sequence 执行 | 可编译 patch、read/write/readback 日志 | 写失败必须可见，禁止伪成功 |
| 4 | P0-3 + P1-1：lifecycle/config | probe/enable 失败路径日志 | 修错误传播，禁止失败后背光打开 |
| 5 | P1-2：产品 DTS graph/timing/DTB | 反编译 DTB、probe graph | graph/route 未闭合不进入视频 |
| 6 | P2-1：MAX96755 normal video | SER ID/link/DSI-video 证据 | 未 lock 升级硬件/FAE |
| 7 | P2-2：MAX96752 LVDS、RK3576 首图 | DRM state、VID_LOCK、首图 | 先查 DSI route/VID_LOCK，再调 mapping |
| 8 | 颜色/棋盘格/灰阶 + power/GPIO | 图像矩阵、scope/readback | 每次只变一个维度 |
| 9 | IRQ/PM/enable-disable 与 optional VPG | 回归日志；或 VPG FAE 包 | sequence 不全则不猜 VPG |
| 10 | 冷启动/重启/挂起 smoke、patch/log 交付 | 放行表、风险清单、commit 清单 | 启动延长 soak/缺陷单 |

硬停止点：Day 4 未稳定 link lock，转硬件/FAE；Day 5 graph 未闭合，停止调寄存器；Day 7 无 VID_LOCK，优先 DSI/video route；Day 9 无 VPG 权威配置，不实现 VPG。

---

## 13. 回归矩阵与 BSP 放行表

| 测试 | 目标 | 结果 |
|---|---:|---|
| 冷启动自动点亮 | ≥10 次 |  |
| 热重启自动点亮 | ≥10 次 |  |
| enable/disable | ≥10 次 |  |
| suspend/resume（支持时） | ≥10 次 |  |
| R/G/B/W/K | 每种 ≥5 分钟 |  |
| bars/checkerboard/gradient | 每种 ≥10 分钟 |  |
| 静态/动态视频 | ≥2 小时 |  |
| I2C status 连续读取 | 1000 次或等效驱动计数 |  |
| link/VID lock | 全程不掉或掉失可恢复并留日志 |  |

最终放行：

```text
[ ] target DTB/load chain confirmed
[ ] P0 wrappers and sequence path perform real writes and report failures
[ ] DRM graph/probe order complete
[ ] normal video is automatic after cold boot
[ ] final timing / LVDS mapping source recorded
[ ] image matrix passed
[ ] error counters and dmesg reviewed
[ ] patches are buildable, ordered, revertible
[ ] known gaps/FAE dependencies explicitly documented
```

---

## 14. 寄存器与每日日志模板

### 14.1 Register write ledger

| 时间 | Commit | 设备/料号 | I2C bus/addr | Reg | Mask | Old | New | Readback | 目的 | 来源（手册版本/页/量产代码） | 回滚值 | 结果 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
|  |  |  |  |  |  |  |  |  |  |  |  |  |

无来源、无 readback 或无回滚值的 write 不得合入产品 sequence。

### 14.2 Daily log

```markdown
# YYYY-MM-DD — RK3576/MAX96755/MAX96752 BSP 点屏日志

## 软件基线
- Board / Panel / cable:
- Kernel commit / defconfig:
- Source DTS / built DTB / flash partition:
- Driver configuration:

## 今日 patch 与目标
- Commit:
- [ ]

## 关键状态
- DRM CRTC/connector/mode:
- MAX96755 ID / LINK / CMU / error:
- MAX96752 LINK / VID_LOCK / decoder-CRC errors:
- Panel power/reset/backlight:

## 变更与证据
| Device | Reg | Old | New | Readback | Source | Result |
|---|---|---|---|---|---|---|

## 附件
- dmesg:
- DRM state:
- DTB decompile:
- scope/photo/video:

## 结论
- PASS / FAIL / BLOCKED:
- Fault domain narrowed to:
- Next action:
```

### 14.3 FAE 升级包最小内容

```text
board revision; exact MAX96755/MAX96752 marking; panel/cable details;
measured rails and reset waveform; built DTB and graph excerpt;
boot dmesg; DRM state; verified register map revision;
ID/link/video-lock/error-counter snapshots; reproduction rate;
request for exact normal-video sequence / VTX bank-crossbar / LVDS mapping.
```

---

## 15. 最终状态表述

不要用“屏亮了，所以全链路正常”。最终报告按下列粒度给结论：

```text
BSP-0 source/DTB/config baseline: PASS / FAIL / BLOCKED
P0 real regmap/sequence execution: PASS / FAIL / BLOCKED
Panel power/reset/backlight: PASS / FAIL / BLOCKED
MAX96755 local I2C + GMSL link: PASS / FAIL / BLOCKED
MAX96752 remote I2C + video lock: PASS / FAIL / BLOCKED
MAX96752 LVDS output → panel: PASS / FAIL / BLOCKED
RK3576 → panel end-to-end normal video: PASS / FAIL / BLOCKED
MAX96755 VPG diagnostic path: PASS / FAIL / BLOCKED / NOT APPLICABLE
MAX96752 local RGB Pattern: PASS / FAIL / UNSUPPORTED-UNVERIFIED
Cold-boot / restart / PM regression: PASS / FAIL / PARTIAL
```

> [!summary]
> BSP 主线的优先级是：**真实 I/O → 正确 DTS graph → normal-video chip lifecycle → RK3576 首图 → 映射/回归**。Pattern 是隔离工具，不是允许猜寄存器的理由；任何未由同料号资料闭环的 sequence 都应以风险或 FAE 阻塞项交付，而不是写入产品内核。



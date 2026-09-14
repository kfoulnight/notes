---
title: MAX96755 → MAX96752 显示时序与时钟流程
cssclasses:
  - archive-page
---

# MAX96755 → MAX96752 显示时序与时钟流程

本文梳理 RK3576 通过 MAX96755 Serializer 与 MAX96752 Deserializer 驱动 LVDS 面板时，显示时序、DRM mode、DSI/PHY 时钟以及 SerDes 内部 PLL 配置之间的关系。

> [!summary] 核心主线
> `panel-timing` 负责生成 DRM 显示模式，Rockchip DRM/DSI 根据该模式配置输入侧时钟；MAX96755/MAX96752 的链路、格式、PLL 和 LVDS 输出则主要依赖 `serdes-init-sequence` 中的寄存器配置。

## 一、硬件与软件拓扑

### 硬件数据链路

1. RK3576 VOP 产生显示帧。
2. Rockchip DSI / MIPI D-PHY 输出 MIPI DSI 数据。
3. MAX96755 Serializer 接收并串行化 DSI 数据。
4. GMSL/SerDes 链路传输数据。
5. MAX96752 Deserializer 解串并输出 LVDS / panel 信号。

本工程使用 Linux DRM + `display-serdes` MFD 框架。MAX96755/MAX96752 通过 I2C 枚举，随后分别创建 bridge、panel、GPIO/pinctrl 等 MFD 子设备。

### 主要代码目录

> [!info] 驱动文件速查
> | 文件 | 主要作用 |
> |---|---|
> | `drivers/mfd/display-serdes/serdes-i2c.c` | SerDes I2C 主驱动与初始化序列 |
> | `drivers/mfd/display-serdes/serdes-core.c` | 创建 MFD 子设备 |
> | `drivers/mfd/display-serdes/serdes-panel.c` | 通用 DRM panel 与 panel-timing 解析 |
> | `drivers/mfd/display-serdes/serdes-bridge.c` | 通用 DRM bridge |
> | `drivers/mfd/display-serdes/maxim/maxim-max96755.c` | MAX96755 芯片操作 |
> | `drivers/mfd/display-serdes/maxim/maxim-max96752.c` | MAX96752 芯片操作 |

## 二、设备树中的三类配置

### 面板显示时序

当前示例位于 `arch/arm64/boot/dts/rockchip/hzy-96755-96752.dtsi:85-99`：

```dts
panel-timing {
    clock-frequency = <89964000>;
    hactive = <1920>;
    vactive = <720>;
    hfront-porch = <40>;
    hsync-len = <40>;
    hback-porch = <40>;
    vfront-porch = <10>;
    vsync-len = <2>;
    vback-porch = <3>;
    hsync-active = <0>;
    vsync-active = <0>;
    de-active = <0>;
    pixelclk-active = <0>;
};
```

> [!info] `panel-timing` 属性
> | 属性 | 含义 |
> |---|---|
> | `clock-frequency` | 面板像素时钟，单位为 Hz |
> | `hactive` / `vactive` | 有效分辨率 |
> | `hfront-porch` / `hsync-len` / `hback-porch` | 水平前肩、同步宽度、后肩 |
> | `vfront-porch` / `vsync-len` / `vback-porch` | 垂直前肩、同步宽度、后肩 |
> | `hsync-active` / `vsync-active` | 同步信号极性 |
> | `de-active` | DE 极性 |
> | `pixelclk-active` | 像素时钟采样边沿 |

### SerDes 芯片寄存器初始化

MAX96755 与 MAX96752 均可通过设备树配置：

```dts
serdes-init-sequence = [
    ...
];
```

序列的基本格式为“寄存器地址 + 寄存器值”：

```text
寄存器地址 寄存器值
```

例如：

```dts
020f 0090
ffff 000a
01ce 0040
```

其中 `ffff value` 表示延时 `value` 毫秒，不表示普通寄存器写入。

初始化序列通常负责以下内容：

- DSI 输入路径
- SerDes 链路
- 视频流路由
- 输出格式与 lane mapping
- LVDS 输出
- 芯片内部 PLL、分频与时钟使能
- 芯片复位、锁定与工作模式

> [!warning] 时钟配置边界
> 当前驱动不会根据 `panel-timing.clock-frequency` 自动生成完整的 MAX96752 PLL 寄存器序列。芯片内部时钟寄存器主要依赖 `serdes-init-sequence`，因此该序列必须与实际显示时序匹配。

### DSI / VOP 时钟

Rockchip 侧 DSI、MIPI D-PHY、VOP 的时钟通常由 Rockchip DRM/DSI 驱动根据 DRM mode 动态计算和配置，而不是由 MAX96755 驱动完成。

## 三、Probe 与初始化流程

### MAX96755/MAX96752 I2C probe

设备树中的 `compatible` 匹配后，`serdes-i2c.c` 大致按以下顺序执行：

1. 进入 I2C probe。
2. 创建 `struct serdes` 与 regmap。
3. 读取并匹配芯片 ID。
4. 匹配 `serdes_chip_data`。
5. 解析 GPIO、供电、reset、lock 等资源。
6. 解析 `serdes-init-sequence`。
7. 执行 `serdes_device_init()`。
8. 创建 MFD 子设备。

MAX96755 的芯片数据定义在 `drivers/mfd/display-serdes/maxim/maxim-max96755.c:723-736`：

```c
struct serdes_chip_data serdes_max96755_data = {
    .name = "max96755",
    .serdes_type = TYPE_SER,
    .bridge_ops = &max96755_bridge_ops,
    ...
};
```

MAX96755 是 `TYPE_SER`，即 Serializer；MAX96752 是 Deserializer。

### 初始化序列执行

通用初始化路径如下：

1. `serdes_get_init_seq()` 获取序列。
2. `serdes_parse_init_seq()` 解析序列。
3. `serdes_device_init()` 执行设备初始化。
4. `serdes_i2c_set_sequence()` 按顺序写入。
5. `serdes_reg_write(reg, value)` 完成单次寄存器写入。

普通芯片的序列执行在 `serdes-i2c.c` 中逐项写寄存器，并可读回校验。

MAX96752 存在专用初始化分支：

```c
if (serdes->chip_data->serdes_id == MAXIM_ID_MAX96752)
    serdes_maxim96752_init(serdes);
```

因此分析 MAX96752 时，需要同时检查 `maxim-max96752.c` 的专用初始化函数，以及 DTS 中的 `serdes-init-sequence`。

## 四、`panel-timing` 转换为 DRM mode

MAX96752 的 panel 子节点由 `serdes-panel.c` 匹配，调用路径为：

1. `serdes_panel_probe()`
2. `serdes_panel_parse_dt()`
3. `of_get_display_timing()`
4. `videomode_from_timing()`
5. `drm_display_mode_from_videomode()`
6. 保存到 `serdes_panel->mode`
7. `drm_panel_add()`

核心代码：

```c
struct display_timing dt;
struct videomode vm;

of_get_display_timing(dev->of_node, "panel-timing", &dt);
videomode_from_timing(&dt, &vm);
drm_display_mode_from_videomode(&vm, &serdes_panel->mode);
```

转换后的关键成员包括：

```c
serdes_panel->mode.clock
serdes_panel->mode.hdisplay
serdes_panel->mode.hsync_start
serdes_panel->mode.hsync_end
serdes_panel->mode.htotal
serdes_panel->mode.vdisplay
serdes_panel->mode.vsync_start
serdes_panel->mode.vsync_end
serdes_panel->mode.vtotal
serdes_panel->mode.flags
```

> [!note] 单位换算
> DTS 中 `clock-frequency` 的单位是 Hz，而 DRM `mode.clock` 的单位是 kHz。因此 `89964000 Hz` 最终约为 `89964 kHz`。

## 五、DRM 获取显示模式

DRM 调用 panel 的 `get_modes()`，流程如下：

1. `serdes_panel_get_modes()` 复制 `serdes_panel->mode`。
2. 设置 `PREFERRED` / `DRIVER` 类型。
3. 调用 `drm_mode_probed_add()`。
4. 将 mode 提供给 DRM connector。

实现位置：`drivers/mfd/display-serdes/serdes-panel.c:91-128`。

驱动会打印实际使用的 mode，例如：

```text
mode clock 89964 kHz
H: 1920 1960 2000 2040
V: 720 730 732 735
```

当前示例的计算结果：

- `HTOTAL = 1920 + 40 + 40 + 40 = 2040`
- `VTOTAL = 720 + 10 + 2 + 3 = 735`
- 刷新率约为 `89,964,000 / (2040 × 735) ≈ 60 Hz`

## 六、DRM bridge 建链

MAX96755 bridge 子节点由 `serdes-bridge.c` 匹配，主要流程为：

1. `serdes_bridge_probe()` 获取父设备 `serdes/regmap`。
2. 解析 device-tree graph endpoint。
3. 设置 bridge 类型。
4. 调用 `drm_bridge_add()`。
5. 如果存在 `sel-mipi`，连接 DSI。

当前 DTS 设置了：

```dts
sel-mipi;
```

因此 bridge 类型会设置为 DSI，并调用 `serdes_attach_dsi()`。

逻辑上的 DRM bridge 链为：

`Rockchip VOP` → `Rockchip DSI` → `MAX96755 DRM bridge` → `MAX96752` → `Panel`

## 七、点屏阶段的回调顺序

整体顺序大致如下：

1. `get_modes()` 获取显示模式。
2. 执行 bridge attach。
3. 执行 `panel prepare`。
4. 执行 bridge `pre-enable/enable`。
5. Rockchip DSI 开始输出视频。
6. 执行 `panel enable`。
7. 打开背光。

### `panel_prepare`

`serdes_panel_prepare()` 会依次：

1. 调用芯片 panel `init`。
2. 调用芯片 panel `prepare`。
3. 设置 pinctrl sleep。
4. 设置 pinctrl default。

当前 MAX96752 的 `panel_prepare()` 返回 0，暂未根据 `mode.clock` 动态配置 PLL。

### `panel_enable`

`serdes_panel_enable()` 会依次：

1. 调用芯片 panel `enable`。
2. 调用 `remdev_panel_enable()`。
3. 对 Deserializer 再执行 `serdes_i2c_set_sequence()`。
4. 打开背光。

相关代码位于 `drivers/mfd/display-serdes/serdes-panel.c:53-72`。

### MAX96755 bridge 回调

回调定义位于 `drivers/mfd/display-serdes/maxim/maxim-max96755.c:437-443`。

| 回调 | 当前实现 |
|---|---|
| `init()` | 空操作 |
| `attach()` | 检查链路 lock 状态 |
| `detect()` | 检查链路状态与热插拔状态 |
| `enable()` | 空操作 |
| `disable()` | 空操作 |

链路锁定检查位于 `drivers/mfd/display-serdes/maxim/maxim-max96755.c:344-374`，优先读取 `lock-gpios`；没有 GPIO 时读取寄存器 `0x0013`，并检查 `LOCKED` 位。

## 八、时钟在整条链路中的流向

1. DTS 的 `panel-timing.clock-frequency` 描述目标面板像素时钟。
2. `serdes-panel.c` 解析该时序并生成 `struct drm_display_mode.clock`。
3. Rockchip DRM/DSI 根据 mode 配置 VOP、MIPI DSI、lane rate 与 PHY PLL。
4. MAX96755 接收 DSI 数据并进行串行化。
5. SerDes 链路传输数据。
6. MAX96752 解串，并使用内部输出 PLL/LVDS clock 输出到面板。
7. 面板接收数据与时钟。

> [!abstract] 两条配置路径
> **显示时序路径**：`panel-timing` → DRM display mode → Rockchip DSI/PHY 时钟
>
> **SerDes 寄存器路径**：`serdes-init-sequence` → MAX96755/MAX96752 内部链路、格式、PLL 与输出配置

## 九、各模块职责速查

> [!info] 模块职责与动态配置范围
> | 模块 | 主要职责 | 是否根据 panel-timing 动态配置 |
> |---|---|---|
> | DTS panel-timing| 分辨率、时序、像素时钟、极性 | 配置来源 |
> | serdes-panel.c | 解析时序并生成 DRM mode | 只负责转换，不配置芯片 PLL |
> | Rockchip VOP/DSI | 配置 VOP、DSI、D-PHY 与输入时钟 | 通常是 |
> | MAX96755 驱动 | Serializer、bridge、锁定检测、GPIO/pinctrl | 当前基本不是 |
> | MAX96752 驱动 | Deserializer、panel、LVDS 输出 | 主要依赖初始化寄存器 |
> | serdes-init-sequence | 直接写入 SerDes 寄存器 | 预先配置 |
> | MAX96755_I2C | MFP 引脚功能复用 | 与显示时序无关 |

## 十、调试建议

### 检查面板时序是否生效

查看 `serdes-panel.c` 打印的以下信息：

```text
mode clock
H: hdisplay hsync_start hsync_end htotal
V: vdisplay vsync_start vsync_end vtotal
```

重点确认 `mode.clock`、水平总计数 `htotal` 和垂直总计数 `vtotal` 是否与 DTS 计算结果一致。

### 检查 Rockchip DSI

重点查看：

- DSI pixel clock
- DSI lane rate
- D-PHY PLL
- lane 数量
- video mode
- DSI 错误中断

### 检查 MAX96755

重点查看：

- `lock-gpios`
- 寄存器 `0x0013`
- `LOCKED` 位
- DSI 输入是否存在时钟和数据

### 检查 MAX96752

重点查看：

- `serdes-init-sequence`
- 输入 / 输出格式
- LVDS clock 使能
- PLL lock 状态
- 输出 pixel clock
- link 配置与 lane mapping

> [!danger] 排查顺序
> 如果面板无显示，不要只修改 `panel-timing`。应先确认 DRM 实际生成的 mode，再确认 Rockchip DSI 是否输出正确，最后核对 MAX96755 链路锁定状态以及 MAX96752 的 PLL、LVDS 和输出配置。

## 最后记住

> [!quote] 一句话总结
> 这套代码将显示模式生成与 SerDes 芯片寄存器配置分开处理：`panel-timing` 决定 DRM/DSI 输入侧时序，`serdes-init-sequence` 决定 MAX96755/MAX96752 的链路、格式、PLL 和 LVDS 输出。排查显示问题时，必须分别确认这两条路径，并验证它们彼此匹配。

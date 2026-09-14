---
title: MAX96755 → MAX96752 从设备树到点屏的完整流程
cssclasses:
  - archive-page
---

# MAX96755 → MAX96752：从设备树到屏幕点亮

本文按照当前工程的实际执行路径，说明 RK3576 从设备树加载，到 MAX96755/MAX96752 初始化，再到 DRM 点亮 LVDS 面板的完整流程。

> [!summary] 核心主线
> 设备树分别描述硬件连接、`panel-timing` 和 `serdes-init-sequence`；驱动将这些配置转换为 DRM bridge、DRM panel、输入侧显示时钟以及 SerDes 寄存器状态，最终完成视频传输和面板点亮。

## 一、整体链路

### 硬件数据链路

1. RK3576 VOP 产生显示帧。
2. Rockchip DSI / MIPI D-PHY 输出 MIPI DSI 数据。
3. MAX96755 Serializer 接收并串行化 DSI 数据。
4. GMSL / SerDes 链路传输视频流。
5. MAX96752 Deserializer 解串并输出 LVDS / Panel 信号。

### 软件框架链路

1. Device Tree 描述设备节点、连接关系和初始化参数。
2. I2C Core 创建 MAX96755/MAX96752 的 I2C client。
3. `serdes-i2c` 主驱动创建 `struct serdes` 和 regmap。
4. `serdes-core` 通过 MFD 创建 bridge、panel、GPIO、pinctrl 等子设备。
5. Linux DRM 连接 VOP、DSI、MAX96755 bridge 和 MAX96752 panel。
6. Rockchip VOP / DSI / PHY 配置输入侧显示时钟。
7. MAX96755 → MAX96752 完成 SerDes 视频传输，面板输出 LVDS。

> [!note] 两条独立路径
> I2C 负责配置 MAX96755/MAX96752 的寄存器；DSI / SerDes 负责传输实时显示视频。控制链路与显示数据链路相互配合，但不是同一条路径。

### 主要代码目录

> [!info] 驱动文件速查
> | 文件 | 主要作用 |
> |---|---|
> | `drivers/mfd/display-serdes/serdes-i2c.c` | SerDes I2C 主驱动与初始化序列 |
> | `drivers/mfd/display-serdes/serdes-core.c` | MFD 子设备创建 |
> | `drivers/mfd/display-serdes/serdes-panel.c` | 通用 DRM panel 与 panel-timing解析 |
> | `drivers/mfd/display-serdes/serdes-bridge.c` | 通用 DRM bridge |
> | `drivers/mfd/display-serdes/maxim/maxim-max96755.c` | MAX96755 专用操作 |
> | `drivers/mfd/display-serdes/maxim/maxim-max96752.c` | MAX96752 专用操作 |

## 二、设备树如何与驱动联动

设备树主要描述以下内容：

- 硬件节点和设备类型；
- I2C 地址；
- 显示数据连接关系；
- 面板显示时序；
- SerDes 芯片寄存器初始化序列；
- GPIO、供电和 pinctrl 资源。

驱动通过 `compatible` 匹配设备树节点，再读取其他属性完成配置。

### MAX96755 Serializer 节点

```dts
serializer@40 {
    compatible = "maxim,max96755";
    reg = <0x40>;

    sel-mipi;
    id-serdes-bridge-split = <0x01>;

    serdes-init-sequence = [
        ...
    ];

    bridge {
        compatible = "maxim,max96755-bridge";
    };
};
```

### MAX96752 Deserializer 节点

```dts
deserializer@48 {
    compatible = "maxim,max96752";
    reg = <0x48>;

    id-serdes-panel-split = <0x01>;
    link = <0x01>;

    serdes-init-sequence = [
        ...
    ];

    panel {
        compatible = "maxim,max96752-panel";

        panel-timing {
            ...
        };
    };
};
```

> [!info] 常见属性作用
> | 属性 | 作用 |
> |---|---|
> | `compatible` | 匹配 Linux 驱动 |
> | `reg` | I2C 地址 |
> | `serdes-init-sequence` | 初始化 SerDes 芯片寄存器 |
> | `panel-timing` | 描述分辨率、时序和像素时钟 |
> | `remote-endpoint` | 描述显示数据连接关系 |
> | `sel-mipi` | 选择 MIPI DSI 输入路径 |
> | `link` | 选择 SerDes 链路 |
> | `lock-gpios` | 获取链路锁定状态 |
> | `pdb-gpios` | 电源或复位控制 |
> | `id-serdes-bridge-split` | 关联 Serializer 与 bridge |
> | `id-serdes-panel-split` | 关联 Deserializer 与 panel |

## 三、I2C 创建 SerDes 主设备

内核解析设备树后，I2C Core 为每个设备节点创建 I2C client。

### MAX96755

1. 设备树节点 `serializer@40` 被解析。
2. I2C Core 创建地址为 `0x40` 的 client。
3. 根据 `compatible` 匹配 `serdes-i2c` 驱动。
4. 进入 `serdes_i2c_probe()`。

### MAX96752

1. 设备树节点 `deserializer@48` 被解析。
2. I2C Core 创建地址为 `0x48` 的 client。
3. 根据 `compatible` 匹配 `serdes-i2c` 驱动。
4. 进入 `serdes_i2c_probe()`。

## 四、通用驱动识别芯片

`serdes_i2c_probe()` 主要完成以下工作：

1. 创建 `struct serdes`。
2. 创建 regmap。
3. 读取并匹配芯片 ID。
4. 获取对应的 `serdes_chip_data`。
5. 解析 GPIO、供电、复位和 lock 资源。

MAX96755 的专用数据定义在 `drivers/mfd/display-serdes/maxim/maxim-max96755.c`：

```c
struct serdes_chip_data serdes_max96755_data = {
    .name = "max96755",
    .serdes_type = TYPE_SER,
    .serdes_id = MAXIM_ID_MAX96755,
    .bridge_ops = &max96755_bridge_ops,
    .pinctrl_info = &max96755_pinctrl_info,
    ...
};
```

该结构将通用 SerDes 框架与 MAX96755 专用的 bridge、GPIO、pinctrl 操作绑定起来。MAX96755 是 `TYPE_SER`，即 Serializer；MAX96752 是 Deserializer。

## 五、解析和执行 `serdes-init-sequence`

设备树中的初始化序列本质上是一段寄存器脚本：

```dts
serdes-init-sequence = [
    0001 0008
    0002 0013
    ffff 000a
    02c4 00b3
];
```

对应含义如下：

> [!example] 初始化序列示例
> | 序列项 | 含义 |
> |---|---|
> | `0001 0008` | 写寄存器 `0x0001 = 0x08` |
> | `0002 0013` | 写寄存器 `0x0002 = 0x13` |
> | `ffff 000a` | 延时 `10 ms` |
> | `02c4 00b3` | 写寄存器 `0x02c4 = 0xb3` |

其中，`ffff value` 表示延时 `value` 毫秒，不是普通寄存器写操作。

通用执行路径为：

1. `serdes_get_init_seq()` 获取序列。
2. `serdes_parse_init_seq()` 解析序列。
3. 保存到 `serdes->serdes_init_seq`。
4. `serdes_device_init()` 执行设备初始化。
5. `serdes_i2c_set_sequence()` 逐项处理。
6. `serdes_reg_write(reg, value)` 完成单次寄存器写入。

普通路径会逐项写入寄存器，并尝试读回校验。

MAX96752 存在专用初始化分支：

```c
if (serdes->chip_data->serdes_id == MAXIM_ID_MAX96752)
    serdes_maxim96752_init(serdes);
```

分析 MAX96752 时，需要同时检查：

1. `maxim-max96752.c` 中的专用初始化函数；
2. DTS 中的 `serdes-init-sequence`；
3. 点屏时 `serdes_panel_enable()` 是否再次执行初始化序列。

## 六、MFD 创建功能子设备

一个 SerDes I2C 芯片会通过 MFD 框架拆分成多个功能子设备。

> [!info] 子设备划分
> | 主设备 | 功能子设备 |
> |---|---|
> | MAX96755 I2C 主设备 | MAX96755 DRM bridge、GPIO、pinctrl |
> | MAX96752 I2C 主设备 | MAX96752 DRM panel、DRM bridge、GPIO、pinctrl |

框架流程如下：

1. `serdes-core.c` 根据 `serdes_id` 选择 `mfd_cell` 数组。
2. 调用 `devm_mfd_add_devices()`。
3. 创建 platform 子设备。
4. 各子设备匹配自己的功能驱动。

各子驱动通过父设备取得同一个 `struct serdes`，共享以下资源：

- `regmap`；
- 芯片类型和芯片操作；
- 初始化序列；
- GPIO 与供电资源；
- 链路状态。

## 七、MAX96755 注册为 DRM bridge

设备树中的 bridge 节点为：

```dts
bridge {
    compatible = "maxim,max96755-bridge";
};
```

该节点匹配 `serdes-bridge.c`，调用过程如下：

1. `serdes_bridge_probe()` 获取父设备的 `struct serdes` 和 regmap。
2. 解析 `remote-endpoint`。
3. 设置 DRM bridge 类型。
4. 调用 `drm_bridge_add()`。
5. 如果存在 `sel-mipi`，通过 `serdes_attach_dsi()` 连接 Rockchip DSI。

MAX96755 bridge 主要负责：

- 注册到 DRM bridge 链；
- 连接 DSI；
- 检测 MAX96755 到 MAX96752 的链路锁定状态；
- 向 DRM 报告连接状态；
- 执行 bridge 生命周期回调。

> [!info] MAX96755 bridge 回调
> | 回调 | 行为 |
> |---|---|
> | `init()` | 空操作 |
> | `attach()` | 检查链路是否 lock |
> | `detect()` | 检查链路和连接状态 |
> | `enable()` | 空操作 |
> | `disable()` | 空操作 |

链路锁定检查顺序为：

1. 优先读取 `lock-gpios`。
2. 没有 GPIO 时读取 MAX96755 寄存器 `0x0013`。
3. 检查 `LOCKED` 位。

## 八、MAX96752 注册为 DRM panel

设备树中的 panel 节点为：

```dts
panel {
    compatible = "maxim,max96752-panel";

    panel-timing {
        ...
    };
};
```

该节点匹配 `serdes-panel.c`，调用过程如下：

1. `serdes_panel_probe()`。
2. `serdes_panel_parse_dt()`。
3. 读取 `panel-size`。
4. 读取 `rate-count-ssc`。
5. 读取 `panel-timing`。
6. 生成 `drm_display_mode`。
7. 调用 `drm_panel_add()`。

## 九、`panel-timing` 转换为 DRM mode

完整转换过程为：

1. DTS `panel-timing`。
2. `of_get_display_timing()` 生成 `struct display_timing`。
3. `videomode_from_timing()` 生成 `struct videomode`。
4. `drm_display_mode_from_videomode()` 生成 `struct drm_display_mode`。
5. 保存到 `serdes_panel->mode`。

核心代码：

```c
struct display_timing dt;
struct videomode vm;

of_get_display_timing(dev->of_node, "panel-timing", &dt);
videomode_from_timing(&dt, &vm);
drm_display_mode_from_videomode(&vm, &serdes_panel->mode);
```

重要字段包括：

```c
mode.clock
mode.hdisplay
mode.hsync_start
mode.hsync_end
mode.htotal
mode.vdisplay
mode.vsync_start
mode.vsync_end
mode.vtotal
mode.flags
```

> [!note] 时钟单位
> DTS 的 `clock-frequency` 单位是 Hz，DRM `mode.clock` 单位是 kHz。例如 `89964000 Hz = 89964 kHz`。

当前示例的计算结果：

- `HTOTAL = 1920 + 40 + 40 + 40 = 2040`；
- `VTOTAL = 720 + 10 + 2 + 3 = 735`；
- 刷新率为 `89,964,000 / (2040 × 735) ≈ 60 Hz`。

## 十、DRM 获取显示模式

DRM 枚举 connector 时调用 panel 的 `get_modes()`：

1. `serdes_panel_get_modes()` 复制 `serdes_panel->mode`。
2. 设置 `DRIVER` / `PREFERRED` 属性。
3. 调用 `drm_mode_set_name()`。
4. 调用 `drm_mode_probed_add()`。
5. 将 mode 加入 connector 的 mode 列表。

此时 DRM 获得以下信息：

> [!info] DRM mode 速查
> | 项目 | 当前示例 |
> |---|---|
> | 分辨率 | `1920 × 720` |
> | 像素时钟 | `89964 kHz` |
> | 刷新率 | 约 `60 Hz` |
> | 同步时序 | H/V total、porch、sync |
> | 极性 | HSYNC、VSYNC、DE、pixel clock |

## 十一、Rockchip 配置 DSI 输入侧时钟

时钟配置路径为：

1. DTS `panel-timing.clock-frequency`。
2. `serdes-panel.c` 解析并生成 `struct drm_display_mode.clock`。
3. DRM connector 保存显示 mode。
4. Rockchip VOP / DSI 根据 mode 配置输入侧时钟。
5. MIPI D-PHY 配置 PLL。
6. 生成 DSI pixel clock 和 lane rate。

Rockchip VOP/DSI 通常会参考：

- `mode.clock`；
- `mode.htotal` 和 `mode.vtotal`；
- DSI lane 数；
- 每像素 bit 数；
- DSI video mode；
- MIPI D-PHY 限制。

> [!warning] 不要混淆不同频率
> 面板 pixel clock、DSI lane bit clock、SerDes link rate 和 MAX96752 LVDS output clock 不是同一个频率。它们存在计算关系，但分别属于显示时序、DSI 传输、SerDes 链路和面板输出等不同阶段。

## 十二、DRM 点屏回调

点屏时的大致顺序如下：

1. `get_modes()` 获取显示模式。
2. bridge attach。
3. `panel prepare`。
4. bridge `pre-enable` / `enable`。
5. Rockchip DSI 开始发送视频。
6. MAX96755 串行化视频流。
7. MAX96752 解串。
8. `panel enable`。
9. 打开背光。

### `panel_prepare`

`serdes_panel_prepare()` 依次完成：

1. 调用 `chip panel_ops->init()`。
2. 调用 `chip panel_ops->prepare()`。
3. 设置 pinctrl sleep。
4. 设置 pinctrl default。

当前 MAX96752 的 `panel_prepare()` 返回 0，没有根据 `mode.clock` 动态计算 PLL。

### `panel_enable`

`serdes_panel_enable()` 依次完成：

1. 调用 `chip panel_ops->enable()`。
2. 调用 `remdev_panel_enable()`。
3. 如果设备类型为 `TYPE_DES`，再次执行 `serdes_i2c_set_sequence()`。
4. 调用 `backlight_enable()`。

因此，MAX96752 可能在两个阶段执行寄存器配置：

> [!warning] 可能存在的重复配置
> **probe / device_init 阶段**执行 MAX96752 专用初始化；**panel_enable 阶段**可能再次执行 Deserializer 初始化序列。调试时需要确认这种重复写寄存器的行为是否符合硬件上电时序。

## 十三、从启动到显示的完整执行顺序

1. 内核解析设备树。
2. I2C 控制器创建 MAX96755/MAX96752 client。
3. `serdes-i2c` 根据 `compatible` 匹配驱动。
4. 创建 `struct serdes` 和 regmap。
5. 读取芯片 ID，绑定 `serdes_chip_data`。
6. 获取并解析 `serdes-init-sequence`。
7. 写入 MAX96755/MAX96752 初始化寄存器。
8. `serdes-core` 创建 MFD 子设备。
9. `serdes-bridge` 创建 DRM bridge。
10. `serdes-panel` 创建 DRM panel。
11. `serdes-panel` 读取 `panel-timing`。
12. 将 `panel-timing` 转换为 `drm_display_mode`。
13. DRM 连接 VOP、DSI、MAX96755 bridge 和 MAX96752 panel。
14. DRM 调用 `get_modes()` 获取显示模式。
15. Rockchip VOP/DSI 根据 mode 配置输入侧时钟。
16. DRM 调用 `panel_prepare()`。
17. DRM 检查 MAX96755 链路是否 lock。
18. Rockchip DSI 开始输出视频。
19. MAX96755 串行化并发送视频流。
20. MAX96752 解串并输出 LVDS。
21. 调用 `panel_enable()`。
22. 打开背光。
23. 面板显示。

## 十四、框架中的对象关系

### 设备模型

> [!info] MFD 子设备关系
> | I2C 主设备 | platform 子设备 |
> |---|---|
> | MAX96755 | bridge、GPIO、pinctrl |
> | MAX96752 | panel、bridge、GPIO、pinctrl |

MFD 子驱动通过父设备获取同一个 `struct serdes`，从而共享 regmap、芯片状态、初始化序列以及 GPIO / 供电资源。

### DRM 对象

> [!info] DRM 对象关系
> | 对象 | 作用 |
> |---|---|
> | `drm_crtc` / VOP | 产生显示帧 |
> | Rockchip DSI | 输出 MIPI DSI 数据 |
> | `drm_bridge: MAX96755` | 接收 DSI、检查链路并加入 bridge 链 |
> | `drm_connector` | 管理可用显示 mode |
> | `drm_panel: MAX96752` | 提供面板 mode 和点屏回调 |

### `struct serdes` 共享内容

`struct serdes` 主要包含：

- dev；
- regmap；
- chip_data；
- serdes_init_seq；
- serdes_bridge；
- serdes_panel；
- GPIO / regulator；
- I2C / link 状态。

## 十五、两条核心配置路径

### 显示时序路径

`DTS panel-timing` → `serdes-panel.c` → `struct drm_display_mode` → DRM connector → Rockchip VOP / DSI / D-PHY → 输入侧显示时钟和视频时序

### SerDes 寄存器路径

`DTS serdes-init-sequence` → `serdes-i2c.c` 解析 → `serdes_reg_write()` → MAX96755/MAX96752 寄存器 → SerDes 链路、输入输出格式、PLL 和 LVDS 输出

> [!abstract] 两条路径必须匹配
> DRM mode 的 pixel clock / 时序必须与 MAX96752 的输出 PLL / LVDS 配置相匹配。前者主要影响 Rockchip 输入侧，后者主要影响 SerDes 链路和面板输出侧。

## 十六、调试建议

### 检查设备树和 DRM mode

确认以下项目：

- clock-frequency；
- hactive / vactive；
- 水平和垂直 porch；
- HSYNC / VSYNC / DE / pixel clock 极性；
- serdes-panel.c打印的 mode.clock；
- htotal 和 vtotal。

### 检查 Rockchip DSI

重点查看：

- DSI pixel clock；
- DSI lane rate；
- lane 数量；
- D-PHY PLL；
- video mode；
- DSI 错误中断。

### 检查 MAX96755

重点查看：

- `lock-gpios`；
- 寄存器 `0x0013`；
- `LOCKED` 位；
- DSI 输入是否有时钟和数据；
- SerDes 链路是否正常。

### 检查 MAX96752

重点查看：

- `serdes-init-sequence`；
- 输入 / 输出格式；
- lane mapping；
- PLL lock 状态；
- LVDS clock 使能；
- 输出 pixel clock；
- 面板供电和背光。

> [!danger] 推荐排查顺序
> 1. 确认设备树节点和 I2C 地址正确。
> 2. 确认 `panel-timing` 被正确解析。
> 3. 确认 DRM 实际生成的 `mode.clock`、`htotal` 和 `vtotal`。
> 4. 确认 Rockchip DSI 有正确的时钟和数据输出。
> 5. 确认 MAX96755 链路处于 `LOCKED` 状态。
> 6. 核对 MAX96752 的 PLL、LVDS、格式和 lane mapping 配置。
> 7. 最后检查面板供电、背光和输出极性。

## 最后记住

> [!quote] 一句话总结
> `panel-timing` 决定 DRM / DSI 输入侧的显示模式和时钟；`serdes-init-sequence` 决定 MAX96755/MAX96752 的寄存器、链路、格式、PLL 和 LVDS 输出。两条配置路径必须彼此匹配，才能正常点屏。

# SiRF VDSS 显示子系统笔记

> 头文件：`linux/include/video/sirfsoc_vdss.h`
> 作用域：SiRF / CSR A7 SoC，仿照 Linux 主线 OMAP DSS 风格的显示框架

---

## 一、整体架构

VDSS 把"显示源 → 图层 → 输出 → 面板"四级流水线抽象为可组合、可注册的对象模型，每级用函数指针表实现多态，配合 `list_head`/`kobject` 接入内核设备模型与 sysfs。

### 数据链路（数据从产生到屏幕）

```
内存帧缓冲 / VIP 外部输入
        │
        ▼
   VPP（视频后处理：去隔行 / 缩放 / 色彩调整）
        │  (bitblt / inline / passthrough / ibv)
        ▼
   Layer（图层 LAYER0~3 / CURSOR，带 colorkey / alpha）
        │
        ▼
   Screen（合成多图层，gamma / vsync / 背景色）
        │
        ▼
   Output（RGB / LVDS1 / LVDS2）
        │
        ▼
   Panel（RGB 屏 / HDMI / LVDS 屏）── Driver（面板驱动）
```

### 控制链路（配置与状态下发）

```
用户态 / 驱动配置
   │
   ├─► sirfsoc_vpp_create_device()        创建 VPP 设备（带 notify 回调）
   │      └─► sirfsoc_vpp_present()       下发 vdss_vpp_op_params
   │
   ├─► sirfsoc_dcu_present()              下发 vdss_dcu_op_params（DCU 合成）
   │
   ├─► sirfsoc_vdss_get_layer()           取图层对象
   │      └─► layer->set_info() / enable() / flip()  配置图层并启用
   │
   ├─► sirfsoc_vdss_get_screen()          取 screen 对象
   │      └─► screen->set_info() / set_output() / apply() / wait_for_vsync()
   │
   ├─► sirfsoc_vdss_get_output() / find_output()
   │      └─► output->ops->connect() / set_timings() / enable()
   │
   └─► sirfsoc_vdss_register_panel()      注册面板
          └─► panel->driver->probe() / connect() / enable() / set_timings()
```

> 中断侧控制：`sirfsoc_lcdc_register_isr()` 注册 ISR，掩码含 DMA 完成、上下溢出、VSYNC（见 `LCDC_INT_*`）。

---

## 二、数据结构速查

### 2.1 基础描述结构

| 结构 | 位置 | 用途 |
|---|---|---|
| `vdss_rect` | :84 | 矩形 left/top/right/bottom，描述源/目的区域 |
| `vdss_surface` | :202 | 一块显示源：fmt/field/width/height/base(物理地址) |
| `vdss_vpp_colorctrl` | :210 | 色彩：hue/brightness/contrast/saturation |
| `vdss_vpp_interlace` | :197 | 去隔行：di_top + di_mode(weave/3median/VMRI) |

### 2.2 VPP 操作参数

| 结构 | 模式 | 用途 |
|---|---|---|
| `vdss_vpp_blt_params` | BitBLT | 源→目的(双缓冲)位块拷贝，做 UI/图形合成 |
| `vdss_vpp_inline_params` | Inline | VPP 直接送显层，无中间 buffer |
| `vdss_vpp_passthrough_params` | Pass-through | 直通，支持去隔行 + flip |
| `vdss_vpp_ibv_params` | IBV | 从外部 VIP 取 3 平面分量，支持缩放/色彩 |
| `vdss_vpp_op_params` | 统一封装 | type + union op，配合 `sirfsoc_vpp_present()` |
| `vdss_vpp_create_device_params` | — | 创建设备时的 notify 回调 |

### 2.3 DCU 操作参数

| 结构 | 用途 |
|---|---|
| `vdss_dcu_inline_params` | DCU inline 合成，2 路 source + flip |
| `vdss_dcu_blt_params` | DCU bitblt，2 路 source → 1 目的 |
| `vdss_dcu_op_params` | 统一封装，配合 `sirfsoc_dcu_present()` |

### 2.4 框架对象（核心实体）

| 结构 | 角色 | 关键字段/方法 |
|---|---|---|
| `sirfsoc_video_timings` | 时序契约 | xres/yres/pixel_clock, hsw/hfp/hbp, vsw/vfp/vbp, *_level, interlace, pclk_edge |
| `sirfsoc_vdss_layer_info` | 图层运行时配置 | src/dst surface+rect, colorkey, alpha, disp_mode |
| `sirfsoc_vdss_layer` | 图层对象 | enable/disable, set_info/get_info, flip, set_screen |
| `sirfsoc_vdss_screen_info` | screen 配置 | top_layer, blank_color, back_color |
| `sirfsoc_vdss_screen` | 合成单元 | set_output, apply, wait_for_vsync, set_gamma, set_err_diff |
| `sirfsoc_vdss_rgb_ops` | RGB 输出操作集 | connect/enable, check_timings/set_timings, set_data_lines |
| `sirfsoc_vdss_lvds_ops` | LVDS 输出操作集 | 同上 + set_fmt(VESA6/8bit) + set_mode(slave/sync) |
| `sirfsoc_vdss_output` | 输出对象 | ops(rgb/lvds union), supported_panel, lcdc_id/screen_id/id |
| `sirfsoc_vdss_panel` | 面板对象 | type, phy.data_lines, timings, driver, src, state |
| `sirfsoc_vdss_driver` | 面板驱动操作集 | probe/remove, connect/enable, get_resolution/bpp, set_timings |

---

## 三、关键枚举

- `enum sirfsoc_panel_type`：NONE / RGB / HDMI / LVDS
- `enum vdss_output`：RGB=1 / LVDS1=2 / LVDS2=4（位掩码）
- `enum vdss_layer`：LAYER0~3, CURSOR=6
- `enum vdss_screen` / `enum vdss_lcdc`：SCREEN0/1, LCDC0/1
- `enum vdss_pixelformat`：RGB 各 BPP / YUV YUYV/UYVY/NV12/YV12… / CUSTOM=0x1000
- `enum vdss_disp_mode`：NORMAL / INLINE / PASS_THROUGH / IBV
- `enum vdss_vpp_op_type`：IDEL / BITBLT / INLINE / PASS_THROUGH / IBV
- `enum vdss_field`：NONE/TOP/BOTTOM/INTERLACED/SEQ_TB/…(隔行场模式)

---

## 四、LCDC 中断掩码

| 宏 | 含义 |
|---|---|
| `LCDC_INT_L0~3_DMA` | Layer 0~3 DMA 传输完成 |
| `LCDC_INT_L0~3_OFLOW` | Layer 0~3 溢出 |
| `LCDC_INT_L0~3_UFLOW` | Layer 0~3 下溢 |
| `LCDC_INT_VSYNC` | 垂直同步 |
| `LCDC_INT_ALL` | 全部 |

---

## 五、外部 API 速查

### Panel
```c
sirfsoc_vdss_register_panel() / unregister_panel()
sirfsoc_vdss_get_primary_device() / get_secondary_device()
sirfsoc_vdss_get_panel() / put_panel() / find_panel() / get_next_panel()
sirfsoc_vdss_is_initialized() / lvds_is_initialized()
```

### Output
```c
sirfsoc_vdss_output_set_panel() / unset_panel()
sirfsoc_vdss_register_output() / unregister_output()
sirfsoc_vdss_get_output() / find_output() / find_output_from_panel()
```

### Screen / Layer
```c
sirfsoc_vdss_get_screen(lcdc_index, num)
sirfsoc_vdss_get_layer(lcdc_index, num)
sirfsoc_vdss_get_layer_from_screen(scn, id, rearview)
sirfsoc_vdss_set_exclusive_layers() / set_video_layers()
sirfsoc_vdss_check_size()  // 校验缩放/裁剪合法性
```

### 时序转换
```c
videomode_to_sirfsoc_video_timings() / sirfsoc_video_timings_to_videomode()
```

### LCDC ISR
```c
sirfsoc_lcdc_register_isr(lcdc_index, isr, arg, mask)
sirfsoc_lcdc_unregister_isr(...)
```

### VPP / DCU
```c
sirfsoc_vpp_create_device() / destroy_device() / present()
sirfsoc_vpp_is_passthrough_support(fmt)
sirfsoc_dcu_reset() / present() / is_inline_support(fmt, field)
```

### Gamma / Encoder
```c
sirfsoc_vdss_get_primary_display_gamma() / set_primary_display_gamma()
sirfsoc_vdss_panel_enable_encoder() / disable_encoder() / find_encoder()
sirfsoc_vdss_get_num_lcdc() / get_num_screens(lcdc_index) / get_num_layers(lcdc_index)
```

---

## 六、内联辅助

```c
sirfsoc_vdss_panel_is_connected(panel)  // = panel->src != NULL
sirfsoc_vdss_panel_is_enabled(panel)    // = panel->state == ENABLED
```

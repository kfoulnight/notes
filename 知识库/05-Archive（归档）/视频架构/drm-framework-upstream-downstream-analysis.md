---
title: DRM 框架上游下游结合分析（RK3576）
cssclasses:
  - archive-page
---

# DRM 框架上游下游结合分析（RK3576）

本文基于 RK3576 的 Android BSP 内核源码，逐行对应"上游 DRM 框架接口"与"下游驱动实现"，说明 DRM 框架怎么使用、上下游怎么结合。

分析源码位置：

- `drivers/gpu/drm/rockchip/rockchip_drm_vop2.c`（VOP2 显示控制器）
- `drivers/gpu/drm/panel/panel-simple.c`（面板）
- `arch/arm64/boot/dts/rockchip/rk3576s-tablet-v10.dts`（RK3576 平板设备树）

> [!summary] 核心主线
> 上游框架定义"怎么用"（注册、回调、原子提交），下游驱动定义"用什么硬件"（寄存器、时序、面板）。结合点是 `drm_driver` 注册和 atomic/vblank 回调：框架调用你，你操作硬件。

## 一、DRM 框架整体架构

DRM 是分层框架，使用它的身份有两种：

### 内核驱动开发者

工作不是"调用框架"，而是实现框架要求实现的东西，再注册进去，让框架在合适时机调用你。

| 你写的东西 | DRM Core 框架做的事 |
|---|---|
| `struct drm_driver` | 分配 / 注册 DRM 设备 |
| `.fops` | 暴露 `/dev/dri/card0` 给用户态 |
| `.gem_create` | GEM buffer 管理 |
| `.prime_*` | dma-buf export/import |
| `.irq_handler` | vblank 中断回调 |
| CRTC/plane/connector 的 atomic 回调 | 登记到 DRM objects |

### 用户态程序

`open("/dev/dri/card0")` 拿资源，建 framebuffer，atomic commit。

| libdrm API | 对应 ioctl |
|---|---|
| `drmModeGetResources()` | `DRM_IOCTL_MODE_GETRESOURCES` |
| `drmModeAddFB2()` | `DRM_IOCTL_MODE_ADDFB2` |
| `drmModeAtomicCommit()` | `DRM_IOCTL_MODE_ATOMIC` |

框架怎么使用，一句话：用户态通过 ioctl 用框架，框架通过注册的回调用你的驱动，你的驱动操作硬件。每一层只负责自己的事。

## 二、上游与下游的含义

DRM 开发里"上游/下游"有两层意思，都决定日常怎么写代码。

### 代码层面

| 角色 | 是什么 | 本 BSP 里的例子 |
|---|---|---|
| 上游（框架） | 定义接口、调度时机、通用逻辑 | `drm_crtc_funcs`、`drm_atomic_helper_*`、`drm_crtc_handle_vblank`、`drm_panel_funcs` |
| 下游（驱动） | 实现接口、操作具体硬件 | `vop2_crtc_*`、`vop2_plane_*`、`panel_simple_*` |
| 下游（DTS） | 承载硬件差异（时序、屏参、引脚） | `rk3576s-tablet-v10.dts` 的 `dsi_panel` 节点 |

### 内核版本层面

| 项目 | 上游（mainline） | 下游（厂商 BSP） |
|---|---|---|
| 来源 | Linux 官方主线 | 基于上游 tag fork + 厂商补丁 |
| 接口 | 统一、通用、稳定 | 功能优先，常有私有接口 |
| 代码风格 | review 严格、干净 | 量大、快、可能有 hack |
| 特性 | 只收通用标准的东西 | 先满足量产和性能 |

同一个文件名（如 `rockchip_drm_vop2.c`），主线版和 BSP 版内容差别很大。BSP 版通常多很多私有改动。

## 三、整条显示链路

一个画面从软件走到屏幕：

用户态（Android HWC / SurfaceFlinger）
→ `drmModeAtomicCommit()`
→ DRM Core（上游框架）`drm_atomic_helper_commit()`
→ Rockchip DRM（`rockchip_drm_drv.c`）
→ VOP2（CRTC + Plane，`rockchip_drm_vop2.c`）
→ DW-MIPI-DSI（encoder/bridge，`dw-mipi-dsi-rockchip.c`）
→ simple-panel（connector，`panel-simple.c`）
→ LCD 屏

上下游分界线：`drm_panel_funcs`、`drm_crtc_funcs`、`drm_plane_funcs` 这些接口是上游框架定义的；各 `xxx_funcs` 里被调用到的 `xxx_` 前缀函数是下游实现。

## 四、逐行对应：DTS 声明一块屏

文件：`arch/arm64/boot/dts/rockchip/rk3576s-tablet-v10.dts`

屏厂差异全部收在 DTS：换屏 = 改 DTS，驱动不动。这是"下游"承载差异的第一层。

| 属性 | 作用 |
|---|---|
| `compatible = "simple-panel-dsi"` | 匹配 `panel-simple.c` 里注册的 id |
| `backlight` | 背光，给 prepare/enable 用 |
| `power-supply` | 电源，给 probe/unprepare 用 |
| `reset-gpios` | 复位脚 |
| `prepare-delay-ms` 等 | 下游私有时序参数 |
| `dsi,format` / `dsi,lanes` | 数据格式与通道数 |
| `panel-init-sequence` | 屏厂给的初始化寄存器序列（原样照抄） |
| `display-timings` | 时序，驱动 `get_modes()` 读这里 |
| `port@0` endpoint | 面板与 DSI 的 remote-endpoint 连接 |

## 五、逐行对应：panel-simple.c（connector 侧）

### 接口与实现绑定

`panel-simple.c:715`

```c
static const struct drm_panel_funcs panel_simple_funcs = {
    .disable     = panel_simple_disable,
    .unprepare   = panel_simple_unprepare,
    .prepare     = panel_simple_prepare,
    .enable      = panel_simple_enable,
    .get_modes   = panel_simple_get_modes,
    .get_timings = panel_simple_get_timings,
};
```

四个回调是 DRM 框架在合适时机调用的，驱动只负责实现，不知道谁、什么时候调用。

### probe 注册流程

`panel-simple.c:996,1004`

```c
drm_panel_init(&panel->base, dev, &panel_simple_funcs, connector_type);
drm_panel_add(&panel->base);
```

`drm_panel_add()` 是上游框架 API，把 panel 挂进框架，DSI 驱动 `of_drm_find_panel()` 才能找到它。

### prepare / enable

`panel-simple.c:595,642`

```c
static int panel_simple_prepare(struct drm_panel *panel)
{
    // 上电 -> 发初始化序列 -> 复位脚时序
}

static int panel_simple_enable(struct drm_panel *panel)
{
    // 打开背光等
}
```

控制屏亮起来的硬件动作，是下游最纯粹的部分：屏厂给的序列原样写进 DTS，驱动照发。

### get_modes

`panel-simple.c:664`

```c
static int panel_simple_get_modes(struct drm_panel *panel,
                                  struct drm_connector *connector)
{
    // 从 DTS display-timings 读时序
    // drm_mode_duplicate() 生成 mode
    // drm_mode_probed_add(connector, mode) 挂到 connector
}
```

上游约定：get_modes 返回支持的模式给框架，框架据此驱动 modeset。时序来自 DTS，就是它与专有驱动唯一区别。

## 六、逐行对应：VOP2（CRTC + Plane 侧）

### CRTC 接口绑定

`rockchip_drm_vop2.c:12141, 12539`

```c
static const struct drm_crtc_helper_funcs vop2_crtc_helper_funcs = {
    .mode_valid     = vop2_crtc_mode_valid,
    .mode_fixup     = vop2_crtc_mode_fixup,
    .atomic_check   = vop2_crtc_atomic_check,
    .atomic_begin   = vop2_crtc_atomic_begin,
    .atomic_flush   = vop2_crtc_atomic_flush,
    .atomic_enable  = vop2_crtc_atomic_enable,
    .atomic_disable = vop2_crtc_atomic_disable,
};

static const struct drm_crtc_funcs vop2_crtc_funcs = {
    .set_config   = drm_atomic_helper_set_config,
    .page_flip    = drm_atomic_helper_page_flip,
    .reset        = vop2_crtc_reset,
    .atomic_duplicate_state = vop2_crtc_duplicate_state,
    .enable_vblank = vop2_crtc_enable_vblank,
    .disable_vblank = vop2_crtc_disable_vblank,
    ...
};
```

`.set_config` / `.page_flip` 直接用上游 helper，这就是"能复用框架就别自己写"。

### Plane 接口绑定

`rockchip_drm_vop2.c:6531, 6837`

```c
static const struct drm_plane_helper_funcs vop2_plane_helper_funcs = {
    .atomic_check   = vop2_plane_atomic_check,
    .atomic_update  = vop2_plane_atomic_update,
    .atomic_disable = vop2_plane_atomic_disable,
};
```

plane 的 `atomic_update` 就是"写 plane 硬件状态"那一步。

### 真正写硬件寄存器

`vop2_crtc_atomic_enable`，行 9311 起

```c
static void vop2_crtc_atomic_enable(struct drm_crtc *crtc,
                                    struct drm_atomic_state *state)
{
    struct drm_crtc_state *old_cstate = drm_atomic_get_old_crtc_state(state, crtc);
    struct drm_display_mode *adjusted_mode = &crtc->state->adjusted_mode;

    u16 hdisplay = adjusted_mode->crtc_hdisplay;
    u16 htotal   = adjusted_mode->crtc_htotal;
    u16 vdisplay = adjusted_mode->crtc_vdisplay;
    u16 vtotal   = adjusted_mode->crtc_vtotal;
    // 把 porch 算成 VP 寄存器值，写进 VOP2 的 VP 时序寄存器
}
```

这里体现上下层关系：mode 是上游框架算好传进来的（`adjusted_mode`），驱动只负责把它翻译成寄存器值写进去。

### vblank

`vop2_crtc_enable_vblank` 行 6849，`vop2_handle_vblank` 行 12566

```c
static int vop2_crtc_enable_vblank(struct drm_crtc *crtc)
{
    VOP_INTR_SET_TYPE(vop2, intr, enable, FS_FIELD_INTR, 1);
    // 打开"帧开始"中断，vblank 的硬件来源
}
```

中断来了之后（下游 ISR 里调 `vop2_handle_vblank` → 调上游 API）：

```c
drm_crtc_handle_vblank(crtc);                       // 上游框架 API，驱动只上报
drm_crtc_send_vblank_event(crtc, vp->event);        // 发 page-flip 完成事件给用户态
```

下游负责"什么时候发生"（硬件中断），上游负责"拿这个时机干什么"（vblank 记账、发事件）。

## 七、上/下游调用时序

一次页面翻转，atomic commit 下来，框架按顺序调用各下游回调：

1. `prepare_fb(plane, new_state)`：框架提交前准备 fb（plane 资源）
2. `atomic_begin(crtc, old)`：VOP2 进入更新窗口
3. `atomic_update(plane, old)`：VOP2 写 WIN 寄存器
4. `atomic_flush(crtc, old)`：VOP2 写 GO bit，所有更新同时生效
5. 等 vblank
6. `vop2_handle_vblank()`：下游 ISR 上报，`drm_crtc_handle_vblank()` 上游记账/发事件
7. `cleanup_fb(plane, old)`：框架释放旧 fb
8. 用户态收到 page-flip event

panel 侧回调穿插在 modeset 里：

1. prepare 阶段：`panel->prepare()`，下游 DTS 序列发出去
2. enable 阶段：`dsi->bridge_enable` → `panel->enable()`，开背光
3. 探测时：`get_modes()` 已被调用
4. disable：`panel->unprepare()` / `panel->disable()`

## 八、三条铁律

1. 下游永远不知道"什么时候"被调用，只负责"被调用时干什么"（回调注册）。
2. 上游永远不碰硬件寄存器，只负责"按什么顺序调用谁"。
3. 差异往 DTS 塞，逻辑往驱动放，这就是 BSP 为什么能一套驱动适配多种屏。

## 九、在 BSP 里动手的场景

| 场景 | 改动位置 |
|---|---|
| 换一块新屏 | 只改 DTS（新屏参、新初始化序列）；若 compatible 驱动不认识，再在 `panel-simple.c` 的 compatible 表加一行 |
| 修显示花屏/时序错乱 | `vop2_crtc_atomic_enable()` 看 VP 时序寄存器换算；或 `vop2_plane_atomic_update()` 看 WIN 地址、stride、格式 |
| 加硬件不支持的功能（如新缩放） | `vop2_plane_atomic_check()` 加检查，`vop2_plane_atomic_update()` 写对应寄存器 |
| 排查 vblank 卡帧 | 跟 `enable_vblank` → ISR → `vop2_handle_vblank` → `drm_crtc_handle_vblank` 这条链 |

## 最后记住

> [!quote] 一句话总结
> 上游框架定义"怎么用"（注册、回调、原子提交），下游驱动定义"用什么硬件"（寄存器、时序、面板）。结合点是 `drm_driver` 注册和 atomic/vblank 回调——框架调用你，你操作硬件；backport 是把上游的东西搬进来，upstream 是把下游的好东西交出去。

---
title: Linux DRM 与视频显示驱动基础
cssclasses:
  - archive-page
---
[【项目原理】DRM驱动概念、组成、框架、源码分析-CSDN博客](https://blog.csdn.net/qq_41709234/article/details/129472180)
# Linux DRM 与视频显示驱动基础

本文介绍 Linux DRM（Direct Rendering Manager）在图形和视频显示系统中的基础作用，重点说明 KMS 显示管线、Framebuffer、Plane、CRTC、Connector、dma-buf 和 Atomic Commit 等组件，以及它们在嵌入式视频平台中的协作方式。

> [!summary] 核心主线
> DRM 负责管理图形硬件和显示输出：KMS 决定画面如何显示，Framebuffer 描述显示缓冲区，Plane 组织图层，CRTC 产生扫描时序，Connector 连接显示接口，dma-buf 负责视频帧跨设备共享，Atomic Commit 负责一次性切换完整显示状态。

## 一、DRM 是什么

DRM 全称是 **Direct Rendering Manager**，是 Linux 内核中的图形和显示设备管理框架。这里的 DRM 不是数字版权管理。

它最初主要服务于 GPU，现在也负责显示控制器、显存、显示管线、显示时序和用户态图形接口。

整体链路可以理解为：

```text
应用程序
  -> Wayland Compositor / Xorg / Android SurfaceFlinger
  -> Mesa / libdrm / 厂商用户态驱动
  -> DRM 内核子系统
  -> GPU、显示控制器、HDMI、DP、MIPI-DSI、LCD
  -> 显示面板
```

> [!info] DRM 的三类核心问题
> | 问题 | DRM 负责的内容 |
> |---|---|
> | 谁访问硬件 | 设备打开、权限、进程和资源管理 |
> | 缓冲区在哪里 | 显存对象、Framebuffer、dma-buf |
> | 如何送到屏幕 | 分辨率、刷新率、Plane、时序和输出接口 |

## 二、KMS：内核模式设置

KMS（Kernel Mode Setting）负责显示输出本身，包括：

- 设置显示分辨率和刷新率。
- 设置像素格式和显示时序。
- 配置显示管线和图层。
- 检测显示器连接状态。
- 管理显示器电源状态。
- 在安全时刻切换显示缓冲区。

### DRM 显示对象

> [!info] KMS 对象速查
> | 对象 | 作用 |
> |---|---|
> | Connector | 表示 HDMI、DP、MIPI-DSI、eDP 等物理输出连接 |
> | Encoder | 把像素数据转换为特定接口信号 |
> | CRTC | 产生扫描输出、同步信号和刷新时序 |
> | Plane | 显示图层，支持缩放、裁剪、透明度和叠加 |
> | Framebuffer | 描述待显示的图像缓冲区 |
> | Bridge | 连接显示控制器和外部转换芯片 |
> | Panel | 表示具体 LCD/OLED 面板 |

典型显示链路：

```text
Framebuffer
  -> Plane
  -> CRTC
  -> Encoder
  -> Bridge
  -> Connector
  -> Panel
```

### Framebuffer

Framebuffer 是 DRM 对图像缓冲区的描述，通常包含：

- 图像宽度和高度。
- 像素格式，例如 `XRGB8888`、`RGB565`、`NV12`。
- 每行跨度，也就是 `pitch` 或 `stride`。
- 底层显存对象或 dma-buf 句柄。
- 多平面图像的地址和偏移。

Framebuffer 本身通常不直接保存全部像素数据，而是描述底层显存或共享缓冲区如何被显示控制器读取。

### Plane

Plane 是显示硬件中的图层，常见类型如下：(drm_plane是显示控制器的一层输入通道；一个 CRTC 可以组合多个 plane.)

> [!info] Plane 类型
> | 类型 | 作用 |
> |---|---|
> | Primary Plane | 主画面图层 |
> | Overlay Plane | 额外叠加的视频或 UI 图层 |
> | Cursor Plane | 鼠标指针或硬件光标 |

Plane 可能支持缩放、裁剪、Alpha、Z-order、像素格式转换、旋转和翻转。Android 和 Wayland 常利用多个 Plane 进行硬件合成，减少 GPU 负担。

### CRTC

CRTC 负责把 Plane 中的像素按照显示时序扫描输出，主要控制：(负责时序和扫描输出，不等于物理显示接口)

- 水平和垂直同步。
- 分辨率和刷新率。
- VBlank。
- 当前显示的 Framebuffer。
- Page Flip 完成事件。

CRTC 不一定等于物理显示器，一个 CRTC 可以通过不同 Encoder 和 Connector 输出到不同接口。

### Connector 与 Panel

Connector 表示显示输出接口，例如 HDMI、DP、LVDS、MIPI-DSI 和 eDP。它可以报告连接状态、支持的显示模式和 EDID 信息。

Panel 表示实际 LCD 或 OLED 面板，通常负责面板电源、复位、背光、初始化命令和休眠唤醒。

## 三、DRM 内存管理

### GEM

GEM（Graphics Execution Manager）是 DRM 中常见的显存对象管理机制，负责：

- 创建和释放显存对象。
- 管理用户态句柄。
- 支持 mmap 映射。
- 建立对象与 Framebuffer 的关系。
- 管理缓冲区生命周期。

```text
GEM object
  -> GEM handle
  -> DRM framebuffer
  -> Plane / CRTC
```

### TTM

TTM（Translation Table Maps）用于更复杂的 GPU 内存管理，可以在系统内存、显存、缓存和 I/O 映射区域之间管理或迁移对象。现代驱动可能使用 TTM、GEM，或基于 GEM 的自定义内存管理。

### Dumb Buffer

Dumb Buffer 是 DRM 提供的简单缓冲区接口，适合基础 framebuffer、DRM 测试程序和没有 GPU 加速需求的嵌入式显示验证。它实现简单，但不适合复杂的 GPU 加速场景。

## 四、dma-buf：视频帧跨设备共享

`dma-buf` 用于多个硬件模块共享同一块缓冲区，避免反复内存拷贝。

典型视频链路：

```text
摄像头 ISP
  -> dma-buf
视频编码器 / 解码器
  -> dma-buf
GPU
  -> dma-buf
DRM Plane
  -> 显示屏
```

在嵌入式平台中，视频帧通常很大。使用 dma-buf 零拷贝可以降低 CPU 占用、内存带宽、显示延迟和功耗。

> [!info] 相关机制
> | 机制 | 作用 |
> |---|---|
> | PRIME | DRM 对象跨设备共享 |
> | dma-buf fd | 用户态传递共享缓冲区 |
> | IOMMU | 为不同设备建立地址映射 |
> | Buffer import/export | 在 GPU、VPU、ISP 和显示控制器之间共享 |

## 五、Atomic Modeset

Atomic KMS 用于一次性提交完整显示状态，避免显示管线处于半更新状态。一次 Atomic Commit 可以同时修改 Plane、CRTC、Connector、Framebuffer、显示位置、缩放、Alpha 和 Z-order。

典型流程：

```text
创建 Framebuffer
  -> 设置 Plane 参数
  -> 设置 CRTC 参数
  -> 绑定 Connector
  -> atomic commit
  -> 硬件在安全时刻统一切换
```

Atomic Commit 特别适合多图层合成、多屏显示、动态分辨率切换、Android 和 Wayland 环境。

## 六、VBlank 与 Page Flip

### VBlank

VBlank 是一帧显示结束、下一帧开始前的垂直空白期。DRM 可以利用 VBlank 做帧同步、避免撕裂、统计刷新节奏和触发 Page Flip 完成事件。

### Page Flip

Page Flip 是把显示控制器当前使用的 Framebuffer 切换到另一块已经绘制完成的缓冲区。

```text
Buffer A 正在显示
Buffer B 已经绘制完成
  -> VBlank 到来
  -> 切换到 Buffer B
```

如果不在合适的显示时刻切换，可能出现画面撕裂。

## 七、DRM 与视频驱动的关系

DRM 不是完整的视频编解码驱动，各子系统职责不同：

> [!info] 视频显示相关组件
> | 模块 | 主要职责 |
> |---|---|
> | V4L2 | 摄像头、视频采集和视频编解码接口 |
> | VPU | H.264、H.265 等硬件编解码 |
> | DRM/KMS | 显示控制器和显示输出 |
> | GPU | 2D/3D 加速和图形合成 |
> | dma-buf | 在硬件模块之间共享视频帧 |
> | Media Framework | 组织采集、编解码、显示流程 |

视频播放链路通常是：

```text
文件
  -> Demuxer 解封装
  -> H.264/H.265 Decoder
  -> V4L2 或厂商硬件解码器
  -> dma-buf
  -> DRM Plane
  -> CRTC
  -> HDMI / MIPI-DSI / eDP
  -> 显示面板
```

## 八、嵌入式平台如何使用 DRM

### 常用验证工具

- `modetest`：查看和设置 DRM 显示资源。
- `kmscube`：验证 DRM/KMS 和 GPU 基础能力。
- `drm_info`：查看 DRM 对象和属性。
- `weston-simple-egl`：验证 Wayland/EGL 显示链路。

查看 DRM 设备：

`ls -l /dev/dri/`

查看 Connector 状态：

`cat /sys/class/drm/card0-*/status`

查看 DRM 资源：

`modetest -M <drm_driver>`

查看对象详情：

`drm_info`

### 内核配置方向

常见基础配置包括：

```text
CONFIG_DRM
CONFIG_DRM_KMS_HELPER
CONFIG_DRM_FBDEV_EMULATION
CONFIG_DRM_GEM_SHMEM_HELPER
CONFIG_DRM_PANEL
CONFIG_DRM_BRIDGE
CONFIG_DRM_MIPI_DSI
```

芯片平台还需要启用对应厂商驱动，例如：

```text
CONFIG_DRM_ROCKCHIP
CONFIG_DRM_SUN4I
CONFIG_DRM_NXP
CONFIG_DRM_MESON
```

### 设备树关系

一个嵌入式显示链路通常是：

```text
display-controller
  -> port / endpoint
  -> encoder
  -> bridge
  -> connector
  -> panel
```

设备树常见配置内容包括显示控制器、`port`/`endpoint`、Pixel Clock、HSync/VSync、Panel Timing、背光 PWM、电源 regulator、复位 GPIO 和 MIPI DSI lane 数量。

## 九、常见问题与排查

> [!warning] 屏幕没有图像
> 1. 确认 DRM 驱动 probe 成功。
> 2. 确认 `/dev/dri/card0` 已生成。
> 3. 确认 Connector 状态是 `connected`。
> 4. 确认 Panel 和 Bridge 正确绑定。
> 5. 检查显示时序和 Pixel Clock。
> 6. 检查 Framebuffer、Pixel Format 和 Stride。
> 7. 检查 Plane 是否启用、dma-buf 是否有效。
> 8. 使用 `modetest` 或 `drm_info` 查看实际资源。

> [!faq] 图像撕裂
> 检查是否使用 Page Flip 或 Atomic Commit，是否在 VBlank 时切换 Framebuffer，以及是否有代码直接修改正在显示的缓冲区。

> [!faq] 颜色异常
> 重点检查 RGB/YUV 格式、通道顺序、色深、Stride、字节序和硬件色彩空间转换配置。

> [!faq] 视频播放卡顿
> 检查解码输出是否通过 dma-buf 零拷贝送入 DRM Plane，同时观察内存带宽、VBlank、Plane 缩放能力和显示刷新率。

## 最后记住

> [!quote] 一句话总结
> DRM 是 Linux 图形和显示设备的内核管理框架。KMS 负责“怎么显示”，GEM/TTM 负责“缓冲区怎么管理”，dma-buf 负责“缓冲区如何跨设备共享”，Atomic Commit 和 VBlank 负责“如何稳定切换画面”。在嵌入式视频系统中，DRM 通常位于硬件解码器和显示面板之间。

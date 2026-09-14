---
title: Linux DRM 数据链路与控制链路分析
tags:
  - drm
  - linux
  - display
  - kernel
cssclasses:
  - archive-page
---
# Linux DRM 数据链路与控制链路分析

> [!info] 本文范围
> 适用源码：`/home/ur/Android/android/kernel-6.1`
>
> 重点范围：DRM Core、KMS、Atomic Modeset、GEM、DMA-BUF、Framebuffer、Plane、CRTC、vblank/page-flip。
>
> 本文以 Linux 6.1 DRM 通用框架为主，具体 SoC 的显示控制器寄存器和提交方式由各自驱动实现。

> [!summary] 核心主线
> DRM/KMS 可以分成控制链路和数据链路：控制链路通过 ioctl 和 atomic state 把用户态配置提交到硬件；数据链路通过 GEM、DMA-BUF、Framebuffer 和 Plane 把像素 buffer 送到 CRTC scanout，最后经 encoder、bridge、connector 输出到屏幕。

## 一、先建立整体模型

DRM/KMS 可以看成两条相互配合的链路：

```text
控制链路：
用户态 compositor
  -> /dev/dri/cardN
  -> ioctl
  -> drm_ioctl()
  -> DRM_IOCTL_MODE_ATOMIC
  -> drm_mode_atomic_ioctl()
  -> drm_atomic_state
  -> atomic_check()
  -> atomic_commit()
  -> atomic_commit_tail()
  -> CRTC/encoder/bridge/plane 驱动回调
  -> 显示控制器寄存器

数据链路：
应用渲染/解码
  -> GEM/GBM buffer object
  -> dma-buf fd（可选，用于跨子系统共享）
  -> DRM GEM handle
  -> drm_framebuffer
  -> drm_plane_state.fb
  -> prepare_fb：pin/map/等待 fence
  -> plane atomic_update
  -> 硬件读取 DMA 地址和格式
  -> scanout
  -> CRTC 输出像素流
  -> encoder/bridge
  -> connector/面板/HDMI/DP/MIPI-DSI
```

> [!abstract] 最重要的认识
> - `drm_framebuffer` 不是像素存储本身，而是对 GEM/其他 backing storage 的格式和布局描述。
> - `drm_plane` 是显示控制器的一层输入通道；一个 CRTC 可以组合多个 plane。
> - `drm_crtc` 负责时序和扫描输出，不等于物理显示接口。
> - `encoder` 和 `bridge` 负责把 CRTC 的像素流转换、传输到 connector。
> - Atomic 的核心价值是：把多个 object 的新状态作为一个整体检查和提交，避免半更新状态。

## 二、DRM 对象关系：像素如何走到屏幕

```text
GEM object / dma-buf
        |
        |  一个或多个 memory handle
        v
drm_framebuffer
  - width / height
  - pixel format（DRM_FORMAT_XRGB8888 等）
  - pitches
  - offsets
  - modifier
        |
        v
drm_plane_state
  - fb
  - crtc
  - src_x/src_y/src_w/src_h
  - crtc_x/crtc_y/crtc_w/crtc_h
  - zpos / alpha / rotation 等属性
        |
        v
drm_crtc_state
  - mode
  - active
  - mode_changed
        |
        v
CRTC -> encoder -> bridge -> connector -> panel/显示器
```

### Plane、CRTC、Connector 的区别

> [!info] DRM 显示对象速查
> | 对象 | 主要职责 | 常见误解 |
> |---|---|---|
> | Plane | 接收 framebuffer，做裁剪、缩放、格式/混合等 | 不是一块显存 |
> | CRTC | 产生像素扫描时序和 vblank | 不是物理 HDMI 接口 |
> | Encoder | 把像素流编码成某种传输格式 | 不负责保存 framebuffer |
> | Bridge | 可串联的显示链路转换器 | 可能连接 PHY、转换芯片或面板控制器 |
> | Connector | 表示用户可见的输出连接状态和能力 | 不一定对应一个可插拔插座，固定面板也可使用 |
> | Framebuffer | 描述像素内存的格式、尺寸和布局 | 不是底层 GEM allocation |

一个典型显示路径可能是:

```text
GEM buffer
  -> primary plane
  -> CRTC
  -> RGB/DSI encoder
  -> bridge chain
  -> panel
```

也可能存在多个 plane：

```text
video plane + UI plane + cursor plane
        \       |       /
             CRTC
```
硬件能否支持某种格式、缩放、旋转、压缩 modifier，最终由 driver 的 `atomic_check` 验证。

## 三、控制链路：用户态如何控制 DRM

### 打开设备和权限
用户态通常打开：

- `/dev/dri/cardN`：具有 modeset/KMS 控制能力，compositor 通常使用它。
- `/dev/dri/renderDN`：render node，面向渲染，不承担 KMS modeset，不需要 DRM master。

DRM Core 使用 `struct drm_file` 跟踪每个打开实例的上下文，包括：

- 文件对应的 `drm_device`；
- GEM handle 表；
- PRIME/dma-buf handle 缓存；
- 是否启用 atomic client capability；
- master/auth 关系。

KMS 控制通常需要 DRM master。`drm_ioctl.c` 的 ioctl 描述表会为每个 ioctl 标记 `DRM_MASTER`、`DRM_RENDER_ALLOW` 等权限。

### ioctl 分发
核心入口在：

- `drivers/gpu/drm/drm_ioctl.c:813`：`drm_ioctl()`
- `drivers/gpu/drm/drm_ioctl.c:557`：`DRM_IOCTL_DEF()`
- `drivers/gpu/drm/drm_ioctl.c:689`：`DRM_IOCTL_MODE_ATOMIC` 注册

典型过程：

```text
用户态 ioctl(fd, DRM_IOCTL_MODE_ATOMIC, arg)
  -> drm_ioctl()
  -> 根据 ioctl number 查 drm_ioctl_desc
  -> drm_ioctl_permit() 检查权限
  -> copy_from_user / compat 处理
  -> drm_mode_atomic_ioctl()
```

`drm_ioctl()` 已经把用户参数整理成内核侧数据后，再调用 `drm_ioctl_t` 类型的处理函数。驱动专用 ioctl 则通常通过 driver 自己的 ioctl 表扩展。

### Atomic ioctl 详细控制链路
核心实现：`drivers/gpu/drm/drm_atomic_uapi.c:1281`。

```text
drm_mode_atomic_ioctl()
  1. 检查 DRIVER_ATOMIC
  2. 检查 file_priv->atomic
  3. 检查 flags 和 reserved 字段
  4. drm_atomic_state_alloc()
  5. 初始化 drm_modeset_acquire_ctx
  6. 遍历 userspace 的 object ID
  7. 查找 drm_mode_object
  8. 遍历 property ID/value
  9. drm_atomic_set_property()
 10. 准备 input fence / output fence / event
 11. TEST_ONLY -> drm_atomic_check_only()
     NONBLOCK   -> drm_atomic_nonblocking_commit()
     普通提交   -> drm_atomic_commit()
```

源码中 `drm_mode_atomic_ioctl()` 在 `1421-1427` 根据 flags 选择三种行为：

- `TEST_ONLY`：只验证，不改变硬件和当前状态，适合 compositor 试算。
- `NONBLOCK`：提交后尽快返回，把硬件提交放到异步工作中。
- 普通 commit：通常等待到提交流程可同步完成后返回。

### Atomic state 是什么
`struct drm_atomic_state` 是一次更新的事务上下文。它包含本次涉及的：

- `drm_crtc_state`；
- `drm_plane_state`；
- `drm_connector_state`；
- modeset acquire context；
- fence、commit completion 和旧状态引用。

Atomic property 并不是直接写硬件寄存器。用户态传入的是 object/property/value，内核先把它们转换成新的对象状态，完成全局检查后才调用驱动。

因此：

```text
SET_PROPERTY != 立即写寄存器
SET_PROPERTY = 修改本次 atomic transaction 的 new_state
COMMIT       = 检查并将 new_state 推进到硬件
```

## 四、Atomic 检查链路：先证明配置可行

`drivers/gpu/drm/drm_atomic.c:1326` 的 `drm_atomic_check_only()` 是通用检查入口。

大致顺序：

```text
drm_atomic_check_only()
  -> drm_atomic_plane_check()
  -> drm_atomic_crtc_check()
  -> drm_atomic_connector_check()
  -> mode_config.funcs->atomic_check()
      -> driver private global check
      -> crtc helper atomic_check
      -> plane helper atomic_check
      -> encoder/connector atomic_check
  -> 检查是否需要 modeset
  -> 检查 ALLOW_MODESET
```

`drm_atomic_helper_check()` 会进一步调用各类 helper。> [!info] 典型 driver atomic_check 回调
> | 回调 | 负责检查 |
> |---|---|
> | `drm_plane_helper_funcs.atomic_check` | 格式、缩放、坐标、带宽、旋转能力 |
> | `drm_crtc_helper_funcs.atomic_check` | 时钟、时序、CRTC 全局资源 |
> | `drm_encoder_helper_funcs.atomic_check` | 编码器约束 |
> | `drm_connector_helper_funcs.atomic_check` | 连接器和链路能力 |
> | `drm_mode_config_funcs.atomic_check` | 跨多个 CRTC/plane 的全局资源分配 |

### 为什么检查必须在提交前完成
提交一旦通过 `swap_state()`，软件对象的 current state 就会改变；之后的硬件提交阶段原则上不能再返回普通配置错误。因此硬件能力验证必须尽量放在 `atomic_check`，而不能拖到 `atomic_update` 才发现错误。

### `-EDEADLK` 与 modeset acquire context
Atomic 检查可能需要按动态顺序获取多个 modeset lock。如果发生锁依赖冲突，返回 `-EDEADLK`，调用者执行：

```text
drm_atomic_state_clear()
  -> drm_modeset_backoff()
  -> 重新获取锁
  -> retry
```

这不是硬件失败，而是 DRM 的 wound/wait 锁排序重试机制。> [!tip] 调试 atomic 提交失败时，先区分错误码
> | 错误码 | 含义 |
> |---|---|
> | `-EINVAL` | 属性/格式/能力/时序不合法 |
> | `-EBUSY` | 资源或状态冲突 |
> | `-EDEADLK` | 需要按 DRM 规则 backoff 重试 |
> | `-ENOMEM` | 状态或内存分配失败 |

## 五、Atomic 提交链路：状态如何落到硬件

### 通用 commit
`drivers/gpu/drm/drm_atomic.c:1437`：

```text
drm_atomic_commit(state)
  -> drm_atomic_check_only(state)
  -> dev->mode_config.funcs->atomic_commit(dev, state, false)
```

很多驱动把 `mode_config.funcs->atomic_commit` 设置为 `drm_atomic_helper_commit`。

### helper commit 的阶段
`drivers/gpu/drm/drm_atomic_helper.c:1976`：

```text
drm_atomic_helper_commit()
  -> setup_commit()
  -> prepare_planes()
  -> 非 NONBLOCK 时等待输入 fences
  -> swap_state(state, true)
  -> NONBLOCK: queue_work(system_unbound_wq, commit_work)
  -> blocking: commit_tail(state)
```

关键点：`swap_state()` 是软件状态的切换点。其后提交阶段通常被视为“不能再因为普通参数问题失败”。

### commit_tail 的默认顺序
`drivers/gpu/drm/drm_atomic_helper.c:1714`：

```text
drm_atomic_helper_commit_tail(old_state)
  -> commit_modeset_disables()
  -> commit_planes()
      -> atomic_begin()
      -> plane atomic_disable / atomic_update
      -> atomic_flush()
  -> commit_modeset_enables()
      -> CRTC atomic_enable
      -> encoder atomic_enable
      -> bridge/connector enable
  -> commit_hw_done()
  -> 后续等待 flip_done
  -> cleanup_planes()
  -> cleanup_done()
```

> [!note] 什么情况下会覆盖默认 commit_tail 顺序
> - 需要 runtime PM 的平台；
> - 需要等待硬件 shadow register 生效的平台；
> - 多个 CRTC 共享带宽/时钟资源的平台；
> - 提交必须通过固件、命令队列或专用线程的平台。

### Plane 更新真正进入驱动的位置
`drm_atomic_helper_commit_planes()` 会依次触发：

- `atomic_begin()`：开始一个 CRTC/控制器批量更新；
- `atomic_disable()`：关闭旧 plane；
- `atomic_update()`：写入新 plane 的地址、stride、格式、位置、缩放等；
- `atomic_flush()`：提交/触发 shadow register 更新。

`atomic_update()` 的典型工作不是搬运像素，而是配置硬件 DMA scanout：

```text
fb->obj / private BO
  -> 取得 DMA/IOMMU 地址
  -> 写 plane base address
  -> 写 pitch/format/modifier
  -> 写 source/destination rectangle
  -> 写 alpha/zpos/blend
```

真正的像素搬运通常在显示控制器每一行扫描时由硬件 DMA 完成。

## 六、数据链路一：GEM buffer object

GEM 是 DRM 中常见的 buffer object 管理接口。用户态拿到的 `handle` 通常是每个 DRM file 私有的整数句柄，不是物理地址，也不是稳定的全局对象 ID。

常见流程：

```text
CREATE_DUMB / driver GEM create ioctl
  -> 分配 drm_gem_object 或 driver private BO
  -> drm_gem_handle_create()
  -> 返回 userspace handle

mmap / PRIME / FB create / atomic property
  -> 通过 handle 查找 GEM object
```

> [!info] 三个容易混淆的概念
> | 概念 | 含义 |
> |---|---|
> | GEM object | 内核对象，持有大小、引用计数、操作函数和 driver private data |
> | GEM handle | 绑定到某个 `drm_file` 的用户态引用 |
> | DMA/IOMMU 地址 | 硬件实际用于 DMA 的地址，通常不会直接暴露给用户态 |

在驱动中常见的对象关系是：

```text
struct drm_gem_object
        embedded in / referenced by
struct driver_bo
        -> DMA address / SG table / VRAM placement / GPU metadata
```

### GEM 生命周期知识点
- 创建 GEM handle 后，file 关闭时 DRM 会释放该 file 上的 handle。
- framebuffer、plane state、fence 和硬件使用期间都必须持有正确引用。
- `drm_gem_object_put()` 不是简单释放内存，而是减少引用，最后引用释放时才调用 driver free path。
- `prepare_fb` 往往负责 pin、迁移、建立 mapping 或提取 implicit fence。
- `cleanup_fb` 必须与 `prepare_fb` 成对，尤其涉及 VRAM pin、IOMMU map、DMA-BUF attachment 时。

## 七、数据链路二：DMA-BUF/PRIME 跨子系统共享

DMA-BUF 用一个 Linux 文件描述符在不同设备/子系统之间共享 buffer。典型场景：

```text
GPU 渲染器
  -> export dma-buf fd
  -> compositor/importer
  -> DRM PRIME_FD_TO_HANDLE
  -> drm_gem_prime_fd_to_handle()
  -> driver gem_prime_import()
  -> attachment + scatterlist/IOMMU mapping
  -> GEM-like object
  -> framebuffer
```

Linux 6.1 中 `drivers/gpu/drm/drm_prime.c:300` 的 `drm_gem_prime_fd_to_handle()` 重点做了：

1. `dma_buf_get(prime_fd)` 获取 DMA-BUF 引用；
2. 查找当前 file 是否已经导入过该 buffer；
3. 未导入时调用 driver 的 `gem_prime_import`，否则走通用 import；
4. 建立 GEM handle；
5. 在 PRIME handle 表中缓存映射关系。

> [!warning] 导入不等于可以 scanout
> 导入后仍可能需要：
> - `dma_buf_attach()`；
> - `dma_buf_map_attachment()` 得到 scatter-gather table；
> - IOMMU 映射；
> - cache coherency 同步；
> - 等待 producer 写入完成的 dma-fence；
> - 检查 buffer 的格式、modifier、stride 是否适合显示硬件。

### 隐式 fence 与显式 fence
- 隐式同步：fence 隐藏在 dma-buf reservation object 中，由 importer/driver 自动等待。
- 显式同步：用户态通过 in-fence/out-fence 或 syncobj 传递同步关系。

显示提交前必须确保显示控制器不会读取仍在被 GPU/视频解码器写入的数据。否则会出现撕裂、花屏或偶发旧帧。

## 八、数据链路三：Framebuffer 创建

Framebuffer UAPI 主要是：

- `DRM_IOCTL_MODE_ADDFB`；
- `DRM_IOCTL_MODE_ADDFB2`。

注册位置在 `drivers/gpu/drm/drm_ioctl.c:678-679`，实现位于 `drm_framebuffer.c`。

`ADDFB2` 的输入包含：

- width/height；
- pixel format；
- 每个 plane 的 handle；
- pitch；
- offset；
- modifier；
- flags。

`drm_mode_addfb2()` 会调用 `dev->mode_config.funcs->fb_create()`。GEM 驱动经常使用 `drm_gem_fb_create()`。

```text
GEM handle(s)
  + format
  + pitch
  + offset
  + modifier
        |
        v
fb_create()
        |
        v
struct drm_framebuffer
        |
        v
返回 framebuffer ID
```

### 多平面格式
以 NV12 为例：

```text
plane 0: Y
plane 1: UV
```

所以一个 framebuffer 可以引用多个 GEM handle，也可以在一个共享对象中通过不同 offset/pitch 描述多个平面。驱动必须验证：

> [!info] 多平面格式校验清单
> - 每个 plane 的尺寸和对齐；
> - offset + size 不越界；
> - pitch 满足硬件要求；
> - modifier 和格式组合有效；
> - 色彩布局与 plane 能力匹配。

### Framebuffer 引用关系
```text
userspace FB ID
  -> drm_framebuffer
  -> obj[] / driver private backing object
  -> plane_state.fb
  -> current hardware state
```

删除用户态 FB ID 不代表硬件立即停止使用对象。只要 plane state、commit、fence 或其他引用仍然持有对象，底层内存就不能释放。

## 九、vblank、page flip 和完成通知链路

控制链路提交新地址后，用户态还需要知道“什么时候真正生效”。这就是 vblank/page-flip 完成链路。

典型流程：

```text
atomic/page-flip commit
  -> 保存 drm_pending_vblank_event 或 drm_crtc_state.event
  -> 使能 vblank IRQ
  -> 硬件到达帧边界
  -> driver IRQ handler
  -> drm_crtc_handle_vblank(crtc)
     或 drm_handle_vblank(dev, pipe)
  -> 更新 vblank counter / 时间戳
  -> drm_crtc_send_vblank_event()
  -> 唤醒用户态 poll/read/ioctl 等待者
  -> 完成 flip_done / cleanup 旧 framebuffer
```

通用接口：

- `drm_crtc_handle_vblank()`：处理某个 CRTC 的 vblank。
- `drm_handle_vblank()`：按 pipe 处理 vblank。
- `drm_crtc_send_vblank_event()`：向用户态发送 page-flip/vblank event。
- `drm_crtc_vblank_get()` / `drm_crtc_vblank_put()`：管理 vblank 引用和中断开关。

### `hw_done`、`flip_done`、event 的区别
不要把几个完成点混为一谈：

> [!info] 几个完成点的区别
> | 完成点 | 含义 |
> |---|---|
> | `hw_done` | 驱动已经把提交动作送入硬件/控制器，不代表新帧已经扫描完成 |
> | `flip_done` | 硬件已经在合适的时刻完成切换，旧 framebuffer 可以按驱动规则释放 |
> | vblank/page-flip event | 面向用户态的可观察通知，通常在帧边界到达时发送 |
> | cleanup done | DRM 对旧 atomic state 和 plane 资源完成清理 |

不同硬件可以改变这些完成点的顺序，但不能错误地过早释放仍在 scanout 的 buffer。

### 中断处理要点
> [!note] IRQ handler 通常只做
> 1. 读取和清除硬件中断状态；
> 2. 判断是否为 vblank/frame-done；
> 3. 调用 DRM vblank/flip 完成接口；
> 4. 必要时推进硬件提交队列。

不能在硬中断中执行可能睡眠的操作。若需要等待、分配或复杂处理，应转移到 threaded IRQ、workqueue 或专用线程。

## 十、一次普通 atomic page flip 的完整时序

```text
用户态：
  1. GPU/解码器写入 buffer A
  2. 准备 framebuffer FB(A)
  3. 为 primary plane 设置 FB_ID=A
  4. 设置 CRTC/plane properties
  5. DRM_IOCTL_MODE_ATOMIC

内核 DRM Core：
  6. drm_ioctl() 分发
  7. drm_mode_atomic_ioctl() 解析 object/property/value
  8. 构造 drm_atomic_state
  9. 等待或记录 in-fence
 10. drm_atomic_check_only()
 11. driver atomic_check() 验证格式、时序、带宽和资源
 12. drm_atomic_helper_commit()
 13. prepare_fb()：pin/map/同步
 14. swap_state()
 15. atomic_commit_tail()
 16. atomic_update() 写 plane 地址和属性
 17. atomic_flush() 触发 shadow register/latch

硬件：
 18. 在下一个安全帧边界应用新配置
 19. 显示 DMA 从 buffer A 读取像素
 20. CRTC 输出完整帧

完成链路：
 21. vblank IRQ
 22. drm_crtc_handle_vblank()
 23. 发送 page-flip event / signal fence
 24. 清理旧 buffer 和旧 atomic state
```

如果硬件支持异步提交，步骤 15 以后可能在 `system_unbound_wq` 或 driver 私有线程中执行；用户态的 ioctl 返回时间不等于屏幕已经显示新帧的时间。

## 十一、结合源码阅读 DRM 驱动的方法

建议按以下顺序阅读一个具体 SoC 驱动：

1. 找 `struct drm_driver` 和 `struct drm_mode_config_funcs`。
2. 找 `fb_create`，确定 framebuffer 如何从 handle 转为 driver BO。
3. 找 `struct drm_plane_helper_funcs` 的 `atomic_check`、`atomic_update`、`prepare_fb`。
4. 找 `struct drm_crtc_helper_funcs` 的 `atomic_enable`、`atomic_flush`、`enable_vblank`。
5. 找 IRQ handler 中的 `drm_crtc_handle_vblank()` 或 `drm_handle_vblank()`。
6. 找 `drm_crtc_send_vblank_event()` 的调用位置。
7. 找 `atomic_commit_tail` 是否被平台覆盖。
8. 找 runtime PM、时钟、IOMMU、DMA-BUF attachment 和 fence 等外围条件。

适合入门对照的源码：

- DRM ioctl 总表：`drivers/gpu/drm/drm_ioctl.c`
- Atomic UAPI：`drivers/gpu/drm/drm_atomic_uapi.c`
- Atomic 状态检查：`drivers/gpu/drm/drm_atomic.c`
- Atomic helper 提交：`drivers/gpu/drm/drm_atomic_helper.c`
- Framebuffer：`drivers/gpu/drm/drm_framebuffer.c`
- GEM framebuffer helper：`drivers/gpu/drm/drm_gem_framebuffer_helper.c`
- GEM atomic helper：`drivers/gpu/drm/drm_gem_atomic_helper.c`
- PRIME/DMA-BUF：`drivers/gpu/drm/drm_prime.c`
- vblank：`drivers/gpu/drm/drm_vblank.c`
- 一个较直观的 KMS 驱动示例：`drivers/gpu/drm/atmel-hlcdc/`
- 复杂的多 plane/异步提交示例：`drivers/gpu/drm/vc4/`、`drivers/gpu/drm/rockchip/`

## 十二、常见故障与定位思路

### atomic_check 返回 -EINVAL
重点检查：

- format/modifier 与 plane 能力不匹配；
- source/destination rectangle 越界；
- pitch、offset、对齐不满足硬件要求；
- CRTC mode 和 connector/encoder 链路不支持；
- 缩放比例超限；
- 多 plane 带宽超过硬件上限；
> - 没有 `ALLOW_MODESET` 却修改了需要 modeset 的属性。

### 花屏、黑屏、偶发旧帧
重点检查：

- DMA/IOMMU 地址是否正确；
- buffer 是否已经 pin/map；
- dma-buf attachment 的 SG table 是否映射到正确设备；
- producer fence 是否等待；
- cache 是否同步；
- format、stride、modifier、plane offset 是否一致；
- 硬件是否需要在 vblank 才 latch 新地址；
> - `atomic_flush` 是否真正触发提交。

### page-flip event 不返回
重点检查：

- vblank IRQ 是否使能；
- IRQ 状态是否被正确清除；
- 驱动是否调用 `drm_crtc_handle_vblank()`；
- 驱动是否调用 `drm_crtc_send_vblank_event()`；
- CRTC 是否已经 active；
- event 是否在错误的状态对象中保存；
> - modeset disable/enable 路径是否遗失 pending event。

### 释放后使用或模块退出崩溃
重点检查：

- `cleanup_fb` 是否早于硬件真正停止 scanout；
- atomic commit work 是否 flush/cancel；
- vblank IRQ 是否关闭并同步；
- dma-buf attachment 是否 detach；
> - GEM object、FB、plane state、fence 的引用是否平衡。

## 十三、必须掌握的知识点清单

> [!info] DRM 基础概念速查
> | 知识点 | 你需要掌握的核心理解 |
> |---|---|
> | DRM node | 用户态访问 DRM 驱动的设备文件，例如 `/dev/dri/cardN` |
> | DRM master | 获得显示控制权的进程；同一时间通常只有一个主要进程负责 KMS 控制 |
> | Render node | 只用于渲染和 buffer 操作，不负责显示模式设置 |
> | drm_ioctl_desc | ioctl 的路由和权限描述表项 |
> | ioctl flags | 描述 ioctl 的访问限制和行为，例如是否需要 master、是否允许 render node 调用 |

### Legacy KMS 与 Atomic KMS

> [!example] 两种 KMS 模型对比
> | 模型 | 工作方式 | 适合理解 |
> |---|---|---|
> | Legacy KMS | 逐个对象修改，并尽快生效 | 旧的、分散的显示配置接口 |
> | Atomic KMS | 先构造完整状态，统一检查，确认无误后一次性提交 | 现代 DRM 的事务化显示状态管理机制 |
>
> RK3576 的 VOP2 驱动应重点沿 `atomic_check -> atomic_update -> atomic_flush -> vblank` 这条路径学习。

### Atomic 状态对象

> [!info] Atomic state 三个核心概念
> | 概念 | 含义 |
> |---|---|
> | `drm_atomic_state` | 一次 atomic 提交的事务容器 |
> | old state | 当前正在使用的状态 |
> | new state | 用户态希望提交的新状态 |

### Atomic 提交 flags

> [!info] DRM_IOCTL_MODE_ATOMIC 常见 flags
> | flag | 作用 | 典型用途 |
> |---|---|---|
> | `DRM_MODE_ATOMIC_TEST_ONLY` | 只检查，不真正应用 | 试算这组配置是否可行 |
> | `DRM_MODE_ATOMIC_NONBLOCK` | 非阻塞提交，ioctl 尽快返回 | 提交异步化，不等硬件完成 |
> | `DRM_MODE_ATOMIC_ALLOW_MODESET` | 允许修改显示模式或显示链路 | 修改 mode、connector、CRTC 等需要 modeset 的配置 |

### `-EDEADLK` backoff 机制

> [!warning] 不要把 `-EDEADLK` 当成普通硬件错误
> 在 DRM Atomic 提交中，`-EDEADLK` 通常表示当前提交暂时无法获取所需的 modeset locks，可能发生锁顺序冲突。
>
> 正确处理方式是释放已获取的锁，执行 backoff，然后重新尝试 atomic 提交。它不一定表示系统真的出错。

### Plane、CRTC、Encoder、Bridge、Connector 职责边界

> [!info] DRM 显示对象职责表
> | 对象 | 主要职责 | 关键问题 |
> |---|---|---|
> | Plane | 提供一个图像图层 | 显示哪张 framebuffer、位置、缩放、格式、zpos、alpha |
> | CRTC | 合成图层并产生显示时序 | 按什么分辨率、刷新率、时序扫描输出 |
> | Encoder | 把 CRTC 的像素流转换成某种输出类型 | RGB、TMDS、DP 等编码方式 |
> | Bridge | 连接并转换显示链路中的中间设备 | DSI、HDMI、DP、LVDS、协议或信号转换 |
> | Connector | 表示最终物理显示端点 | 是否连接、支持哪些 mode、EDID 是什么 |

简单记法：

- Plane：显示什么图像、放在哪里。
- CRTC：如何合成，并按什么时序输出。
- Encoder：把像素流编码成什么输出形式。
- Bridge：经过哪些协议或硬件转换。
- Connector：最终接到哪个物理显示端。

### GEM object、GEM handle、DMA/IOMMU 地址

> [!info] Buffer 相关概念对比
> | 概念 | 含义 | 关键点 |
> |---|---|---|
> | GEM object | 内核中的 buffer 对象 | 可以理解为实际像素数据所在内存的内核对象，用户态一般不直接拿到指针 |
> | GEM handle | 用户态访问 GEM object 的编号 | 只在当前 DRM fd 内有效，不是物理地址，也不是全局唯一 ID |
> | DMA/IOMMU 地址 | 硬件访问 buffer 时使用的地址 | VOP2 最终读取的是驱动配置好的 DMA/IOMMU 地址 |

简单记法：handle 是编号，object 是内核对象，DMA 地址是硬件访问地址。

### Framebuffer、pitch、offset、format、modifier

Framebuffer 是像素布局描述，不是显存分配器。

> [!info] Scanout 相关字段
> | 字段 | 含义 | 对硬件 scanout 的影响 |
> |---|---|---|
> | pitch / stride | 一行数据实际占用的字节数 | 决定硬件读完一行后如何跳到下一行 |
> | offset | 某个图像 plane 在 GEM object 中的起始偏移 | 决定从 buffer 的哪里开始读 |
> | format | 像素格式，例如 `DRM_FORMAT_XRGB8888`、`ARGB8888` | 决定硬件如何解释内存中的字节 |
> | modifier | 像素在内存中的特殊布局 | 可能表示线性布局、压缩布局或厂商私有布局 |

例子：`format=XRGB8888`、`width=1920`、`height=1080`、`pitch[0]=7680`、`offset[0]=0`、`modifier=LINEAR`。

硬件大致会从 `DMA_BASE + offset[0]` 开始读，每 4 字节解释成 1 个 XRGB8888 像素；每读完一行 1920 个像素后，按 7680 字节跨度跳到下一行，直到读完 1080 行。

### DMA-BUF / PRIME export-import

> [!info] DMA-BUF / PRIME 核心理解
> | 概念 | 说明 |
> |---|---|
> | dma-buf fd | 可以跨进程、跨驱动传递的共享 buffer 文件描述符 |
> | export | 把本地 GEM handle 变成可共享的 dma-buf fd |
> | import | 把 dma-buf fd 变成另一个 DRM 设备中可用的 GEM handle |
> | 数据复制 | export/import 通常不复制像素数据，只共享同一块底层 buffer |
> | VOP2 视角 | VOP2 不认识 handle 或 fd，只读取驱动配置好的 DMA/IOMMU 地址 |

### dma-buf attachment、SG table、IOMMU

> [!info] DMA-BUF 被另一个设备使用时的三层关系
> | 层 | 作用 |
> |---|---|
> | attachment | 表示某个 importer device 想使用这个 dma-buf，建立“谁要访问我”的关系 |
> | SG table | 描述 buffer 实际由哪些物理页组成，是物理/DMA 分段清单 |
> | IOMMU 映射 | 把物理页映射到设备可访问的 DMA 地址空间 |

attachment 本身不是马上给设备地址，而是建立设备关系。因为不同 device 的 DMA 能力不同，所以 dma-buf 必须知道哪个设备要访问它。

### implicit fence 与 explicit fence

> [!info] Fence 同步模型
> | 类型 | 同步信息在哪里 | 用户态怎么用 |
> |---|---|---|
> | implicit fence | 同步信息藏在 dma-buf reservation object 中 | 应用只传 buffer，kernel / driver 自动从 dma-buf 中找到 fence 并等待 |
> | explicit fence | fence 作为独立对象显式传递 | 应用同时传 buffer 和 fence，buffer 与 fence 分离 |

fence 可以理解成一个异步完成标记，用来说明这个 buffer 什么时候可以被下一个设备或队列安全使用。

### 函数调用阶段

> [!info] Atomic helper 关键函数阶段
> | 函数 | 发生阶段 | 主要作用 |
> |---|---|---|
> | `prepare_fb` | commit 前、plane 更新前 | pin / map buffer，处理 fence 或 DMA-BUF attachment |
> | `atomic_check` | commit 前检查阶段 | 完成格式、带宽、缩放、时序等硬件能力验证 |
> | `swap_state` | 软件状态切换点 | 将 new state 推进为当前状态，是重要边界 |
> | `atomic_update` | 硬件更新阶段 | 写 plane 地址、stride、format、位置、缩放等寄存器 |
> | `atomic_flush` | 批量更新末尾 | 触发 shadow register 或硬件 latch 生效 |
> | `cleanup_fb` | commit 完成清理阶段 | 释放 prepare_fb 中持有的资源 |

### 完成点和释放时机

> [!warning] 不要过早释放 framebuffer
> | 完成点 | 含义 |
> |---|---|
> | hw_done | 驱动已把提交动作送入硬件，不代表新帧已经显示完成 |
> | flip_done | 硬件已经完成翻页，旧 framebuffer 通常可以按规则释放 |
> | vblank event | 面向用户态的帧边界通知 |
> | cleanup done | DRM 对旧 atomic state 和 plane 资源完成清理 |
>
> 释放 framebuffer 前必须确认硬件不再读取 backing buffer，否则容易出现花屏、黑屏或 use-after-free 类问题。

## 十四、推荐的动态调试手段

在具备权限和相应内核配置时，可以使用：

```bash
## 查看 DRM 设备和节点
ls -l /dev/dri

## 查看 KMS 对象、connector、CRTC、plane 和属性
cat /sys/kernel/debug/dri/0/state

## 查看 DRM 信息
cat /sys/kernel/debug/dri/0/name

## 查看内核日志中的 DRM/KMS 信息
dmesg | rg -i 'drm|kms|vblank|atomic|plane|crtc|dma'
```

如果内核启用了 DRM debug，可以关注：

```text
drm.debug=0x1ff
```

生产系统不建议长期打开全量 debug。更实用的方式是先确认：

```text
用户态提交的 property
  -> atomic state 中的 new state
  -> driver atomic_check 是否通过
  -> prepare_fb 得到的 DMA 地址
  -> atomic_update 是否写寄存器
  -> vblank IRQ 是否到达
  -> event/fence 是否完成
```

这条检查路径可以快速区分“控制参数错误”“内存映射错误”“硬件提交错误”和“完成通知错误”。

## 十五、RK3576 从零学习路线

你的目标平台是 Rockchip RK3576，建议以 Rockchip DRM 主显示控制器和 VOP2 为主线，同时先掌握 DRM Core 的通用框架。RK3576 的实际显示链路还会受到具体板级设备树、MIPI-DSI/HDMI/DP 输出、桥接芯片、面板以及 Android 图形栈配置的影响，因此源码分析时要把“通用 DRM”与“平台驱动”分开定位。

### 第一阶段：先建立对象模型
先理解这些对象之间的关系：

```text
GEM/分配器 buffer
  -> dma-buf（跨模块共享，可选）
  -> drm_framebuffer
  -> drm_plane
  -> drm_crtc
  -> encoder/bridge
  -> connector
  -> panel 或 HDMI/DP 显示器
```

建议先掌握：

1. `/dev/dri/cardN` 与 `/dev/dri/renderDN` 的区别。
2. GEM object、GEM handle、dma-buf fd、framebuffer ID 的区别。
3. plane、CRTC、connector、encoder、bridge 的职责边界。
4. pixel format、pitch、offset、modifier 对显示的影响。
5. vblank、page flip、fence 和 atomic commit 完成事件的区别。

### 第二阶段：沿 RK3576 的 KMS 初始化阅读
建议在内核源码中按以下顺序定位：

```text
Rockchip DRM platform driver
  -> rockchip_drm_bind()
  -> component bind
  -> VOP/VOP2 注册
  -> drm_mode_config_init()
  -> CRTC / plane 创建
  -> encoder / connector / bridge / panel 连接
  -> drm_vblank_init()
  -> drm_dev_register()
```

重点目录：

- `drivers/gpu/drm/rockchip/rockchip_drm_drv.c`
- `drivers/gpu/drm/rockchip/rockchip_drm_vop2.c`
- `drivers/gpu/drm/rockchip/rockchip_drm_vop.c`
- `drivers/gpu/drm/rockchip/rockchip_drm_fb.c`
- `drivers/gpu/drm/rockchip/rockchip_drm_gem.c`
- `drivers/gpu/drm/rockchip/rockchip_drm_vop2.h`
- `drivers/gpu/drm/rockchip/rockchip_rgb.c`
- `drivers/gpu/drm/rockchip/rockchip_lvds.c`
- `drivers/gpu/drm/rockchip/dw-mipi-dsi-rockchip.c`
- `drivers/gpu/drm/bridge/` 下实际使用的 DesignWare/HDMI/DP bridge

源码分析时，优先确认 RK3576 使用的是 VOP 还是 VOP2 代码路径，以及设备树中实际启用了哪些 `route_*`、VOP、DSI、HDMI、DP 和 panel 节点。

### 第三阶段：学习一帧图像的控制链路
以 Android compositor 或 Linux DRM 用户态提交一帧为例，按下面路径追踪：

```text
Surface buffer
  -> dma-buf fd
  -> PRIME_FD_TO_HANDLE 或平台图形缓冲导入
  -> ADDFB2 / drm_gem_fb_create()
  -> ATOMIC 设置 PLANE_FB_ID
  -> 设置 CRTC、SRC/CRTC 坐标、格式、zpos 等属性
  -> drm_mode_atomic_ioctl()
  -> rockchip atomic_check()
  -> drm_atomic_helper_commit()
  -> VOP2 plane atomic_update()
  -> 写 VOP2 WIN/VP 寄存器
  -> atomic_flush() 触发配置生效
  -> vblank IRQ
  -> drm_crtc_handle_vblank()
  -> drm_crtc_send_vblank_event()
```

在 RK3576 上，`VOP2 plane atomic_update()` 是理解“抽象 framebuffer 如何变成硬件窗口配置”的关键。重点关注：

- framebuffer 对象如何转成 GEM/DMA 地址；
- YUV、多平面格式如何计算每个 plane 的地址；
- stride、offset、格式和 modifier 如何写入窗口寄存器；
- source rectangle 到 destination rectangle 的缩放配置；
- alpha、zpos、blend、遮罩和窗口使能；
- 带宽、时钟和跨 VP 资源检查在哪里完成。

### 第四阶段：分别深入三条链路
#### 控制链路

```text
libdrm / Android HWC
  -> ioctl
  -> drm_ioctl
  -> drm_mode_atomic_ioctl
  -> drm_atomic_state
  -> atomic_check
  -> atomic_commit
  -> rockchip atomic_commit_tail 或 helper
  -> VOP2 atomic_update/flush
```

目标是回答：用户态设置的每一个 property，最后由哪个 driver 回调消费，写入哪个硬件寄存器。

#### 数据链路

```text
GPU/解码器 buffer
  -> dma-buf
  -> Rockchip GEM/allocator import
  -> sg_table 或 IOMMU/DMA 地址
  -> drm_framebuffer
  -> plane state
  -> VOP2 window address/stride/format
```

目标是回答：像素数据在哪里分配、如何共享、如何映射、如何保证显示硬件读取时地址和 cache 状态正确。

#### 完成链路

```text
VOP2 frame-start/frame-end IRQ
  -> Rockchip IRQ handler
  -> drm_crtc_handle_vblank()
  -> pending event / flip_done
  -> drm_crtc_send_vblank_event()
  -> drm_read()/poll()/fence
  -> 释放旧 framebuffer 引用
```

目标是回答：一次提交何时算“送入硬件”、何时算“真正翻页”、何时可以释放旧 buffer。

### 推荐的学习顺序
1. 先读本文第 1-5 节，建立 DRM/KMS 和 Atomic 对象模型。
2. 再读 `drm_ioctl.c`、`drm_atomic_uapi.c` 和 `drm_atomic_helper.c`，跟通用控制链路。
3. 再读 `drm_gem.c`、`drm_prime.c`、`drm_framebuffer.c`，跟数据对象生命周期。
4. 再读 Rockchip 的 `rockchip_drm_drv.c`，明确设备初始化和组件绑定。
5. 再读 `rockchip_drm_vop2.c`，重点看 plane、CRTC、IRQ 和 atomic 回调。
6. 最后根据 RK3576 板子的设备树，沿实际输出链路读 DSI/HDMI/DP/bridge/panel。
7. 使用 debugfs 的 `/sys/kernel/debug/dri/0/state` 对照源码中的 object state。

这样学习可以形成一个闭环：

```text
用户态 property
  -> DRM atomic state
  -> Rockchip check
  -> VOP2 register update
  -> display controller scanout
  -> IRQ/vblank event
  -> 用户态下一帧
```

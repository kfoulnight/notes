---
title: Linux Input 子系统深度分析
cssclasses:
  - archive-page
---

# Linux Input 子系统深度分析

本文基于 Linux 3.18.41（SiRF Atlas7 BSP）的内核源码，分析 input 子系统的核心机制：中断上下文与工作队列上报、evdev 事件环形缓冲、阻塞读取、SYN_REPORT 帧同步和完整调用链。

> [!summary] 核心主线
> input 子系统通过 `event_lock` 保护事件路径，把驱动上报的事件按帧累积，在 `SYN_REPORT` 时批量分发到 evdev 环形缓冲；用户空间只在存在完整帧时才能读到事件。理解“帧边界”和“锁序”是掌握整个子系统的关键。

> [!info] 分析环境
> | 项目 | 内容 |
> |---|---|
> | 内核版本 | Linux 3.18.41（SiRF Atlas7 BSP） |
> | 核心文件 | `drivers/input/input.c` |
> | 核心文件 | `drivers/input/evdev.c` |
> | 核心头文件 | `include/linux/input.h` |

## 一、中断上下文与工作队列上报

### 为什么 `input_event()` 必须使用 `spin_lock_irqsave`

```c
// input.c:427-438
void input_event(struct input_dev *dev, unsigned int type, unsigned int code, int value)
{
    unsigned long flags;
    if (is_event_supported(type, dev->evbit, EV_MAX)) {
        spin_lock_irqsave(&dev->event_lock, flags);  // ★ 关中断+自旋锁
        input_handle_event(dev, type, code, value);
        spin_unlock_irqrestore(&dev->event_lock, flags);
    }
}
```

> [!info] 锁变体对比
> | 锁变体 | 硬中断 | 软中断/timer | 进程上下文 | 结论 |
> |---|---|---|---|---|
> | `spin_lock` | 同 CPU 自死锁 | 同 CPU 自死锁 | 可用 | 不安全 |
> | `spin_lock_irq` | 可用 | 错误恢复 IRQ 状态 | 可用 | 不安全 |
> | `spin_lock_irqsave` | 可用 | 可用 | 可用 | 唯一通用解 |

Input 子系统**设计为可从所有上下文调用**，`_irqsave` 是必然选择。

### 四种上报模式

> [!example] 上报模式对比
> | 模式 | 驱动 | 上报上下文 | 特点 |
> |---|---|---|---|
> | A | gpio_keys | 硬中断直接 | 唯一在硬中断直接上报的 BSP 驱动 |
> | B | gt9xx | 硬中断 + kworker | 推荐模式，硬中断只做 disable 和 queue_work |
> | C | sirfsoc_ts、ft5x06 | irq thread | `IRQF_ONESHOT`，可阻塞读取 |
> | D | atlas7-keypad | 轮询 kworker | 无中断，按周期自调度

#### 模式 A: 硬中断直接上报 (gpio_keys)

```
gpio_keys_irq_isr()                          ← 硬中断上下文
  ├── spin_lock_irqsave(&bdata->lock)         ← 驱动自身锁
  ├── input_event(EV_KEY, code, 1)            ← 直接在硬中断中调用
  ├── input_sync()
  ├── input_event(EV_KEY, code, 0)            ← 无消抖时释放也在这里
  ├── input_sync()
  └── spin_unlock_irqrestore(&bdata->lock)
```

**唯一在硬中断中直接上报的BSP驱动**，额外用 `bdata->lock` 保护按键按下/释放原子性。

#### 模式 B: 硬中断 → 工作队列 (gt9xx) — **推荐模式**

```
goodix_ts_irq_handler()                       ← 硬中断, 仅4行
  ├── gtp_irq_disable(ts)                     ← 屏蔽中断
  └── queue_work(goodix_wq, &ts->work)        ← 丢给专用工作队列

goodix_ts_work_func()                         ← kworker 上下文
  ├── gtp_i2c_read()                          ← 阻塞I2C (硬中断中非法!)
  ├── input_report_abs(ABS_MT_POSITION_X, x)
  ├── input_report_abs(ABS_MT_POSITION_Y, y)
  ├── input_mt_sync()
  ├── input_sync()
  └── gtp_irq_enable(ts)                      ← 重新使能中断
```

**硬中断只做 disable+queue_work，耗时I2C+上报全在kworker中。**

#### 模式 C: 纯线程化中断 (sirfsoc_ts, ft5x06)

```
devm_request_threaded_irq(irq,
    NULL,                                      ← 硬中断handler=NULL
    sirfsoc_ts_thread_irq,                     ← 全在irq thread中
    IRQF_ONESHOT, ...)

sirfsoc_ts_thread_irq()                        ← irq thread 上下文
  ├── do {
  │     ├── i2c_read()                         ← 可阻塞
  │     ├── input_report_abs(...)
  │     ├── input_mt_sync()
  │     ├── input_sync()
  │     └── usleep_range(5000, 8000)           ← 可睡眠!
  │   } while (press_down);
  └── input_sync() 释放帧
```

**IRQF_ONESHOT** 保证thread执行期间中断线屏蔽，无需驱动自身锁。

#### 模式 D: 纯轮询 (atlas7-keypad, input_polled_dev)

```
input_polled_device_work()                     ← system_freezable_wq
  ├── dev->poll(dev)                           ← 驱动轮询回调
  │     ├── iio_read_channel_processed()       ← 阻塞读ADC
  │     ├── input_report_key(...)
  │     └── input_sync()
  └── queue_delayed_work(wq, &dev->work, 30ms) ← 自调度, 30ms周期
```

`input_open_device()` 启动轮询, `input_close_device()` 取消。默认间隔500ms，sysfs可调。

> [!info] 驱动上报方式汇总
> | 驱动 | 上报上下文 | 自身锁 | 中断类型 |
> |---|---|---|---|
> | sirfsoc_ts | irq thread | 无 | `IRQF_ONESHOT`，hard 为 NULL |
> | ft5x06 | irq thread | 无 | `IRQF_ONESHOT`，hard 为 NULL |
> | gt9xx | kworker（专用 wq） | `irq_lock`（仅 IRQ 状态） | `request_irq` 硬中断后 queue_work |
> | gpio_keys | 硬中断直接 | `bdata->lock` | `request_any_context_irq` |
> | atlas7-keypad | kworker（专用 wq） | 无 | 无 IRQ，纯自调度轮询 |

---

## 二、evdev 事件环形缓冲

### 核心数据结构

```c
// evdev.c:45-60
struct evdev_client {
    unsigned int head;          // 写指针
    unsigned int tail;          // 读指针
    unsigned int packet_head;   // ★ 帧边界指针: 下一个完整帧起始位置
    spinlock_t buffer_lock;     // 保护 buffer + head/tail
    unsigned int bufsize;       // 始终是2的幂
    struct input_event buffer[];// 柔性数组环形缓冲
};
```

```
环形缓冲示意 (bufsize=8):
        tail          packet_head   head
         ↓               ↓           ↓
   ┌───┬───┬───┬───┬───┬───┬───┬───┐
   │ A │ B │ C │ D │ E │   │   │   │
   └───┴───┴───┴───┴───┴───┴───┴───┘
         ← 可读(完整帧) →← 不可读  →
```

**`packet_head` 是核心**: 只有 `SYN_REPORT` 写入时才更新 `packet_head = head`。读端只暴露 `[tail, packet_head)`——不完整帧对用户空间**不可见**。

### 缓冲大小计算

```c
// evdev.c:15-16
#define EVDEV_MIN_BUFFER_SIZE   64U     // 至少64个事件
#define EVDEV_BUF_PACKETS       8       // 缓存8个完整帧

// evdev.c:390-397
bufsize = roundup_pow_of_two(
    max(hint_events_per_packet * 8, 64)
);
```

示例: 触摸屏每帧5事件 → `roundup_pow_of_two(40→64)` = 64槽 = ~12帧缓冲。

### 写入路径

```c
// evdev.c:204-226 — handler->events 回调
evdev_events(handle, vals, count)
  ├── time_mono = ktime_get()           // ★ 整帧共享一个时间戳
  ├── time_real = ktime_mono_to_real(time_mono)
  ├── rcu_read_lock()
  ├── if (grab) → 只发给grab客户端
  └── else → 遍历所有客户端:
        evdev_pass_values(client, vals, count, mono, real)
          ├── spin_lock(&client->buffer_lock)  // IRQ已关, 普通spin_lock即可
          ├── for each value:
          │     __pass_event(client, &event)
          │       ├── buffer[head++] = event
          │       ├── if (head==tail):         // ★ 缓冲区满 → SYN_DROPPED
          │       │     tail = head-2
          │       │     buffer[tail] = {SYN, SYN_DROPPED, 0}
          │       │     packet_head = tail
          │       └── if (SYN_REPORT):
          │             packet_head = head     // ★ 关闭当前帧
          ├── spin_unlock(&client->buffer_lock)
          └── if (SYN_REPORT):
                wake_up_interruptible(&evdev->wait)  // ★ 只在帧边界唤醒
```

**关键设计**:
- **一帧一时间戳**: `ktime_get()` 在批次入口调用一次
- **批量写入, 单次唤醒**: 只在SYN_REPORT后唤醒
- **每客户端独立buf**: 快读者不被慢读者阻塞

### 缓冲区溢出与 SYN_DROPPED

```
溢出前: [E1][E2][E3][SYN][E4][E5][E6][满]
                                 ↑tail  ↑head

溢出后: [DROPPED][E_new][ ][ ][ ][ ][ ][ ]
         ↑tail       ↑head
         packet_head
```

tail回退2格，只保留SYN_DROPPED+最新事件。用户空间读到SYN_DROPPED后**必须**通过 `EVIOCGABS`/`EVIOCGKEY` ioctl重新同步设备状态。

---

## 三、阻塞读取机制

### 等待条件

```c
// evdev.c:533-536
error = wait_event_interruptible(evdev->wait,
    client->packet_head != client->tail ||   // ★ 有完整帧
    !evdev->exist || client->revoked);       // 设备消失
```

**所有打开同一设备的客户端共享 `evdev->wait`**。

### 完整读取流程

```
evdev_read(file, buf, count)                         [evdev.c:494]
  │
  ├── count < sizeof(input_event) → -EINVAL          // 不允许半事件读
  │
  ├── O_NONBLOCK && packet_head==tail → -EAGAIN      // 无完整帧, 非阻塞返回
  │
  ├── count==0 → break                                // 纯错误检查(设备存活?)
  │
  ├── while (用户buf有空间 && evdev_fetch_next_event())
  │     ├── evdev_fetch_next_event()                  [evdev.c:473]
  │     │     ├── spin_lock_irq(&buffer_lock)
  │     │     ├── have = (packet_head != tail)        // ★ 只读完整帧
  │     │     ├── *event = buffer[tail++]
  │     │     └── spin_unlock_irq(&buffer_lock)
  │     └── copy_to_user(event) → 累加 read
  │
  ├── if (read > 0) break;                            // ★ 读了就返回, 不填满buf
  │
  └── wait_event_interruptible(evdev->wait,           // ★ 阻塞等待
        packet_head != tail || !exist || revoked)
```

**核心行为**:
- **不以填满用户buf为目标** — 至少读到一个帧就返回
- **不完整帧绝不暴露** — `packet_head != tail` 是所有读取的前提条件
- 用户空间应循环调用直到读完所有可用数据

### Poll 机制

```c
// evdev.c:546-563
evdev_poll(file, wait)
  ├── poll_wait(file, &evdev->wait, wait)   // 注册到设备等待队列
  ├── if (packet_head != tail)              // ★ 有完整帧
  │     mask |= POLLIN | POLLRDNORM
  └── if (!exist || revoked)
        mask |= POLLHUP | POLLERR
```

### 异步通知（SIGIO）

```c
// evdev.c:164 — __pass_event() 中
if (event->type == EV_SYN && event->code == SYN_REPORT) {
    kill_fasync(&client->fasync, SIGIO, POLL_IN);  // ★ 帧边界发SIGIO
}
```

通过 `fcntl(fd, F_SETOWN, pid)` + `fcntl(fd, F_SETFL, O_ASYNC)` 启用。

---

## 四、SYN_REPORT 帧同步机制

### 帧构建：累积后批量分发

```
驱动调用:
  input_report_abs(dev, ABS_X, 100)
  input_report_key(dev, BTN_TOUCH, 1)
  input_sync(dev)                      ← 帧边界

input_handle_event() 内部处理:
  │
  ├── input_get_disposition()          ← 事件分类
  │     ├── EV_SYN/SYN_REPORT  → INPUT_PASS_TO_HANDLERS | INPUT_FLUSH
  │     ├── EV_SYN/SYN_MT_REPORT → INPUT_PASS_TO_HANDLERS  (不flush!)
  │     ├── EV_ABS              → 去抖, 过滤重复值
  │     ├── EV_KEY              → 检查状态变化, 更新key[]
  │     └── EV_LED/EV_SND       → INPUT_PASS_TO_ALL (也发给设备)
  │
  ├── if (PASS_TO_DEVICE)
  │     dev->event()              ← LED/声音/力反馈回传设备
  │
  ├── if (PASS_TO_HANDLERS)
  │     dev->vals[num_vals++] = v ← 累积到帧缓冲
  │
  └── if (INPUT_FLUSH)            ← SYN_REPORT触发
        input_pass_values(dev, vals, num_vals)  ← 一次性批量分发
        num_vals = 0
```

### 时间线

```
t1: input_report_abs(X,100)  → vals[0] = {ABS, X, 100}       (累积)
t2: input_report_abs(Y,200)  → vals[1] = {ABS, Y, 200}       (累积)
t3: input_report_key(BTN,1)  → vals[2] = {KEY, BTN_TOUCH, 1} (累积)
t4: input_sync()             → vals[3] = {SYN, REPORT, 0}    (累积+FLUSH!)
                             → input_pass_values(vals, 4)
                             → evdev_events(handle, vals, 4) 一次调用
                             → wake_up 唤醒所有读者
```

### 溢出保护：自动插入 SYN_REPORT

```c
// input.c:402-406
if (dev->num_vals >= dev->max_vals - 2) {
    dev->vals[dev->num_vals++] = input_value_sync;  // ★ 自动SYN_REPORT
    input_pass_values(dev, dev->vals, dev->num_vals);
    dev->num_vals = 0;
}
```

`max_vals = hint_events_per_packet + 2`, 保留2格给自动SYN_REPORT+最后事件。即使驱动忘记调用 `input_sync()`, 内核也会在缓冲区满时强制分帧。

### 空帧抑制

```c
// input.c:398-401
if (disposition & INPUT_FLUSH) {
    if (dev->num_vals >= 2)    // SYN_REPORT + 至少1个数据事件
        input_pass_values(...);
    dev->num_vals = 0;
}
```

连续的SYN_REPORT中间无数据 → 不产生空帧。

### MT 多点触摸帧结构

```
一次完整的2指触摸帧:
  ABS_MT_SLOT 0               ← 暂存(不发出)
  ABS_MT_TRACKING_ID 45       ← 触发SLOT flush → 发出 slot+id
  ABS_MT_POSITION_X 1200
  ABS_MT_POSITION_Y 3400
  SYN_MT_REPORT               ← 关闭触点0 (入队, 不flush)
  ABS_MT_SLOT 1
  ABS_MT_TRACKING_ID 46
  ABS_MT_POSITION_X 800
  ABS_MT_POSITION_Y 2000
  SYN_MT_REPORT               ← 关闭触点1
  SYN_REPORT                  ← ★ 关闭整个帧 → FLUSH一次性分发
```

**`ABS_MT_SLOT` 暂存机制**: 只有后续实际MT数据变化时才将slot号插入事件流, 允许驱动随意重复写入slot。

### SYN_DROPPED 产生场景

> [!warning] SYN_DROPPED 场景
> | 场景 | 位置 | 机制 |
> |---|---|---|
> | evdev 客户端环形缓冲满 | `evdev.c:143` | tail 回退 2 格，覆盖为 SYN_DROPPED 加最新事件 |
> | `EVIOCGKEY` 等 ioctl 失败 | `evdev.c:778` | 已 flush 的事件无法恢复，插入 SYN_DROPPED |
> | 驱动断开连接 | `input.c:687` | `input_dev_release_keys()` 模拟全键释放 |

### 帧生命周期

```
[驱动]                [input core]          [evdev]              [用户空间]
  input_report_abs ──→
  input_report_key ──→
  input_sync ────────→
                      spin_lock_irqsave
                      累积到 dev->vals[]
                      FLUSH
                      input_pass_values()
                         ┌────────────────→ evdev_events(vals,count)
                         │                  ktime_get() 整帧时间戳
                         │                  遍历客户端:
                         │                    __pass_event() ×N
                         │                    packet_head=head (边界)
                         │                    wake_up_interruptible() ──→
                         │                                             poll返回
                         │                                             read返回
                      spin_unlock_irqrestore
```

---

## 五、完整调用链全景图

```
┌──────────────────────────────────────────────────────────────────┐
│                      上报路径全景图                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [硬中断] gpio_keys_irq_isr          ← 直接调用 input_event       │
│       │                                                           │
│       └──→ input_event(dev, EV_KEY, code, value)                 │
│                 │                                                 │
│  [irq thread] sirfsoc_ts_thread_irq  ← threaded IRQ              │
│       │         ft5x0x_ts_interrupt                              │
│       └──→ input_report_abs() / input_sync()                     │
│                 │                                                 │
│  [硬中断] goodix_ts_irq_handler      ← 只做 queue_work            │
│       └──→ queue_work(wq, &work)                                 │
│                 │                                                 │
│  [kworker] goodix_ts_work_func        ← 实际I2C读+上报            │
│       └──→ input_report_abs() / input_sync()                     │
│                 │                                                 │
│  [kworker] input_polled_device_work   ← 轮询模式                  │
│       └──→ dev->poll() → input_report_*() / input_sync()         │
│                 │                                                 │
├──────────────────────────────────────────────────────────────────┤
│                 ▼                                                 │
│         input_event(dev, type, code, value)                      │
│           spin_lock_irqsave(&dev->event_lock)                    │
│                 │                                                 │
│         input_handle_event()                                     │
│           ├── input_get_disposition()  [分类/去抖/状态更新]       │
│           ├── dev->event()             [PASS_TO_DEVICE: LED等]    │
│           ├── dev->vals[] 累积         [PASS_TO_HANDLERS]         │
│           └── input_pass_values()      [INPUT_FLUSH: SYN_REPORT]  │
│                 │                                                 │
├──────────────────────────────────────────────────────────────────┤
│                 ▼                                                 │
│         input_pass_values()                                       │
│           rcu_read_lock()                                         │
│           ├── [grab] → input_to_handler(grab)                     │
│           └── [normal] → for_each open handle:                    │
│                 input_to_handler(handle)                          │
│                   ├── handler->filter()   [过滤]                  │
│                   └── handler->events()   [批量投递]              │
│           rcu_read_unlock()                                       │
│                 │                                                 │
├──────────────────────────────────────────────────────────────────┤
│                 ▼                                                 │
│         evdev_events(handle, vals, count)                        │
│           ktime_get()  ← 整帧时间戳                               │
│           for_each_client:                                        │
│             evdev_pass_values(client, vals)                      │
│               spin_lock(&buffer_lock)                            │
│               for_each v: __pass_event(&event)                   │
│                 buffer[head++] = event                           │
│                 if (SYN_REPORT) packet_head = head  ← 帧边界      │
│                 if (buffer full) → SYN_DROPPED      ← 溢出        │
│               spin_unlock(&buffer_lock)                          │
│               if (SYN_REPORT) wake_up_interruptible(wait)        │
│                 │                                                 │
├──────────────────────────────────────────────────────────────────┤
│                 ▼                                                 │
│         用户空间 read(/dev/input/eventX)                          │
│           wait_event_interruptible(packet_head != tail)          │
│           while: evdev_fetch_next_event()                        │
│             buffer[tail++] → copy_to_user                       │
│           返回读取字节数                                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 六、关键同步机制总结

> [!info] 锁与保护对象
> | 锁 | 保护对象 | 上下文 | 获取方式 |
> |---|---|---|---|
> | `dev->event_lock` | 事件路径：`vals[]`、`key[]`、`sw[]`、`pass_values()` | 所有上下文 | `spin_lock_irqsave` |
> | `dev->mutex` | open/close/grab、handle 链表增删 | 进程上下文 | `mutex_lock` |
> | `input_mutex` | 全局 dev_list、handler_list、connect/disconnect | 进程上下文 | `mutex_lock` |
> | `client->buffer_lock` | evdev 环形缓冲 head/tail/packet_head | IRQ 已关闭上下文 | `spin_lock` |
> | `evdev->client_lock` | client_list 的增删 | 进程上下文 | `spin_lock` |
> | RCU | grab 指针读取、h_list 遍历 | 任意上下文 | `rcu_read_lock` |

**锁序**: `input_mutex` → `dev->mutex` → `dev->event_lock` → `client->buffer_lock`

---

## 七、BSP 关键定制点

> [!example] 倒车影像场景定制
> ```c
> // input.c:285-287 — 倒车影像场景
> case EV_KEY:
>     if (is_event_supported(code, dev->keybit, KEY_MAX)) {
>         if (code == KEY_CAMERA) {  // 倒车事件直接分发，避免同状态被屏蔽
>             disposition = INPUT_PASS_TO_HANDLERS;
>         }
> ```

Android BSP 特有扩展:
- evdev 集成了 `wake_lock` 机制 (`EVIOCSSUSPENDBLOCK` ioctl)
- `EVIOCREVOKE` 被注释掉 (与 Android 用户空间冲突)

---

## 八、关键源码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| `input_event()` 入口 | `drivers/input/input.c` | 427-438 |
| `input_handle_event()` 帧构建 | `drivers/input/input.c` | 369-408 |
| `input_get_disposition()` 事件分类 | `drivers/input/input.c` | 259-367 |
| `input_pass_values()` 分发 | `drivers/input/input.c` | 130-163 |
| `input_to_handler()` filter+投递 | `drivers/input/input.c` | 96-123 |
| `input_register_device()` 设备注册 | `drivers/input/input.c` | 2077-2169 |
| `input_register_handler()` handler注册 | `drivers/input/input.c` | 2206-2226 |
| `input_attach_handler()` 匹配绑定 | `drivers/input/input.c` | 992-1007 |
| `input_match_device()` ID表匹配 | `drivers/input/input.c` | 935-989 |
| `struct input_dev` | `include/linux/input.h` | 121-190 |
| `struct input_handler` | `include/linux/input.h` | 284-305 |
| `struct input_handle` | `include/linux/input.h` | 319-331 |
| `struct input_value` | `include/linux/input.h` | 33-37 |
| `input_sync()` 宏 | `include/linux/input.h` | 412-415 |
| `input_mt_sync()` 宏 | `include/linux/input.h` | 417-420 |
| `struct evdev_client` 环形缓冲 | `drivers/input/evdev.c` | 45-60 |
| `evdev_events()` 批量接收 | `drivers/input/evdev.c` | 204-226 |
| `evdev_pass_values()` 写入缓冲 | `drivers/input/evdev.c` | 168-199 |
| `__pass_event()` 单事件+SYN_DROPPED | `drivers/input/evdev.c` | 137-166 |
| `evdev_read()` 阻塞读取 | `drivers/input/evdev.c` | 494-543 |
| `evdev_fetch_next_event()` | `drivers/input/evdev.c` | 473-492 |
| `evdev_poll()` | `drivers/input/evdev.c` | 546-563 |
| `evdev_open()` | `drivers/input/evdev.c` | 399-434 |
| `evdev_compute_buffer_size()` | `drivers/input/evdev.c` | 390-397 |
| `evdev_queue_syn_dropped()` | `drivers/input/evdev.c` | 109-135 |
| `evdev_connect()` | `drivers/input/evdev.c` | 1157-1226 |
| `input_polldev_queue_work()` | `drivers/input/input-polldev.c` | 25-34 |
| `input_polled_device_work()` | `drivers/input/input-polldev.c` | 36-43 |
| `gt9xx` 硬中断+工作队列 | `drivers/input/touchscreen/gt9xx.c` | 1071-1082, 573-1039 |
| `sirfsoc_ts` 线程化中断 | `drivers/input/touchscreen/sirfsoc_ts.c` | 499-521, 462-497 |
| `gpio_keys` 硬中断直接上报 | `drivers/input/keyboard/gpio_keys.c` | 398-431 |
| `input_mt_sync_frame()` | `drivers/input/input-mt.c` | 276-294 |
| `struct input_event`（用户空间 ABI） | `include/uapi/linux/input.h` | 24-28 |
| EV_SYN/SYN_REPORT/SYN_DROPPED | `include/uapi/linux/input.h` | 201-206 |

## 最后记住

> [!quote] 一句话总结
> input 子系统的核心是把驱动上报的零散事件累积成帧，再由 `SYN_REPORT` 触发批量分发。`spin_lock_irqsave` 保护所有上下文的调用，`packet_head` 保证用户空间永远读不到不完整帧，`SYN_DROPPED` 则是缓冲溢出的唯一显式信号。

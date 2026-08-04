# Linux Input 子系统学习路线与触摸屏实战任务拆解

## 1. 总体判断

你列出的目标可以分成两档：

- **合格线**：能读懂基础 input 驱动，能跑用户态事件读取程序，能完成触摸屏驱动适配，让 `/dev/input/eventX` 正常出现。
- **优秀线**：能深入理解 input/evdev 内核事件流，能设计触摸性能实验，能写测试工具，能定位漏点、延时、误触、多点错乱等问题。

建议学习顺序不要一开始就钻 `evdev` 缓存和调度细节。更稳的路线是：

1. 先会用：能读 `/dev/input/eventX`。
2. 再会配：设备树、内核配置、probe、中断、坐标范围。
3. 再会读：input core、evdev、触摸屏驱动上报流程。
4. 再会测：帧率、延时、坐标精度、丢点、误触。
5. 最后会定位：从硬件采样、总线、中断、驱动、input core、用户态逐层排查。

---

## 2. 优先级排序

### P0：必须先掌握，决定能不能入门

#### 任务 1：会读取原始 input 事件

目标：

- 能打开 `/dev/input/eventX`。
- 能解析 `struct input_event`。
- 能区分 `EV_KEY`、`EV_ABS`、`EV_REL`、`EV_SYN`。
- 能看到鼠标、键盘、触摸屏的原始事件。

原因：

- 这是 input 子系统的用户态入口。
- 后续所有测试工具、触摸坐标打印、延时统计都依赖它。

建议输出物：

- 一个 `ev_read.c` 示例程序。
- 一份事件日志样例。
- 能解释一组触摸事件含义。

验收标准：

- 能运行 `cat /proc/bus/input/devices` 找到设备。
- 能运行自己的程序读取 `/dev/input/eventX`。
- 能说清楚一次点击为什么会有 `ABS_X/ABS_Y/BTN_TOUCH/SYN_REPORT`。

---

#### 任务 2：理解 input 驱动最小上报流程

目标：

- 掌握 `input_allocate_device()`。
- 掌握 `input_register_device()`。
- 掌握 `input_report_key()`、`input_report_abs()`、`input_mt_slot()`、`input_mt_report_slot_state()`。
- 掌握 `input_sync()` 和 `SYN_REPORT` 的关系。

原因：

- 不懂这些函数，就看不懂任何触摸屏、按键、鼠标驱动。
- `input_sync()` 是理解事件成帧的关键。

建议阅读路径：

- 简单按键驱动。
- 简单 GPIO input 驱动。
- I2C 触摸屏驱动。
- input core 源码。

验收标准：

- 能从驱动中指出哪里申请 input 设备。
- 能指出哪里设置支持的事件类型。
- 能指出哪里在中断或工作队列里上报事件。
- 能解释为什么一批 `input_report_*()` 后要调用 `input_sync()`。

---

### P1：驱动适配能力，决定能不能上板调试

#### 任务 3：完成触摸屏物理与硬件链路梳理

目标：

- 区分电容屏和电阻屏。
- 理解触摸 IC 通过 I2C/SPI 输出坐标。
- 理解 IC 采样、滤波、坐标转换、中断通知主控的基本链路。

重点：

- 电容屏常见于多点触摸，依赖电容变化检测。
- 电阻屏通常通过压力接触形成电压分压，常见于单点或低成本方案。
- 触摸 IC 内部通常完成扫描、滤波、坐标计算，再通过 I2C/SPI 给主控读取。
- 主控侧驱动通常在中断触发后读取坐标寄存器，再通过 input 子系统上报。

验收标准：

- 能画出链路：手指触摸 -> 传感器矩阵 -> 触摸 IC -> I2C/SPI -> 中断 -> 驱动读取 -> input 上报 -> evdev -> 应用读取。
- 能解释采样频率、滤波参数为什么会影响触摸延时。

---

#### 任务 4：触摸屏驱动适配配置

目标：

- 设备树配置正确。
- 内核配置启用对应驱动。
- 驱动 probe 成功。
- 中断能触发。
- 坐标范围、翻转、交换、校准参数正确。
- `/dev/input/eventX` 正常生成。

建议优先排查顺序：

1. 硬件供电、复位、INT 引脚。
2. I2C/SPI 总线是否能扫描到设备。
3. 设备树 compatible 是否匹配驱动。
4. probe 是否执行成功。
5. 中断号、触发方式是否正确。
6. input device 是否注册成功。
7. `/proc/bus/input/devices` 是否出现。
8. `/dev/input/eventX` 是否生成。
9. 原始坐标是否正常。
10. 坐标方向、范围、屏幕映射是否正确。

验收标准：

- 能通过日志证明 probe 成功。
- 能通过中断计数证明触摸时中断增加。
- 能通过事件读取程序看到触摸坐标。
- 能解释设备树里每个关键字段的作用。

---

### P2：源码深入能力，决定能不能达到优秀

#### 任务 5：吃透 input 内核事件流

目标：

- 理解中断上下文和工作队列上报的差异。
- 理解 input core 如何接收 `input_report_*()`。
- 理解 evdev 如何缓存事件。
- 理解用户态阻塞读取如何被唤醒。
- 理解 `SYN_REPORT` 如何定义一帧事件边界。

核心流程：

```text
硬件触摸
  -> 触摸 IC 产生中断
  -> 驱动中断处理函数
  -> 读取 I2C/SPI 坐标数据
  -> input_report_abs/input_report_key/input_mt_*
  -> input_sync
  -> input core 分发事件
  -> evdev 接收并缓存事件
  -> 唤醒阻塞在 read/poll/epoll 的用户进程
  -> 用户态读取 struct input_event
```

重点源码方向：

- `drivers/input/input.c`
- `drivers/input/evdev.c`
- 具体触摸屏驱动，例如 Goodix、FocalTech、Ilitek、Synaptics 等平台实际使用的驱动。

重点问题：

- `input_report_abs()` 是否马上到用户态？
- `input_sync()` 为什么重要？
- evdev 缓存满了怎么办？
- 阻塞 `read()` 是在哪里睡眠、在哪里被唤醒？
- 中断里直接上报和工作队列里上报有什么差异？

验收标准：

- 能从驱动函数调用一路讲到用户态 `read()` 返回。
- 能解释 `SYN_REPORT` 代表一帧结束，不是一个普通坐标点。
- 能解释为什么多点触摸必须关注 slot、tracking id、position、sync。

---

### P3：测试实验与工具，决定能不能形成工程闭环

#### 任务 6：设计触摸性能验证实验

目标：

- 测单点触摸采样帧率。
- 测多点触摸采样帧率。
- 测触摸响应延时。
- 测坐标精度。
- 复现漏点、误触、多点识别问题。

实验指标建议：

| 指标 | 计算方式 | 数据来源 |
|---|---|---|
| 单点采样帧率 | 单点滑动时每秒 `SYN_REPORT` 帧数 | event 事件时间戳 |
| 多点采样帧率 | 多指滑动时每秒完整 MT 帧数 | event 事件时间戳 |
| 响应延时 | 外部触发时间到首个触摸事件时间差 | GPIO/示波器/高速摄像/event 时间 |
| 坐标精度 | 实测坐标与标定点坐标误差 | 测试治具或屏幕标定点 |
| 丢点率 | 预期点数与实际识别点数差异 | event 日志统计 |
| 误触率 | 无触摸状态下异常触摸事件数量 | 静置采样日志 |

实验报告结构：

1. 测试环境：板卡、内核版本、触摸 IC、屏幕尺寸、驱动版本。
2. 测试工具：事件读取程序、日志脚本、示波器或高速摄像。
3. 测试方法：每项指标如何执行。
4. 原始数据：事件日志和统计表。
5. 结果分析：帧率、延时、精度、异常现象。
6. 问题定位：对应硬件、驱动、调度、滤波、上层。
7. 临时缓解方案：调参、降噪、调整中断、限制误触、修改滤波。
8. 后续优化：长期修复建议。

---

#### 任务 7：输出可运行测试工具

工具建议分三版开发：

##### V1：原始事件读取工具

功能：

- 打开 `/dev/input/eventX`。
- 打印事件时间、type、code、value。
- 支持鼠标、键盘、触摸屏。

验收：

- 能替代 `evtest` 看基础事件。

##### V2：多点触摸解析工具

功能：

- 解析 `ABS_MT_SLOT`。
- 解析 `ABS_MT_TRACKING_ID`。
- 解析 `ABS_MT_POSITION_X`。
- 解析 `ABS_MT_POSITION_Y`。
- 每次 `SYN_REPORT` 打印当前所有活动触点。

验收：

- 能清楚看到 finger0、finger1、finger2 的坐标和 tracking id。
- 两指交换位置时能判断是否发生识别错乱。

##### V3：统计工具

功能：

- 统计每秒触摸帧率。
- 统计相邻帧间隔。
- 统计长间隔，判断疑似丢帧。
- 统计 tracking id 异常断开，判断疑似丢点。
- 静置时统计异常触摸，判断疑似误触。

验收：

- 能输出 CSV。
- 能输出简单汇总：平均帧率、最大间隔、疑似丢帧次数、疑似误触次数。

---

## 3. 推荐学习顺序

### 第一阶段：事件读取与 input 基础

周期建议：3 到 5 天。

任务：

- 学习 `struct input_event`。
- 编译运行事件读取程序。
- 使用鼠标、键盘、触摸屏观察事件。
- 理解 `EV_SYN` 和 `SYN_REPORT`。

输出：

- `ev_read.c`
- 事件日志样例
- 基础事件类型说明

优先级：最高。

---

### 第二阶段：驱动上报流程

周期建议：5 到 7 天。

任务：

- 阅读一个简单 input 驱动。
- 阅读实际触摸屏驱动。
- 梳理 `probe -> irq -> read coord -> input_report -> input_sync`。
- 理解中断上下文和工作队列。

输出：

- 驱动调用流程图
- input_report 系列函数笔记
- 中断与工作队列差异总结

优先级：高。

---

### 第三阶段：触摸屏适配

周期建议：1 到 2 周。

任务：

- 梳理电容屏、电阻屏原理。
- 理解 I2C/SPI 通信。
- 配置设备树。
- 检查内核配置。
- 验证 probe、中断、event 节点。
- 校准坐标范围和方向。

输出：

- 设备树配置说明
- probe 日志
- event 节点验证记录
- 坐标校准记录

优先级：高。

---

### 第四阶段：evdev 和 input core 深入

周期建议：1 到 2 周。

任务：

- 阅读 `drivers/input/input.c`。
- 阅读 `drivers/input/evdev.c`。
- 梳理 evdev 事件缓存。
- 梳理阻塞读取和唤醒路径。
- 理解 `SYN_REPORT` 成帧机制。

输出：

- input core 到 evdev 的源码流程图
- 阻塞 read 调用链
- evdev 缓存机制笔记

优先级：中高。

---

### 第五阶段：性能实验与故障定位

周期建议：2 到 3 周。

任务：

- 写多点触摸解析工具。
- 写帧率、延时、丢点统计工具。
- 设计并执行单点、多点、精度、误触实验。
- 建立故障定位表。

输出：

- 测试工具源码
- CSV 数据
- 实验报告
- 故障定位 checklist

优先级：优秀线核心。

---

## 4. 故障定位速查表

### 漏点

现象：

- 手指触摸存在，但 event 无坐标。
- 多指触摸时少识别一个点。
- 滑动过程中 tracking id 突然消失。

排查顺序：

1. 硬件是否真实采样到触摸。
2. 触摸 IC 是否有原始数据或寄存器状态。
3. 中断是否触发，中断计数是否增加。
4. I2C/SPI 读取是否失败或超时。
5. 驱动是否过滤掉了该点。
6. `input_sync()` 是否调用。
7. evdev 缓存是否溢出。
8. 用户态是否读取太慢。

临时缓解：

- 降低滤波阈值。
- 调整触摸 IC 灵敏度。
- 优化中断处理，避免耗时操作。
- 把耗时读取放到线程化中断或工作队列。
- 增大用户态读取频率。

---

### 触摸延时

现象：

- 手指触摸后 UI 反应慢。
- 滑动有拖影或跟手性差。
- event 帧间隔不稳定。

排查顺序：

1. IC 采样频率是否偏低。
2. IC 滤波窗口是否过大。
3. 中断触发是否及时。
4. 驱动是否在中断里做了耗时操作。
5. 工作队列是否被调度延迟。
6. input 上报是否及时。
7. 用户态是否阻塞或处理慢。
8. 图形系统或应用层是否引入延时。

临时缓解：

- 提高 IC report rate。
- 减小滤波窗口。
- 使用线程化中断。
- 减少中断处理路径耗时。
- 提高相关线程优先级。
- 减少用户态事件处理阻塞。

---

### 事件误报

现象：

- 无触摸时出现随机点。
- 静置时坐标抖动。
- 特定环境下频繁鬼点。

排查顺序：

1. 是否存在电磁干扰。
2. 地线、屏蔽、FPC、供电是否稳定。
3. IC 原始数据是否噪声过大。
4. 驱动是否缺少边界过滤。
5. 滤波阈值是否过低。
6. 坐标范围是否越界。
7. 是否把 release/report 顺序写错。

临时缓解：

- 提高触摸阈值。
- 增加去抖或稳定帧判断。
- 过滤明显越界坐标。
- 屏蔽边缘异常区域。
- 调整硬件接地和屏蔽。

---

### 多点识别错乱

现象：

- 两指交叉时坐标互换。
- 某个手指 tracking id 跳变。
- 多点坐标漂移或粘连。

排查顺序：

1. IC 上报的 tracking id 是否稳定。
2. 驱动是否正确使用 `ABS_MT_SLOT`。
3. `ABS_MT_TRACKING_ID` 是否按 down/up 正确设置。
4. release 时是否上报 `tracking id = -1`。
5. 每个 slot 的 X/Y 是否独立维护。
6. 是否每帧调用 `input_sync()`。
7. 是否坐标映射或旋转逻辑影响了多点。

临时缓解：

- 修正 slot 与 tracking id 维护逻辑。
- release 时补齐 `input_mt_report_slot_state()` 或 tracking id 释放。
- 增加坐标稳定性过滤。
- 对跳变过大的点做限速或丢弃。

---

## 5. ABS_MT 协议最小理解

多点触摸常见事件形态：

```text
EV_ABS ABS_MT_SLOT          0
EV_ABS ABS_MT_TRACKING_ID   45
EV_ABS ABS_MT_POSITION_X    320
EV_ABS ABS_MT_POSITION_Y    500
EV_ABS ABS_MT_SLOT          1
EV_ABS ABS_MT_TRACKING_ID   46
EV_ABS ABS_MT_POSITION_X    800
EV_ABS ABS_MT_POSITION_Y    520
EV_SYN SYN_REPORT           0
```

理解方式：

- `ABS_MT_SLOT`：当前正在更新哪个手指槽位。
- `ABS_MT_TRACKING_ID`：这个槽位对应哪一次连续触摸。
- `ABS_MT_POSITION_X/Y`：当前槽位的坐标。
- `SYN_REPORT`：这一帧多点数据结束。
- `TRACKING_ID = -1`：该 slot 的手指释放。

常见错误：

- 只上报 X/Y，不维护 slot。
- 两个手指复用同一个 slot。
- 手指释放时没有上报 tracking id 结束。
- 坐标更新后忘记 `input_sync()`。
- 旋转屏幕时只转换单点，没有正确转换所有 slot。

---

## 6. 测试工具设计草案

### 命令形式

```bash
touchmon /dev/input/eventX --mode raw
touchmon /dev/input/eventX --mode mt
touchmon /dev/input/eventX --mode stat --csv touch.csv
```

### raw 模式

输出：

```text
time=123.456789 type=EV_ABS code=ABS_MT_POSITION_X value=320
time=123.456800 type=EV_ABS code=ABS_MT_POSITION_Y value=500
time=123.456820 type=EV_SYN code=SYN_REPORT value=0
```

### mt 模式

输出：

```text
frame=102 time=123.456820 points=2
slot=0 id=45 x=320 y=500 active=1
slot=1 id=46 x=800 y=520 active=1
```

### stat 模式

输出：

```text
duration=60.0s
frames=7198
avg_fps=119.9
max_frame_gap_ms=24.3
suspected_drop_frames=3
suspected_false_touch=0
tracking_id_breaks=1
```

---

## 7. 最推荐的执行优先级

| 优先级 | 任务 | 原因 |
|---|---|---|
| 1 | 读取 `/dev/input/eventX` 原始事件 | 所有分析和测试的入口 |
| 2 | 掌握 `input_report_*()` 和 `input_sync()` | 看懂驱动的核心 |
| 3 | 梳理触摸屏硬件链路 | 方便区分硬件和软件问题 |
| 4 | 完成设备树、内核配置、probe、中断适配 | 真正上板调试必备 |
| 5 | 学习 ABS_MT 多点协议 | 多点触摸问题定位核心 |
| 6 | 阅读 input core 和 evdev | 达到优秀线的源码深度 |
| 7 | 写 raw/mt/stat 测试工具 | 建立可复用工程能力 |
| 8 | 做性能实验并写报告 | 形成完整闭环和成果沉淀 |
| 9 | 建立故障定位 checklist | 提升现场问题处理速度 |

---

## 8. 每周学习计划

### Week 1：input 用户态基础

- 编译运行事件读取程序。
- 分析键盘、鼠标、触摸屏事件。
- 理解 `struct input_event`。
- 理解 `SYN_REPORT`。

成果：

- 原始事件读取工具 V1。
- 事件类型笔记。

### Week 2：input 驱动基础

- 阅读简单 input 驱动。
- 学习 `input_allocate_device()`、`input_register_device()`。
- 学习 `input_report_key()`、`input_report_abs()`、`input_sync()`。
- 梳理中断上报逻辑。

成果：

- 驱动上报流程图。
- input_report 函数总结。

### Week 3：触摸屏硬件与驱动适配

- 学习电容屏、电阻屏原理。
- 学习 I2C/SPI 坐标读取流程。
- 配置设备树和内核选项。
- 验证 probe、中断、event 节点。

成果：

- 触摸屏适配记录。
- 坐标校准记录。

### Week 4：多点触摸协议

- 学习 `ABS_MT_SLOT` 和 `ABS_MT_TRACKING_ID`。
- 写多点坐标打印工具 V2。
- 复现两点触摸、交叉滑动、释放再按下。

成果：

- 多点触摸解析工具。
- 多点协议笔记。

### Week 5：input core 与 evdev

- 阅读 `drivers/input/input.c`。
- 阅读 `drivers/input/evdev.c`。
- 梳理事件分发、缓存、阻塞读取、唤醒。
- 理解 evdev client buffer。

成果：

- 源码调用链文档。
- read/poll 唤醒流程图。

### Week 6：性能实验

- 写统计工具 V3。
- 测单点、多点帧率。
- 测坐标精度。
- 测误触、漏点、延时。
- 输出实验报告。

成果：

- CSV 数据。
- 性能实验报告。
- 故障定位 checklist。

---

## 9. 最小测试代码方向

后续建议你先实现下面三个程序：

1. `ev_read.c`
   - 读取并打印原始 input_event。

2. `mt_print.c`
   - 解析 ABS_MT slot，打印每一帧多点坐标。

3. `touch_stat.c`
   - 统计帧率、最大帧间隔、疑似丢点、静置误触。

这三个工具可以逐步合并成一个 `touchmon`。

---

## 10. 最终能力验收清单

### 合格

- 能独立编译并运行 input 示例程序。
- 能读取鼠标、键盘、触摸屏原始事件。
- 能看懂基础 input 驱动。
- 能解释 `input_report_*()` 和 `input_sync()`。
- 能说明电容屏、电阻屏基本原理。
- 能说明 I2C/SPI 触摸 IC 坐标读取流程。
- 能完成设备树和内核配置。
- 能让 `/dev/input/eventX` 正常生成。

### 优秀

- 能讲清 input core 到 evdev 到用户态的完整路径。
- 能解释中断上下文、工作队列、线程化中断的取舍。
- 能解释 evdev 事件缓存和阻塞读取唤醒。
- 能正确理解 `SYN_REPORT` 成帧机制。
- 能看懂 ABS_MT 多点协议。
- 能写工具打印多点坐标。
- 能统计帧率、延时、丢帧、丢点。
- 能设计触摸性能实验并输出报告。
- 能定位漏点、延时、误触、多点错乱等基础触摸屏故障。


---
title: Goodix GT9xx 触摸驱动与 Linux Input 子系统分析
cssclasses:
---

# Goodix GT9xx 触摸驱动与 Linux Input 子系统分析

本文从 Linux input 子系统视角分析 Goodix GT9xx 触摸驱动，覆盖 input 设备注册、IRQ 与 workqueue、触点解析、MT Protocol A/B、同步事件、用户态 event 节点、电源管理和 remove 生命周期。

> [!summary] 核心主线
> GT9xx 驱动将 I2C 读取到的触点数据转换为标准 Linux input 事件：IRQ 负责触发，workqueue 负责读取和解析，`input_report_*()` 负责上报，`input_sync()` 负责提交完整帧，evdev 最终将事件暴露为 `/dev/input/eventX`。

> [!info] 分析范围
> | 模块 | 关键内容 |
> |---|---|
> | 设备注册 | `input_allocate_device()`、`input_register_device()` |
> | 事件能力 | `EV_SYN`、`EV_KEY`、`EV_ABS`、`INPUT_PROP_DIRECT` |
> | 触摸上报 | `ABS_MT_*`、`BTN_TOUCH`、`input_mt_sync()`、`input_sync()` |
> | 异步处理 | IRQ 顶半部、workqueue 底半部、hrtimer 轮询 |
> | 生命周期 | probe、suspend/resume、remove、错误回收 |

## 一、驱动在 Input 子系统中的位置

GT9xx 是一个基于 I2C 的触摸屏驱动。它负责从 Goodix 芯片读取触摸数据，再转换成 Linux input 事件。

总体链路：

```text
Goodix 触摸 IC
    ↓ I2C 坐标寄存器
INT GPIO
    ↓
Linux IRQ
    ↓
goodix_ts_irq_handler()
    ↓ queue_work()
goodix_ts_work_func()
    ↓
解析触点 ID、X、Y、W
    ↓
gtp_touch_down()/gtp_touch_up()
    ↓
input_report_abs()/input_report_key()
    ↓ input_sync()
Linux input core
    ↓
evdev
    ↓
/dev/input/eventX
    ↓
Android InputReader、libinput、tslib、evtest 等
```

驱动的工作可以分成四层：

1. **I2C 通信层**：读写 Goodix 寄存器。
2. **事件采集层**：通过 IRQ 或定时器触发坐标读取。
3. **坐标和触点处理层**：解析 ID、X、Y、触摸宽度，并进行坐标换算。
4. **input 上报层**：向 Linux input core 发送标准事件。

相关结构体：

```c
struct goodix_ts_data {
    struct i2c_client *client;
    struct input_dev  *input_dev;
    struct work_struct work;
    struct hrtimer timer;
    s32 use_irq;
    ...
};
```

定义位置：

```text
kernel/drivers/input/touchscreen/gt9xx.h
```

## 二、Input 设备的创建和注册

核心函数是：

```c
static s8 gtp_request_input_dev(struct goodix_ts_data *ts)
```

位置：

```text
kernel/drivers/input/touchscreen/gt9xx.c
```

### 2.1 分配 input_dev

```c
ts->input_dev = input_allocate_device();
```

这一步分配 Linux input 设备对象。`input_dev` 是驱动与 input core 之间的核心数据结构。

分配成功后，驱动继续配置该设备支持的事件类型、按键、绝对坐标轴和设备属性。

### 2.2 声明事件类型

```c
ts->input_dev->evbit[0] =
    BIT_MASK(EV_SYN) |
    BIT_MASK(EV_KEY) |
    BIT_MASK(EV_ABS);
```

表示设备支持：

| 事件类型 | 作用 |
|---|---|
| `EV_SYN` | 事件同步，一帧触摸数据结束时使用 |
| `EV_KEY` | 按键类事件，例如 `BTN_TOUCH`、触摸按键 |
| `EV_ABS` | 绝对坐标事件，例如 X/Y、压力、触摸 ID |

触摸屏不是鼠标，不使用 `EV_REL` 相对位移事件，而是使用 `EV_ABS` 绝对坐标事件。

### 2.3 声明直接触摸设备

```c
__set_bit(INPUT_PROP_DIRECT,
          ts->input_dev->propbit);
```

`INPUT_PROP_DIRECT` 表示这是直接输入设备：

```text
触摸点位置 直接对应 屏幕位置
```

这与触摸板不同。触摸板通常是间接输入设备，手指移动对应鼠标指针移动。

### 2.4 设置设备名称和总线类型

```c
ts->input_dev->name = goodix_ts_name;
ts->input_dev->phys = goodix_input_phys;
ts->input_dev->id.bustype = BUS_I2C;
ts->input_dev->id.vendor = 0xDEAD;
ts->input_dev->id.product = 0xBEEF;
ts->input_dev->id.version = 10427;
```

当前定义：

```c
static const char *goodix_ts_name = "goodix-ts";
static const char *goodix_input_phys = "input/ts";
```

用户态可以看到类似信息：

```text
Name="goodix-ts"
Phys=input/ts
BUSTYPE=I2C
```

这些信息可通过以下方式查看：

```bash
cat /proc/bus/input/devices
```

或：

```bash
evtest /dev/input/eventX
```

### 2.5 注册 input 设备

```c
ret = input_register_device(ts->input_dev);
```

注册成功后，input core 会接管该设备，并由 evdev 等 input handler 创建用户态访问路径。

驱动不会自己调用 `mknod()` 创建 `/dev/input/eventX`。通常链路是：

```text
input_register_device()
    ↓
input core
    ↓
evdev handler
    ↓
/dev/input/eventX
```

## 三、绝对坐标轴和触摸能力

### 3.1 坐标轴

驱动配置：

```c
input_set_abs_params(ts->input_dev,
                     ABS_MT_POSITION_X,
                     0,
                     ts->display_resolution_x,
                     0,
                     0);

input_set_abs_params(ts->input_dev,
                     ABS_MT_POSITION_Y,
                     0,
                     ts->display_resolution_y,
                     0,
                     0);
```

主要绝对轴如下：

| 轴 | 作用 |
|---|---|
| `ABS_MT_POSITION_X` | 多点触摸 X 坐标 |
| `ABS_MT_POSITION_Y` | 多点触摸 Y 坐标 |
| `ABS_MT_TOUCH_MAJOR` | 触摸接触面积/主轴尺寸 |
| `ABS_MT_WIDTH_MAJOR` | 触摸宽度 |
| `ABS_MT_TRACKING_ID` | 触点生命周期 ID |

### 3.2 当前坐标范围

设备树示例：

```dts
display_resolution_x = <1024>;
display_resolution_y = <600>;
```

驱动注册的逻辑范围是：

```text
ABS_MT_POSITION_X：0 ~ 1024
ABS_MT_POSITION_Y：0 ~ 600
```

但实际缩放公式使用：

```c
display_resolution_x - 1
display_resolution_y - 1
```

因此实际报告值通常是：

```text
X：0 ~ 1023
Y：0 ~ 599
```

这是 input 能力声明最大值和实际报告最大值之间的一个小差异。

## 四、IRQ、Workqueue 与 Input 上报

### 4.1 中断入口

中断处理函数：

```c
static irqreturn_t goodix_ts_irq_handler(int irq, void *dev_id)
{
    struct goodix_ts_data *ts = dev_id;

    gtp_irq_disable(ts);
    queue_work(goodix_wq, &ts->work);

    return IRQ_HANDLED;
}
```

中断上下文中只做两件事：

1. 禁止当前触摸 IRQ，防止重复进入。
2. 将实际工作放入 workqueue。

I2C 通信可能睡眠，不能直接在硬中断上下文中执行，因此坐标读取放在 workqueue 中是合理的。

### 4.2 workqueue 入口

初始化：

```c
INIT_WORK(&ts->work, goodix_ts_work_func);
```

工作函数：

```c
static void goodix_ts_work_func(struct work_struct *work)
{
    struct goodix_ts_data *ts;

    ts = container_of(work,
                      struct goodix_ts_data,
                      work);
    ...
}
```

`container_of()` 根据 `work` 成员地址反推出包含它的 `goodix_ts_data` 地址。

### 4.3 处理完成后重新开启 IRQ

工作函数完成 I2C 读取、坐标解析和 input 上报后：

```c
if (ts->use_irq)
    gtp_irq_enable(ts);
```

因此一次 IRQ 周期是：

```text
IRQ 触发
    ↓
gtp_irq_disable()
    ↓
queue_work()
    ↓
I2C 读取坐标
    ↓
input 上报
    ↓
写 end_cmd 清除芯片状态
    ↓
gtp_irq_enable()
```

## 五、触点数据到 Input 事件

### 5.1 读取触摸数据

工作函数首先读取 Goodix 坐标寄存器：

```c
ret = gtp_i2c_read(ts->client,
                   point_data,
                   12);
```

读取结果中的状态字节：

```c
finger = point_data[GTP_ADDR_LENGTH];
```

通常：

```text
finger bit[7]：数据是否有效
finger bit[3:0]：触点数量
```

代码：

```c
if ((finger & 0x80) == 0)
    goto exit_work_func;

touch_num = finger & 0x0f;
```

如果触点数超过：

```c
#define GTP_MAX_TOUCH 5
```

则当前帧被丢弃：

```c
if (touch_num > GTP_MAX_TOUCH)
    goto exit_work_func;
```

### 5.2 解析单个触点

当前每个触点占 8 字节：

```c
id = coor_data[0] & 0x0F;

input_x = coor_data[1] |
          (coor_data[2] << 8);

input_y = coor_data[3] |
          (coor_data[4] << 8);

input_w = coor_data[5] |
          (coor_data[6] << 8);
```

字段含义：

```text
id      ：触点 ID
input_x  ：Goodix 原始 X 坐标
input_y  ：Goodix 原始 Y 坐标
input_w  ：触摸面积或压力近似值
```

然后进入：

```c
gtp_touch_down(ts,
               id,
               input_x,
               input_y,
               input_w);
```

### 5.3 触点释放

当 Goodix 报告没有触点，同时上一帧仍有触点时：

```c
else if (pre_touch)
{
    gtp_touch_up(ts, 0);
}
```

非 slot 模式的释放代码：

```c
input_report_key(ts->input_dev,
                 BTN_TOUCH,
                 0);
```

然后在工作函数末尾通过：

```c
input_sync(ts->input_dev);
```

提交释放事件。

## 六、MT Protocol A 与 Protocol B

头文件中：

```c
#define GTP_ICS_SLOT_REPORT 0
```

因此当前 BSP 默认走 **MT Protocol A**，不是 slot-based Protocol B。

### 6.1 Protocol A

Protocol A 触点上报代码：

```c
input_report_key(ts->input_dev,
                 BTN_TOUCH,
                 1);

input_report_abs(ts->input_dev,
                 ABS_MT_POSITION_X,
                 x);

input_report_abs(ts->input_dev,
                 ABS_MT_POSITION_Y,
                 y);

input_report_abs(ts->input_dev,
                 ABS_MT_TOUCH_MAJOR,
                 w);

input_report_abs(ts->input_dev,
                 ABS_MT_WIDTH_MAJOR,
                 w);

input_report_abs(ts->input_dev,
                 ABS_MT_TRACKING_ID,
                 id);

input_mt_sync(ts->input_dev);
```

其中：

- `input_report_abs()` 报告当前触点的字段。
- `input_mt_sync()` 分隔当前帧中的不同触点。
- `input_sync()` 结束整帧事件。

一次多点触摸帧大致是：

```text
触点 0：X/Y/W/ID
SYN_MT_REPORT
触点 1：X/Y/W/ID
SYN_MT_REPORT
触点 2：X/Y/W/ID
SYN_MT_REPORT
SYN_REPORT
```

### 6.2 Protocol B

如果配置为：

```c
#define GTP_ICS_SLOT_REPORT 1
```

驱动会使用 slot 方式：

```c
input_mt_init_slots(ts->input_dev, 16);
```

触点按下或更新：

```c
input_mt_slot(ts->input_dev, id);
input_report_abs(ts->input_dev,
                 ABS_MT_TRACKING_ID,
                 id);
input_report_abs(ts->input_dev,
                 ABS_MT_POSITION_X,
                 x);
input_report_abs(ts->input_dev,
                 ABS_MT_POSITION_Y,
                 y);
```

触点释放：

```c
input_mt_slot(ts->input_dev, id);
input_report_abs(ts->input_dev,
                 ABS_MT_TRACKING_ID,
                 -1);
```

Protocol B 用 slot 保存每个触点的状态，更适合现代 input 子系统。

当前代码虽然包含 B 路径，但默认宏为 0，所以实际编译运行时主要走 A 路径。

### 6.3 两种协议对比

| 项目 | Protocol A | Protocol B |
|---|---|---|
| 触点分隔 | `input_mt_sync()` | slot 状态管理 |
| 触点状态 | 由事件序列表达 | 由 slot 保存 |
| 主要接口 | `ABS_MT_*` + `SYN_MT_REPORT` | `input_mt_slot()`、slot state |
| 当前驱动默认 | 是 | 否 |
| 现代推荐 | 旧式兼容 | 更推荐 |

## 七、同步事件的含义

本驱动中有两类同步。

### 7.1 触点级同步

Protocol A 中：

```c
input_mt_sync(ts->input_dev);
```

含义是：

```text
当前这个触点的坐标、ID、宽度等字段已经上报完成。
```

它用于分隔同一帧中的多个触点。

### 7.2 帧级同步

```c
input_sync(ts->input_dev);
```

含义是：

```text
当前整帧触摸数据已经上报完成，可以交给 input core 和用户态。
```

因此：

```text
input_mt_sync()：结束一个触点
input_sync()：结束一整帧
```

## 八、触摸按键事件

头文件中：

```c
#define GTP_HAVE_TOUCH_KEY 1
```

驱动支持将触摸 IC 数据中的按键位转换成 Linux key event。

能力声明：

```c
input_set_capability(ts->input_dev,
                     EV_KEY,
                     touch_key_array[index]);
```

按键上报：

```c
input_report_key(ts->input_dev,
                 touch_key_array[i],
                 key_value & (0x01 << i));
```

如果设备树配置了：

```dts
touchscreen-key-map = <KEY_HOME KEY_BACK>;
```

驱动会读取该属性并使用自定义键值映射。

触摸按键和手指触摸存在互斥处理：

```c
touch_num = 0;
```

这会屏蔽当前帧的手指触点，避免同一份数据同时被解释成普通触点和触摸按键。

## 九、Input 设备和用户态的关系

### 9.1 内核侧

```text
gtp_touch_down()
    ↓
input_report_abs()
    ↓
input_mt_sync()
    ↓
input_sync()
    ↓
input core
```

### 9.2 用户态

```text
input core
    ↓
evdev
    ↓
/dev/input/eventX
    ↓
evtest / getevent / libinput / Android InputReader
```

驱动本身主要输出标准 input 事件，不直接负责：

- 手势识别
- 窗口分发
- 多指手势算法
- UI 坐标处理

这些通常由用户态输入框架继续处理。

### 9.3 常用检查接口

查看所有 input 设备：

```bash
cat /proc/bus/input/devices
```

查看 event 设备：

```bash
ls -l /dev/input/
```

查看事件：

```bash
evtest /dev/input/eventX
```

Android 系统常用：

```bash
getevent -lp
getevent -lt
```

查看中断：

```bash
cat /proc/interrupts
```

## 十、从 Probe 到 Input 设备可用

`goodix_ts_probe()` 中的主要顺序：

```text
I2C 设备匹配
    ↓
分配 goodix_ts_data
    ↓
解析设备树
    ↓
打开电源
    ↓
申请 RST/INT GPIO
    ↓
复位 Goodix
    ↓
测试 I2C 通信
    ↓
读取版本和配置
    ↓
初始化 work_struct
    ↓
分配并注册 input_dev
    ↓
申请 IRQ
    ↓
开启 IRQ 或启动轮询
    ↓
等待触摸事件
```

关键代码：

```c
INIT_WORK(&ts->work, goodix_ts_work_func);
```

```c
ret = gtp_request_input_dev(ts);
```

```c
ret = gtp_request_irq(ts);
```

正常情况下，input 设备注册完成后才打开触摸事件入口。

## 十一、Suspend/Resume 对 Input 的影响

### suspend

```c
goodix_ts_suspend(ts);
```

主要操作：

```text
设置 gtp_is_suspend
    ↓
关闭 ESD 检查
    ↓
关闭 IRQ 或停止 hrtimer
    ↓
发送 Goodix sleep 命令
```

input 设备通常不会被注销，`/dev/input/eventX` 仍然存在，只是底层不再产生新的触摸事件。

### resume

```c
goodix_ts_resume(ts);
```

主要操作：

```text
唤醒/复位 Goodix
    ↓
重新发送配置
    ↓
重新打开 IRQ 或 hrtimer
    ↓
恢复 ESD 检查
```

## 十二、Remove 与生命周期问题

当前 `goodix_ts_remove()` 中需要重点关注以下问题。

### 12.1 没有取消主 work

probe 中初始化：

```c
INIT_WORK(&ts->work, goodix_ts_work_func);
```

但是 remove 中没有看到：

```c
cancel_work_sync(&ts->work);
```

如果 `ts->work` 仍在 `goodix_wq` 中排队，remove 又释放了：

```c
input_unregister_device(ts->input_dev);
kfree(ts);
```

就可能出现 work 继续访问已释放 `ts` 或 `input_dev` 的风险。

建议的资源释放顺序通常是：

```text
停止新 IRQ/停止 timer
    ↓
free_irq()
    ↓
cancel_work_sync()
    ↓
注销 input_dev
    ↓
释放 GPIO、workqueue、ts
```

### 12.2 proc 节点清理不完整

probe 中创建：

```c
gt91xx_config_proc = proc_create(...);
```

remove 中应对称删除 proc 节点，例如旧内核使用：

```c
remove_proc_entry(...);
```

或者新内核使用：

```c
proc_remove(gt91xx_config_proc);
```

当前代码需要检查并补充对应清理。

### 12.3 sysfs group 清理不完整

probe 中：

```c
sysfs_create_group(&client->dev.kobj,
                   &gtp_attr_group);
```

remove 中应调用：

```c
sysfs_remove_group(&client->dev.kobj,
                   &gtp_attr_group);
```

否则设备反复 bind/unbind 时可能留下不完整的 sysfs 生命周期。

### 12.4 RST GPIO 释放

申请 RST GPIO 的代码在 `gtp_request_io_port()`，但 remove 主要释放 INT GPIO。应确认并补充：

```c
GTP_GPIO_FREE(gtp_rst_gpio);
```

### 12.5 probe 错误路径

`gtp_request_input_dev()` 失败后当前代码只是打印错误，仍可能继续申请 IRQ：

```c
ret = gtp_request_input_dev(ts);
if (ret < 0) {
    GTP_ERROR("GTP request input dev failed");
}

ret = gtp_request_irq(ts);
```

更稳妥的方式是立即进入统一错误回收路径，避免出现“驱动 probe 看似成功，但没有有效 input 设备”的情况。

## 十三、值得掌握的 Input 子系统知识

1. `struct input_dev` 的创建和注册。
2. `EV_SYN`、`EV_KEY`、`EV_ABS` 的含义。
3. `INPUT_PROP_DIRECT` 和触摸屏/触摸板的区别。
4. `input_set_abs_params()` 配置绝对坐标轴。
5. `input_report_abs()` 上报坐标、触摸宽度和 tracking ID。
6. `input_report_key()` 上报 `BTN_TOUCH` 和触摸按键。
7. `input_mt_sync()` 与 `input_sync()` 的区别。
8. MT Protocol A 和 Protocol B 的区别。
9. 触点 ID 与 tracking ID 的关系。
10. IRQ 顶半部和 workqueue 底半部的配合。
11. input core、evdev 和 `/dev/input/eventX` 的关系。
12. suspend/resume 时 input 设备和底层硬件的关系。
13. input 设备注销与异步 work 的生命周期管理。
14. probe/remove 错误路径和资源回收。
15. 设备树参数如何影响 input 坐标范围和坐标变换。

## 十四、完整流程总结

```text
input_allocate_device()
    ↓
设置 EV_SYN/EV_KEY/EV_ABS
    ↓
设置 INPUT_PROP_DIRECT
    ↓
设置 ABS_MT_POSITION_X/Y 等绝对轴
    ↓
input_register_device()
    ↓
IRQ 或 hrtimer 触发
    ↓
workqueue 中读取 I2C 触摸数据
    ↓
解析触点 ID/X/Y/W
    ↓
坐标变换和缩放
    ↓
input_report_abs()/input_report_key()
    ↓
input_mt_sync()
    ↓
input_sync()
    ↓
/dev/input/eventX
```

这份代码默认：

```c
#define GTP_ICS_SLOT_REPORT 0
```

因此实际采用 MT Protocol A 风格。代码同时包含 Protocol B 的 slot 分支，但并未作为当前默认路径使用。

### 学习重点

从学习角度看，最值得掌握的是：

```text
硬件中断
    → workqueue
    → I2C 读取
    → 触点解析
    → input_report_*()
    → input_sync()
    → evdev
    → 用户态
```


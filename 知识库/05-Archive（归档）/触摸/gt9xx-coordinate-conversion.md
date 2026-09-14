---
title: Goodix GT9xx 触摸坐标换算与缩放分析
cssclasses:
  - archive-page
---

# Goodix GT9xx 触摸坐标换算与缩放分析

本文总结 `gt9xx.c` 中触摸坐标从 Goodix 芯片原始数据，到 Linux input 子系统坐标的完整过程，并结合设备树示例说明各参数作用。

> [!summary] 核心主线
> Goodix 原始坐标依次经过方向翻转、有效区域平移、线性缩放和偏移修正，最后通过 `input_report_abs()` 上报到 `/dev/input/eventX`。排查偏移时，应先确认设备树参数，再逐步比对转换前后的坐标。

> [!info] 阅读入口
> | 关注点 | 对应内容 |
> |---|---|
> | 驱动入口 | `goodix_ts_probe()`、`gtp_parse_dt()` |
> | 数据读取 | `goodix_ts_work_func()`、`gtp_i2c_read()` |
> | 坐标转换 | `gtp_touch_down()`、`transform_coordinate()` |
> | Input 上报 | `gtp_request_input_dev()`、`input_report_abs()` |
> | 源码路径 | `drivers/input/touchscreen/gt9xx.c` |

## 一、相关代码

主要函数：

- `goodix_ts_probe()`：驱动 probe，解析设备树并初始化触摸设备
- `gtp_parse_dt()`：读取设备树中的 GPIO、触摸区域、分辨率、方向和偏移
- `goodix_ts_work_func()`：I2C 读取并解析触摸数据
- `gtp_touch_down()`：单个触点的坐标换算和 input 上报
- `transform_coordinate()`：坐标方向变换和触摸区域平移
- `gtp_request_input_dev()`：设置 input 坐标轴范围

源码位置：

```text
drivers/input/touchscreen/gt9xx.c
```

## 二、坐标数据链路

```text
手指触摸
    ↓
Goodix 芯片内部扫描、计算和滤波
    ↓
坐标写入 Goodix 坐标寄存器
    ↓
INT GPIO 触发 Linux IRQ
    ↓
goodix_ts_irq_handler()
    ↓
queue_work(goodix_wq, &ts->work)
    ↓
goodix_ts_work_func()
    ↓
gtp_i2c_read() 读取原始坐标
    ↓
解析 ID、原始 X、原始 Y、触摸宽度
    ↓
gtp_touch_down()
    ↓
方向变换、区域平移、坐标缩放、偏移修正
    ↓
input_report_abs(ABS_MT_POSITION_X/Y)
    ↓
input_sync()
    ↓
/dev/input/eventX
```

驱动中断处理函数只负责关中断和调度 work，不在硬中断上下文中进行 I2C 读写：

```c
static irqreturn_t goodix_ts_irq_handler(int irq, void *dev_id)
{
    struct goodix_ts_data *ts = dev_id;

    gtp_irq_disable(ts);
    queue_work(goodix_wq, &ts->work);

    return IRQ_HANDLED;
}
```

真正的 I2C 读取和坐标处理在 `goodix_ts_work_func()` 中完成。

## 三、原始坐标解析

Goodix 每个触点通常占 8 字节。当前头文件中：

```c
#define GTP_MAX_TOUCH 5
#define GTP_ICS_SLOT_REPORT 0
```

因此当前使用非 slot 的 Type A 风格上报。

代码解析单个触点：

```c
id = coor_data[0] & 0x0F;

input_x = coor_data[1] |
          (coor_data[2] << 8);

input_y = coor_data[3] |
          (coor_data[4] << 8);

input_w = coor_data[5] |
          (coor_data[6] << 8);
```

其中：

```text
input_x：Goodix 原始 X 坐标
input_y：Goodix 原始 Y 坐标
input_w：触摸宽度/压力值
id：触点 ID
```

然后调用：

```c
gtp_touch_down(ts, id, input_x, input_y, input_w);
```

注意：此时 `input_x/input_y` 还不是最终 LCD 或 input 子系统坐标。

## 四、设备树配置

当前工程中可参考：

```text
kernel/arch/arm/boot/dts/atlas7-cvte-after-market-common.dtsi
```

Goodix 节点示例：

```dts
goodix_ts@5d {
    status = "okay";
    compatible = "goodix,gt9xx";
    reg = <0x5d>;

    interrupt-parent = <&gpio_1>;
    interrupts = <0 8>;

    goodix,rst-gpio = <&gpio_1 1 0>;
    goodix,irq-gpio = <&gpio_1 0 0>;

    display_area_x_min = <53>;
    display_area_y_min = <0>;
    display_area_x_max = <1044>;
    display_area_y_max = <600>;

    display_resolution_x = <1024>;
    display_resolution_y = <600>;

    origin_position = <1>;
};
```

### 设备树参数含义

| 属性 | 示例值 | 作用 |
|---|---:|---|
| `reg` | `0x5d` | Goodix I2C 地址 |
| `goodix,rst-gpio` | `gpio_1 1` | 芯片复位 GPIO |
| `goodix,irq-gpio` | `gpio_1 0` | 触摸中断 GPIO |
| `display_area_x_min` | `53` | 原始 X 有效区域最小值 |
| `display_area_x_max` | `1044` | 原始 X 有效区域最大值 |
| `display_area_y_min` | `0` | 原始 Y 有效区域最小值 |
| `display_area_y_max` | `600` | 原始 Y 有效区域最大值 |
| `display_resolution_x` | `1024` | 最终 input X 分辨率 |
| `display_resolution_y` | `600` | 最终 input Y 分辨率 |
| `origin_position` | `1` | 坐标方向/镜像方式 |
| `x_offset` | 可选 | X 方向整体偏移 |
| `y_offset` | 可选 | Y 方向整体偏移 |

`gtp_parse_dt()` 在 probe 阶段读取这些属性并保存到：

```c
struct goodix_ts_data
```

例如：

```c
ret = of_property_read_u32(np,
                           "display_area_x_max",
                           &ts->display_area_x_max);
```

如果属性不存在，代码使用默认值：

```c
if (ret) {
    ts->display_area_x_max = 1024;
}
```

因此：

```text
DTS 属性
   ↓ probe 阶段
of_property_read_u32()
   ↓
ts->display_area_x_max
   ↓触摸发生时
gtp_touch_down()/transform_coordinate()
```

## 五、坐标换算实现

核心代码：

```c
static void gtp_touch_down(struct goodix_ts_data* ts,
                           s32 id,
                           s32 input_x,
                           s32 input_y,
                           s32 w)
{
    s32 x, y, cx, cy;

    x = input_x;
    y = input_y;

    cx = ts->display_area_x_max - ts->display_area_x_min;
    cy = ts->display_area_y_max - ts->display_area_y_min;

    transform_coordinate(ts, &x, &y);

#if GTP_CHANGE_X2Y
    GTP_SWAP(x, y);
#endif

    if (x < 0) {
        /* 当前代码这里没有 return，见后面的注意事项 */
    }
    else if ((x > ts->display_area_x_max) &&
             (0 != ts->display_area_x_xingwei)) {
        x = -ts->display_area_x_xingwei;
    }
    else {
        x = x * (ts->display_resolution_x - 1) / cx
            + ts->x_offset;

        y = y * (ts->display_resolution_y - 1) / cy
            + ts->y_offset;
    }

    input_report_abs(ts->input_dev, ABS_MT_POSITION_X, x);
    input_report_abs(ts->input_dev, ABS_MT_POSITION_Y, y);
}
```

## 六、坐标换算步骤

### 第一步：读取原始坐标

```text
x = input_x
y = input_y
```

例如：

```text
input_x = 100
input_y = 200
```

### 第二步：计算原始有效区域宽高

```c
cx = display_area_x_max - display_area_x_min;
cy = display_area_y_max - display_area_y_min;
```

按照设备树示例：

```text
cx = 1044 - 53 = 991
cy = 600 - 0 = 600
```

这里的 `cx/cy` 是 Goodix 原始坐标的有效触摸范围，不是 LCD 分辨率。

### 第三步：方向变换

代码：

```c
switch (ts->origin_position) {
case LEFT_TOP:
    break;
case RIGHT_TOP:
    *x = ts->display_area_x_max - *x;
    break;
case LEFT_DOWN:
    *y = ts->display_area_y_max - *y;
    break;
case RIGHT_DOWN:
    *x = ts->display_area_x_max - *x;
    *y = ts->display_area_y_max - *y;
    break;
}
```

方向定义：

| `origin_position` | 含义 | 处理 |
|---:|---|---|
| `0` | `LEFT_TOP` | X/Y 不变 |
| `1` | `RIGHT_TOP` | X 翻转 |
| `2` | `LEFT_DOWN` | Y 翻转 |
| `3` | `RIGHT_DOWN` | X/Y 都翻转 |

设备树示例中：

```dts
origin_position = <1>;
```

因此：

```c
x = display_area_x_max - x;
```

即：

```text
x = 1044 - 原始 x
y = 原始 y
```

### 第四步：减去有效区域起点

```c
*x = *x - ts->display_area_x_min;
*y = *y - ts->display_area_y_min;
```

设备树示例中：

```text
x = x - 53
y = y - 0
```

这一步把原始触摸坐标转换到从 0 开始的有效触摸区域坐标系。

### 第五步：线性缩放到显示分辨率

代码：

```c
x = x * (display_resolution_x - 1) / cx + x_offset;
y = y * (display_resolution_y - 1) / cy + y_offset;
```

通用公式：

```text
Xout = (Xtransformed - x_min)
       × (resolution_x - 1)
       / (x_max - x_min)
       + x_offset
```

```text
Yout = (Ytransformed - y_min)
       × (resolution_y - 1)
       / (y_max - y_min)
       + y_offset
```

其中 `Xtransformed/Ytransformed` 是方向翻转后的坐标。

## 七、代入设备树的完整公式

设备树：

```dts
display_area_x_min = <53>;
display_area_x_max = <1044>;
display_area_y_min = <0>;
display_area_y_max = <600>;
display_resolution_x = <1024>;
display_resolution_y = <600>;
origin_position = <1>;
```

因为 `origin_position = 1`，当前代码的 X 方向先翻转：

```text
X1 = 1044 - RawX
Y1 = RawY
```

然后减去区域起点：

```text
X2 = X1 - 53
Y2 = Y1 - 0
```

有效区域尺寸：

```text
cx = 1044 - 53 = 991
cy = 600 - 0 = 600
```

最终输出：

```text
Xout = X2 × 1023 / 991 + x_offset
Yout = Y2 × 599  / 600 + y_offset
```

如果 `x_offset/y_offset` 未配置，默认都是 0，则：

```text
Xout = (1044 - RawX - 53) × 1023 / 991
Yout = RawY × 599 / 600
```

也可以写成：

```text
Xout = (991 - RawX) × 1023 / 991
Yout = RawY × 599 / 600
```

这里的 `RawX` 代入的是代码方向变换之前的原始坐标。

## 八、完整计算示例

假设 Goodix 上报：

```text
RawX = 100
RawY = 200
```

### 8.1 X 方向翻转

```text
X1 = 1044 - 100 = 944
```

### 8.2 减去有效区域起点

```text
X2 = 944 - 53 = 891
```

Y 方向：

```text
Y2 = 200 - 0 = 200
```

### 8.3 X/Y 缩放

```text
Xout = 891 × 1023 / 991 ≈ 920
Yout = 200 × 599 / 600 ≈ 199
```

### 8.4 最终上报

```text
ABS_MT_POSITION_X = 920
ABS_MT_POSITION_Y = 199
```

同时还会报告触摸 ID、触摸面积等事件，最后通过：

```c
input_sync(ts->input_dev);
```

提交一帧 input 事件。

## 九、Input 子系统坐标范围

在 `gtp_request_input_dev()` 中设置：

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

设备树示例对应：

```text
ABS_MT_POSITION_X：0 ~ 1024
ABS_MT_POSITION_Y：0 ~ 600
```

但缩放公式使用：

```c
display_resolution_x - 1
display_resolution_y - 1
```

所以实际通常上报：

```text
X：0 ~ 1023
Y：0 ~ 599
```

这造成 input 能力声明和实际上报最大值存在 1 个单位的差异。若希望严格一致，可将 `input_set_abs_params()` 的最大值改为：

```c
 ts->display_resolution_x - 1
 ts->display_resolution_y - 1
```

## 十、`x_offset` 与 `y_offset`

设备树可选配置：

```dts
x_offset = <0 10>;
y_offset = <1 5>;
```

解析逻辑：

```c
ts->x_offset = offset[0] ? (-offset[1]) : offset[1];
ts->y_offset = offset[0] ? (-offset[1]) : offset[1];
```

因此：

```text
<0 10> → +10
<1 10> → -10
```

上述示例得到：

```text
x_offset = +10
y_offset = -5
```

最终公式变为：

```text
Xout = scaled_x + 10
Yout = scaled_y - 5
```

用途是修正触摸整体偏移：

```text
触摸整体偏左 → 增大 x_offset
触摸整体偏右 → 减小 x_offset
触摸整体偏上 → 增大 y_offset
触摸整体偏下 → 减小 y_offset
```

## 十一、代码注意事项

### 11.1 `origin_position` 的镜像公式

当前代码：

```c
*x = ts->display_area_x_max - *x;
```

当 `display_area_x_min != 0` 时，更严格的有效区域镜像通常应考虑最小值：

```c
*x = ts->display_area_x_min +
     ts->display_area_x_max - *x;
```

或：

```c
*x = ts->display_area_x_max -
     (*x - ts->display_area_x_min);
```

当前设备树中 `x_min = 53`，因此建议实测左右边界，确认是否存在固定偏差。

### 11.2 `if (x < 0)` 并没有真正丢弃触点

当前代码：

```c
if (x < 0) {
    // do nothing
}
```

这里只是不执行缩放，但函数后续仍可能继续上报该坐标。如果设计意图是丢弃越界点，应改为：

```c
if (x < 0)
    return;
```

或者进行钳位：

```c
if (x < 0)
    x = 0;
```

### 11.3 `display_area_x_max` 不是 LCD 分辨率

```text
display_area_x_max
```

表示 Goodix 原始坐标有效区域上限；

```text
display_resolution_x
```

表示最终 input/LCD 逻辑坐标分辨率。

例如：

```text
原始有效 X：53 ~ 1044
输出 X：0 ~ 1023
```

两者可以不同，驱动正是通过比例公式完成映射。

## 十二、调试方法

### 12.1 查看设备树解析结果

驱动在 `gtp_parse_dt()` 中会打印配置：

```c
dev_err(dev,
        "xi(%d),yi(%d),xa(%d),ya(%d),dx(%d),dy(%d),op(%d),...",
        ts->display_area_x_min,
        ts->display_area_y_min,
        ts->display_area_x_max,
        ts->display_area_y_max,
        ts->display_resolution_x,
        ts->display_resolution_y,
        ts->origin_position);
```

查看：

```bash
dmesg | grep -i 'xi('
```

期望看到类似：

```text
xi(53),yi(0),xa(1044),ya(600),dx(1024),dy(600),op(1)
```

### 12.2 查看最终 input 事件

先查看设备：

```bash
cat /proc/bus/input/devices
```

然后使用：

```bash
evtest /dev/input/eventX
```

重点观察：

```text
ABS_MT_POSITION_X
ABS_MT_POSITION_Y
ABS_MT_TRACKING_ID
BTN_TOUCH
SYN_REPORT
```

### 12.3 增加转换日志

建议在 `gtp_touch_down()` 中保存并打印转换前后的坐标：

```c
s32 raw_x = input_x;
s32 raw_y = input_y;

/* 方向变换、区域平移、缩放之后 */
pr_info("gtp: raw=(%d,%d), output=(%d,%d), area=(%d,%d)-(%d,%d), res=(%d,%d)\n",
        raw_x, raw_y,
        x, y,
        ts->display_area_x_min,
        ts->display_area_y_min,
        ts->display_area_x_max,
        ts->display_area_y_max,
        ts->display_resolution_x,
        ts->display_resolution_y);
```

## 最后记住

> [!quote] 一句话总结
> 坐标处理顺序是“Goodix 原始坐标 -> `origin_position` 方向翻转 -> 减去触摸区域起点 -> 按有效区域线性缩放 -> 加 `x_offset/y_offset` -> `input_report_abs()` -> `/dev/input/eventX`”。

> [!info] 本文设备树示例
> | 项目 | 配置 |
> |---|---|
> | 原始触摸区域 | X=`53~1044`，Y=`0~600` |
> | 屏幕/input 分辨率 | `1024×600` |
> | 方向 | `RIGHT_TOP`，X 翻转 |
> | 最终输出 | X 约为 `0~1023`，Y 约为 `0~599` |

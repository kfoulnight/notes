---
title: 触摸屏驱动完整分析：GT9xx 与 MAX11801
cssclasses:
  - archive-page
---

# 触摸屏驱动完整分析：GT9xx 与 MAX11801

这份笔记基于 Linux 内核中的触摸屏驱动源码，对比分析 Goodix GT9xx 电容屏和 MAXI MAX11801 电阻屏的硬件原理、I2C 通信、坐标采样、事件上报、休眠唤醒、ESD 保护和关键数据结构。

> [!summary] 核心主线
> GT9xx 通过 I2C 读取多点坐标并经过坐标变换后上报 ABS_MT 事件；MAX11801 从 FIFO 读取单点坐标后直接上报。定位问题时，应沿着硬件、IC、总线、中断、驱动解析和 input 上报逐层验证。

> [!info] 参考源码
> | 驱动 | 文件 | 屏幕类型 |
> |---|---|---|
> | Goodix GT9xx | `drivers/input/touchscreen/gt9xx.c` | 电容屏，多点触摸 |
> | MAXI MAX11801 | `drivers/input/touchscreen/max11801_ts.c` | 电阻屏，单点触摸 |

---

## 一、电容屏 vs 电阻屏 物理原理

> [!info] 电容屏与电阻屏对比
> | 维度 | 电容屏（GT9xx） | 电阻屏（MAX11801） |
> |---|---|---|
> | 结构 | 玻璃基板与 ITO 透明导电涂层形成电容矩阵 | 两层 ITO 薄膜，中间有空气间隙 |
> | 原理 | 手指靠近改变电容，通过电容变化定位 | 手指按压使两层接触导通，通过分压测量定位 |
> | 多点能力 | 原生支持多点触摸 | 不支持多点，只有一组 X/Y 分压值 |
> | 灵敏度 | 轻触即可，无需明显压力 | 需要产生物理按压位移 |
> | 抗干扰 | 易受电源噪声、水渍和手套影响 | 抗干扰相对较强，接触物体即可触发 |
> | 线性度 | 整体较好，边缘通常需要软件补偿 | 边缘非线性明显，需要校准 |
> | 寿命 | 无机械磨损 | ITO 薄膜反复形变，寿命有限 |

电容屏传感矩阵示意：

```
         TX0   TX1   TX2  ... TXn   (驱动线, 逐行发激励信号)
    RX0  [C00] [C01] [C02] ...      每个交叉点是一个互电容
    RX1  [C10] [C11] [C12]
    RX2  [C20] [C21] [C22]
    ...                             手指靠近 → 该区域互电容 ↓
    RXm                              逐列检测 RX 信号变化 → 定位
```

IC 内部：TX 逐行发高频脉冲 → 每列 RX 同时采样 → 全矩阵电容值 → 算法提取触点坐标。

---

## 二、硬件通信与坐标采样

### GT9xx：I2C 通信

- 地址: `0x14` / `0x28` / `0x29` / `0xBA` / `0xBB` (由 RST/INT 引脚电平组合决定)
- 寄存器地址 2 字节，寄存器数据宽度不等

I2C 读 (2 条 msg 组成一个读写事务)：

```c
// gtp_i2c_read()
msgs[0]: flags = 0 (写), addr = client->addr, len = 2, buf = {REG_ADDR_H, REG_ADDR_L}
msgs[1]: flags = I2C_M_RD (读), addr = client->addr, len = N, buf = data[]
i2c_transfer(adapter, msgs, 2);  // START...REPEATED-START...STOP
```

I2C 写 (1 条 msg)：

```c
// gtp_i2c_write()
msg: flags = 0 (写), addr = client->addr, len = 2+N, buf = {REG_ADDR_H, REG_ADDR_L, data[]}
i2c_transfer(adapter, &msg, 1);
```

防抖读 (`gtp_i2c_read_dbl_check`): 读两次比较一致才返回，用于关键数据 (sensor_id, config).

重试机制: 读写失败重试 5 次，仍失败则 `gtp_reset_guitar()` 硬件复位。

握手机制 (`end_cmd`): 读完坐标后向 `0x814E` 写 0，通知 IC 数据已取走，可以刷新下一帧。

### MAX11801 — I2C (SMBus)

```c
// SMBus 快捷函数，本质仍是 i2c_transfer
i2c_smbus_read_byte_data(client, addr << 1);          // 单字节读
i2c_smbus_write_byte_data(client, addr << 1, data);   // 单字节写
i2c_smbus_read_i2c_block_data(client, FIFO_RD_CMD, len, buf);  // FIFO块读
```

### 对比: I2C vs SPI

| | I2C | SPI |
|---|---|---|
| 信号线 | 2 (SCL/SDA) | 4 (CS/SCLK/MOSI/MISO) |
| 双工 | 半双工 | 全双工 |
| 速度 | 100k/400k (标准) | 10MHz+ |
| 读方式 | 先写寄存器地址再读 | 直接全双工操作 |
| 适用场景 | 触摸坐标、config | 大吞吐固件下载 |

GT9xx 固件更新 (`gup_fw_download_proc`) 用 I2C 传输，速度受限；MAX11800 用 SPI，但 MAX11801 降级为 I2C。

---

## 三、坐标采样逻辑

### GT9xx 电容屏采样链

```
手指触摸
  → 电容矩阵变化
  → IC内部AFE逐行激励 + 逐列采样
  → ADC → 原始电容值矩阵
  → IC内DSP算法:
      1. 基线减除 (raw - baseline > threshold)
      2. 质心计算 (重心法插值，亚像素精度)
      3. 触点分离 (连通域标记，区分多点)
      4. 坐标滤波 (防抖)
  → 写入寄存器 0x814E，拉低 INT 引脚
  → Host IRQ → I2C 读取坐标
```

驱动层坐标再处理 (`gtp_touch_down` → `transform_coordinate`):

```
硬件原始坐标 (input_x, input_y)
  → transform_coordinate()   原点镜像:
      LEFT_TOP:    不变
      RIGHT_TOP:   x = x_max - x
      LEFT_DOWN:   y = y_max - y
      RIGHT_DOWN:  x = x_max - x, y = y_max - y
  → x -= x_min, y -= y_min   偏移到触摸区域坐标系
  → 线性映射到LCD分辨率:
      x = x * (display_resolution_x - 1) / (area_x_max - area_x_min) + x_offset
      y = y * (display_resolution_y - 1) / (area_y_max - area_y_min) + y_offset
  → input_report_abs()       上报
```

### GT9xx 坐标数据协议

从 `0x814E` 读出的数据布局 (每触点 8 字节):

```
字节偏移   | 内容
[0~1]     | 状态寄存器 (GTP_READ_COOR_ADDR)
[2]       | finger 标志: bit[7]=数据就绪, bit[3:0]=触点数
[3]       | 触点0: trackID (bit[3:0])
[4~5]     | 触点0: X坐标 (低8位|高8位)
[6~7]     | 触点0: Y坐标 (低8位|高8位)
[8~9]     | 触点0: 触摸面积
[10]      | 保留
[11]      | 触点1: trackID
[12~17]   | 触点1: X, Y, 面积
...
[3+8*n]   | 按键值 (所有触点之后)
```

多点数据读取:

```c
// 首次固定读 12 字节 (2状态 + 触点0完整8字节 + 2)
gtp_i2c_read(client, point_data, 12);

// touch_num > 1 时追加读取剩余触点
if (touch_num > 1) {
    gtp_i2c_read(client, buf, 2 + 8*(touch_num-1));  // 从 0x814E+10 起
    memcpy(&point_data[12], &buf[2], 8*(touch_num-1));
}
```

多点上报 — ICS_SLOT_REPORT (Type B，硬件追踪):

用 `pre_touch` 位掩码追踪每个触点的 按下/保持/抬起:

```c
for (i = 0; i < GTP_MAX_TOUCH; i++) {
    if (touch_index & (0x01 << i)) {
        gtp_touch_down(ts, id, x, y, w);   // 按下或更新
    } else {
        gtp_touch_up(ts, i);                // 抬起
    }
}
```

上报事件序列:
```
input_mt_slot()           → 切到指定slot
input_report_abs(TRACKING_ID, id)  → 绑定 slot→触点ID
input_report_abs(POSITION_X, x)
input_report_abs(POSITION_Y, y)
input_report_abs(TOUCH_MAJOR, w)
input_sync()              → SYN_REPORT，一帧结束
```

多点上报 — 非 ICS 模式 (当前生效，Type A 风格):

简化处理，只记录触点数变化:
```c
if (touch_num) {
    for (i = 0; i < touch_num; i++) {
        gtp_touch_down(ts, id, x, y, w);
    }
} else if (pre_touch) {
    gtp_touch_up(ts, 0);    // 全释放
}
pre_touch = touch_num;       // 只记触点数，不记谁是谁
```

每个触点独立 `input_mt_sync()` 分隔。不逐指追踪，靠触点数变化配合 userspace 推断。

### MAX11801 电阻屏采样链

```
手指按压
  → 两层ITO接触导通
  → IC分压测量:
      X轴: 上层加Vcc/GND → 下层测分压 → ADC_X
      Y轴: 下层加Vcc/GND → 上层测分压 → ADC_Y
  → 写入FIFO (每帧4字节带tag)
  → Host读FIFO → 直接上报
```

FIFO 帧格式 (4 字节):

```
[buf[0]] X高8位
[buf[1]] bit[3:2]=测量类型 (MEASURE_X_TAG / MEASURE_Y_TAG)
         bit[1:0]=事件类型 (EVENT_INIT/MIDDLE/RELEASE/FIFO_END)
[buf[2]] Y高8位
[buf[3]] bit[3:2]=测量类型, bit[1:0]=事件类型

X = (buf[0] << 4) + (buf[1] >> 4)
Y = (buf[2] << 4) + (buf[3] >> 4)
```

max11801 事件状态机:

```c
switch (buf[1] & EVENT_TAG_MASK) {
    EVENT_INIT:    // 首次按下 → 上报 ABS_X/Y + BTN_TOUCH=1
    EVENT_MIDDLE:  // 持续按压 → 更新坐标
    EVENT_RELEASE: // 抬起 → BTN_TOUCH=0
    EVENT_FIFO_END // FIFO为空
}
```

电阻屏不需要质心算法，分压值直接对坐标，但需要 `PANEL_SETUPTIME`、`APERTURE` 等模拟参数。

---

## 四、数据链路 (从触摸到 userspace)

| 层次 | GT9xx | MAX11801 |
|------|-------|----------|
| 物理层 | 手指改变电容矩阵 | 手指使薄膜接触导通 |
| IC层 | AFE→ADC→DSP(质心/分离/滤波)→寄存器 | 分压→ADC→FIFO |
| 中断层 | `request_irq` 顶半部关中断 + workqueue底半部 | `request_threaded_irq` 线程化 |
| 传输层 | `gtp_i2c_read(0x814E)`, 重试5次→reset | `i2c_smbus_read_block_data(FIFO)`, 无重试 |
| 解析层 | `finger&0x80` 就绪, `finger&0x0F` 触点数, 8byte/触点 | FIFO tag状态机 |
| 变换层 | 原点镜像+区域偏移+分辨率映射+x/y_offset | 无变换 |
| 上报层 | `ABS_MT_POSITION_X/Y`, `ABS_MT_TRACKING_ID`, `BTN_TOUCH`, `input_mt_sync` | `ABS_X/Y`, `BTN_TOUCH` |
| 用户态 | `/dev/input/eventX` → libinput/Android InputReader → 窗口分发 |

### GT9xx 一个完整 IRQ 周期的事件序列

```
1. IRQ触发
2. gtp_irq_disable()                          ← 关中断
3. queue_work(goodix_wq, &ts->work)
4. goodix_ts_work_func():
     gtp_i2c_read(0x814E, 12)                ← 读坐标
     解析 finger byte → touch_num
     (多点) gtp_i2c_read 追加坐标
     for each 触点:
         gtp_touch_down()
           input_report_key(BTN_TOUCH, 1)
           input_report_abs(ABS_MT_POSITION_X, x)
           input_report_abs(ABS_MT_POSITION_Y, y)
           input_report_abs(ABS_MT_TOUCH_MAJOR, w)
           input_report_abs(ABS_MT_WIDTH_MAJOR, w)
           input_report_abs(ABS_MT_TRACKING_ID, id)
           input_mt_sync()
     或 (无触摸):
         gtp_touch_up()
           input_report_key(BTN_TOUCH, 0)
     input_sync()                              ← SYN_REPORT
     gtp_i2c_write(0x814E, 0)                 ← end_cmd通知IC读完
     gtp_irq_enable()                          ← 重开中断
```

### 其他上报路径

触摸按键 (`GTP_HAVE_TOUCH_KEY = 1`):

按键值位于坐标数据尾部的 `point_data[3+8*touch_num]`:
```c
for (i = 0; i < ts->key_nums; i++) {
    input_report_key(ts->input_dev, ts->key_map[i], key_value & (0x01 << i));
}
touch_num = 0;  // 有按键事件时屏蔽触摸坐标
```

手势唤醒 (`GTP_GESTURE_WAKEUP = 0`, 当前关闭):

读 `0x814B` 匹配手势字符 (a~z, ^) 或滑动方向 (0xAA/0xBB/0xAB/0xBA) 或双击 (0xCC):

```c
input_report_key(ts->input_dev, KEY_POWER, 1);
input_sync(ts->input_dev);
input_report_key(ts->input_dev, KEY_POWER, 0);
input_sync(ts->input_dev);
// 手势唤醒在 work_func 前半段就 return，不走到坐标处理
```

主动笔 (`GTP_WITH_PEN = 0`, 当前关闭):

走独立的 `pen_dev` 设备，区别于手指的 `input_dev`:
```c
input_report_key(ts->pen_dev, BTN_TOOL_PEN, 1);
input_mt_slot(ts->pen_dev, id);
input_report_abs(ts->pen_dev, ABS_MT_POSITION_X, x);
input_report_abs(ts->pen_dev, ABS_MT_POSITION_Y, y);
input_report_abs(ts->pen_dev, ABS_MT_PRESSURE, w);
input_sync(ts->pen_dev);
```

---

## 五、控制链路

### 驱动加载

```
module_init(goodix_ts_init)
  → create_singlethread_workqueue("goodix_wq")      ← 主数据workqueue
  → INIT_DELAYED_WORK(&gtp_esd_check_work, ...)      ← ESD延迟work
  → create_workqueue("gtp_esd_check")                ← ESD独立workqueue
  → i2c_add_driver(&goodix_ts_driver)
  → goodix_ts_probe():
      ├─ gtp_parse_dt()              设备树解析 (irq/rst gpio, 坐标区域,
      │                                分辨率, 偏移, esd, key_map)
      ├─ gtp_power_switch(1)         拉电 (regulator vdd_ana + vcc_i2c)
      ├─ gtp_request_io_port()       GPIO申请 + gtp_reset_guitar()
      ├─ gtp_get_chip_type()         读0x8000区分 GT9/GT9F
      ├─ gtp_read_version()          读0x8140固件版本
      ├─ gtp_init_panel()            读sensor_id→选cfg_group→
      │                               checksum→gtp_send_cfg()
      ├─ gtp_request_input_dev()     注册input设备
      ├─ gtp_request_irq()           request_irq或fallback到hrtimer轮询
      ├─ gtp_esd_switch(SWITCH_ON)  启动ESD看门狗
      └─ gtp_register_powermanger()  fb notifier / PM ops / earlysuspend
```

### 休眠/唤醒

```
休眠 (3种触发方式):
  FB_BLANK_POWERDOWN / PM.suspend / early_suspend
    → goodix_ts_suspend()
        ├─ gtp_esd_switch(SWITCH_OFF)    停ESD
        ├─ [手势唤醒] gtp_enter_doze()   写0x8046=5→写0x8040=8
        └─ [普通休眠] gtp_enter_sleep()  INT拉低→写0x8040=5

唤醒:
  FB_BLANK_UNBLANK / PM.resume / early_resume
    → goodix_ts_resume()
        ├─ gtp_wakeup_sleep()   reset→I2C测试→gtp_send_cfg
        ├─ gtp_irq_enable()     恢复中断
        └─ gtp_esd_switch(ON)   恢复ESD
```

### ESD 保护 (独立 workqueue 并行运行)

```
gtp_esd_check_workqueue (延迟work, 默认周期 5s)
  → gtp_esd_check_func()
      ├─ 读 0x8040/0x8041 (3次确认防误判)
      ├─ 0x8040==0xAA 或 0x8041!=0xAA → IC异常
      │   ├─ GT9F: gtp_esd_recovery()  重下固件 + fw_startup
      │   └─ GT9:  gtp_reset_guitar() + gtp_send_cfg()
      ├─ 正常: 写0x8040=0xAA (喂狗)
      └─ queue_delayed_work() 重新调度，生命周期直到 suspend
```

ESD开关 (`gtp_esd_switch`):

```c
// SWITCH_ON:  ts->esd_running=1 → queue_delayed_work()
// SWITCH_OFF: ts->esd_running=0 → cancel_delayed_work_sync()
// 用 esd_lock 保护 esd_running 标志
```

### 固件更新

```
sysfs dofwupdate 写入文件名
  → gup_update_proc(file_name)
  → gup_fw_download_proc(NULL, GTP_FL_FW_BURN)
  → I2C 逐块写入固件
  → gtp_reset_guitar()
  → gtp_fw_startup()
  → gtp_send_cfg()  重新下发配置
```

### 运行时调试接口

| 接口 | 路径 | 读写 | 功能 |
|------|------|------|------|
| procfs | `/proc/gt9xx_config` | R/W | 读写 config 寄存器表 |
| sysfs | `dofwupdate` | W | 固件更新 |
| sysfs | `productinfo` | R | 读固件 PID + 版本号 |

---

## 六、关键寄存器

| 地址 | 宏 | 用途 |
|------|-----|------|
| `0x814E` | `GTP_READ_COOR_ADDR` | 坐标数据寄存器 (读+写0清) |
| `0x8047` | `GTP_REG_CONFIG_DATA` | Config 配置数据 |
| `0x8040` | `GTP_REG_SLEEP` | 休眠控制 / ESD 看门狗 |
| `0x8041` | — | ESD 看门狗状态 |
| `0x8140` | `GTP_REG_VERSION` | 固件版本 |
| `0x814A` | `GTP_REG_SENSOR_ID` | Sensor ID (0~5) |
| `0x814B` | — | 手势唤醒字符寄存器 |
| `0x8000` | `GTP_REG_CHIP_TYPE` | 芯片类型 "GOODIX_GT9" / "GOODIX_GT9F" |
| `0x8046` | — | 手势 doze 模式控制 |
| `0x8043` | — | GT9XXF 请求状态 |
| `0x804E` | `GTP_REG_HAVE_KEY` | 是否有触摸按键 |
| `0x8069` | `GTP_REG_MATRIX_DRVNUM` | 驱动线数量 |
| `0x806A` | `GTP_REG_MATRIX_SENNUM` | 感应线数量 |

---

## 七、核心数据结构

### `struct goodix_ts_data` ([gt9xx.h:101-160](gt9xx.h#L101-L160))

```c
struct goodix_ts_data {
    spinlock_t irq_lock;           // 中断使能/禁止自旋锁
    struct i2c_client *client;     // I2C 客户端
    struct input_dev  *input_dev;  // input 设备 (触摸)
    struct hrtimer timer;          // 轮询模式高精度定时器
    struct work_struct  work;      // 中断底半部 work
    s32 irq_is_disable;            // 中断是否已关闭
    s32 use_irq;                   // 是否使用中断模式 (否则轮询)
    u16 abs_x_max;                 // 触摸物理 X 最大值
    u16 abs_y_max;                 // 触摸物理 Y 最大值
    u8  max_touch_num;             // 最大触点数
    u8  int_trigger_type;          // 中断触发类型 (上升/下降/高/低)
    u8  enter_update;              // 固件升级中标志
    u8  gtp_is_suspend;            // 休眠中标志
    u8  gtp_rawdiff_mode;          // raw diff 调试模式
    int gtp_cfg_len;               // config 长度
    u8  fw_error;                  // 固件错误标志
    u8  pnl_init_error;            // panel 初始化错误
    int x_offset, y_offset;        // 坐标偏移校准
    int display_area_x_min, y_min; // 触摸区域原点
    int display_area_x_max, y_max; // 触摸区域边界
    int display_resolution_x, y;   // LCD 显示分辨率
    int origin_position;           // 原点位置 (左下/右下/左上/右上)
    int display_area_x_xingwei;    // 兴为TP定制: 触摸区域X固定值
    int tp_compatible;             // TP兼容模式
    int esd_protect;               // ESD保护使能
    u32 driver_send_cfg;           // 驱动下发config使能
    u32 key_map[MAX_KEY_NUMS];     // 触摸按键映射表
    u32 key_nums;                  // 触摸按键数量
    // 休眠管理 (三选一):
    // CONFIG_FB:     struct notifier_block notifier
    // CONFIG_HAS_EARLYSUSPEND: struct early_suspend early_suspend
    // CONFIG_PM:     (无额外字段, 用dev_pm_ops)
    struct input_dev *pen_dev;     // 主动笔独立 input 设备
    spinlock_t esd_lock;           // ESD状态锁
    u8  esd_running;               // ESD运行标志
    s32 clk_tick_cnt;              // ESD周期 (5*HZ)
    struct goodix_fw_info fw_info; // 固件信息
    // GT9XXF 扩展:
    u16 bak_ref_len;               // backup reference 长度
    s32 ref_chk_fs_times;          // ref 文件系统检查次数
    s32 clk_chk_fs_times;          // clock 文件系统检查次数
    CHIP_TYPE_T chip_type;         // CHIP_TYPE_GT9 / CHIP_TYPE_GT9F
    u8 rqst_processing;            // 正在处理请求标志
    u8 is_950;                     // 是否 GT9xx 950 型号
};
```

### `struct goodix_fw_info` ([gt9xx.h:95-99](gt9xx.h#L95-L99))

```c
struct goodix_fw_info {
    u8 pid[7];         // 产品ID 字符串 (3或4字符)
    u32 version;       // 固件版本号
    u8 sensor_id;      // 传感器ID (0~5, 由硬件引脚决定)
};
```

### `struct max11801_data` ([max11801_ts.c:82-85](max11801_ts.c#L82-L85))

```c
struct max11801_data {
    struct i2c_client *client;     // I2C 客户端
    struct input_dev  *input_dev;  // input 设备
};
```

---

## 八、重要函数

### GT9xx 核心函数

| 函数 | 行号 | 层次 | 作用 |
|------|------|------|------|
| `gtp_i2c_read()` | [126](gt9xx.c#L126) | 传输层 | I2C读 (写reg addr + 读data) |
| `gtp_i2c_write()` | [194](gt9xx.c#L194) | 传输层 | I2C写 (reg addr + data) |
| `gtp_i2c_read_dbl_check()` | [254](gt9xx.c#L254) | 传输层 | I2C两次读比较防抖 |
| `gtp_send_cfg()` | [292](gt9xx.c#L292) | 控制层 | 下发 config 表到 IC (0x8047) |
| `gtp_reset_guitar()` | [1107](gt9xx.c#L1107) | 控制层 | 硬件复位时序 (RST拉低→INT拉低→RST高→INT同步) |
| `gtp_irq_enable()` / `gtp_irq_disable()` | [350](gt9xx.c#L350)/[327](gt9xx.c#L327) | 控制层 | 中断开关 (spin_lock + enable/disable_irq) |
| `goodix_ts_irq_handler()` | [1071](gt9xx.c#L1071) | 中断层 | 中断顶半部: 关中断 + queue_work |
| `goodix_ts_timer_handler()` | [1050](gt9xx.c#L1050) | 中断层 | 轮询模式 hrtimer 回调 |
| `goodix_ts_work_func()` | [573](gt9xx.c#L573) | 数据核心 | 底半部: 读坐标→解析→上报→sync→end_cmd |
| `gtp_touch_down()` | [403](gt9xx.c#L403) | 上报层 | 单点按下上报 (坐标变换 + input_report_*) |
| `gtp_touch_up()` | [456](gt9xx.c#L456) | 上报层 | 单点抬起上报 |
| `transform_coordinate()` | [365](gt9xx.c#L365) | 变换层 | 原点镜像变换 |
| `gtp_init_panel()` | [1373](gt9xx.c#L1373) | 初始化 | sensor_id→cfg→checksum→send_cfg |
| `gtp_request_input_dev()` | [1956](gt9xx.c#L1956) | 初始化 | 分配+注册 input 设备 |
| `gtp_request_irq()` | [1915](gt9xx.c#L1915) | 初始化 | 注册中断或 fallback 到 hrtimer |
| `goodix_ts_probe()` | [2764](gt9xx.c#L2764) | 初始化 | 驱动 probe 入口 |
| `goodix_ts_suspend()` | [2982](gt9xx.c#L2982) | 电源 | 休眠 (ESD停 + doze/sleep) |
| `goodix_ts_resume()` | [3027](gt9xx.c#L3027) | 电源 | 唤醒 (wakeup + send_cfg + ESD开) |
| `gtp_esd_check_func()` | [3306](gt9xx.c#L3306) | ESD | 周期性检查 IC 健康，异常则恢复 |
| `gtp_esd_switch()` | [3246](gt9xx.c#L3246) | ESD | ESD 启停 (queue/cancel delayed_work) |
| `gtp_esd_recovery()` | [2083](gt9xx.c#L2083) | ESD | GT9XXF: 重下固件 + fw_startup |
| `gtp_recovery_reset()` | [2122](gt9xx.c#L2122) | ESD | GT9XXF: ESD关→recovery→ESD开 |
| `gtp_enter_sleep()` | [1190](gt9xx.c#L1190) | 电源 | 写 0x8040=5 进休眠 |
| `gtp_wakeup_sleep()` | [1250](gt9xx.c#L1250) | 电源 | reset + I2C测试 + send_cfg |
| `gtp_enter_doze()` | [1147](gt9xx.c#L1147) | 电源 | 手势唤醒 doze 模式 |
| `gtp_parse_dt()` | [2483](gt9xx.c#L2483) | 初始化 | 设备树解析 |
| `gtp_parse_dt_cfg()` | [2626](gt9xx.c#L2626) | 初始化 | 从设备树解析 config 配置 |
| `gtp_power_switch()` | [2647](gt9xx.c#L2647) | 控制层 | regulator 电源开关 |
| `gtp_bak_ref_proc()` | [2138](gt9xx.c#L2138) | GT9XXF | backup-reference 读写/存储 |
| `gtp_main_clk_proc()` | [2329](gt9xx.c#L2329) | GT9XXF | 主时钟校准 |

### MAX11801 核心函数

| 函数 | 行号 | 作用 |
|------|------|------|
| `max11801_ts_interrupt()` | [99](max11801_ts.c#L99) | 线程化中断: 读FIFO→解析状态机→上报 |
| `max11801_ts_phy_init()` | [158](max11801_ts.c#L158) | 寄存器初始化 (采样/panel/auto mode) |
| `max11801_ts_probe()` | [176](max11801_ts.c#L176) | 驱动 probe 入口 |

---

## 九、架构差异总结

> [!info] GT9xx 与 MAX11801 对比
> | 维度 | GT9xx | MAX11801 |
> |---|---|---|
> | 屏类型 | 电容，多点（最多 5 点） | 电阻，单点 |
> | 数据结构复杂度 | 40+ 字段 | 2 个核心字段 |
> | 代码量 | 约 3500 行 | 约 240 行 |
> | 中断模式 | `request_irq` 加 workqueue 底半部 | `request_threaded_irq` 线程化中断 |
> | 上报协议 | `ABS_MT_*`，Type A/B 风格 | `ABS_X/Y` 单点 |
> | 数据读取 | 首次读取 12 字节，多点时追加读取 | 固定 4 字节 FIFO |
> | 握手机制 | `end_cmd` 写 0 确认读取完成 | FIFO 自带状态机 |
> | 坐标变换 | 镜像、偏移、映射和 `x/y_offset` | 无 |
> | ESD 保护 | 独立 workqueue，默认 5 秒周期 | 无 |
> | 休眠/唤醒 | FB notifier、PM ops 或 earlysuspend | 无 |
> | 固件更新 | proc/sysfs 触发，通过 I2C 分块写入 | 无 |
> | 配置下发 | 设备树、头文件和 proc 三条路径 | 寄存器硬编码初始化 |
> | 手势唤醒 | 支持 `GTP_GESTURE_WAKEUP` | 无 |
> | 触摸按键 | 支持 `GTP_HAVE_TOUCH_KEY` | 无 |
> | 主动笔 | 支持 `GTP_WITH_PEN`，独立 `pen_dev` | 无 |
> | GT9F 兼容 | 支持 backup-reference 和 clock | 无 |

GT9xx 是一个覆盖固件管理、ESD 保护、手势唤醒和多点上报的完整工业级电容屏驱动。MAX11801 则是一个结构更简单的电阻屏驱动，核心路径是“线程化中断读取 FIFO，再直接上报事件”。

## 最后记住

> [!quote] 一句话总结
> GT9xx 的调试重点是坐标寄存器读取、多点 slot/tracking id、坐标变换、ESD 和电源管理；MAX11801 的调试重点是 FIFO 数据格式、事件状态机和模拟采样参数。遇到触摸异常时，先确认 IC 是否产生数据，再确认驱动是否正确解析，最后检查 input 事件是否完整到达用户态。

# 车载以太网TJA1120/1103 PHY 开发文档

> 平台：RK3576 (Android / Linux)
> 适用范围：以太网 PHY 驱动开发、移植、调试

---

## 目的

本文档面向**基于 RK3576 SoC + NXP TJA1103 / TJA1120 车载以太网 PHY** 的软硬件开发场景，系统讲解以太网 PHY 的开发方法。

主要内容与目标：

- 介绍以太网基础概念，明确 PHY 在 OSI 模型中所处的物理层位置及其与 MAC 的协作关系；
- 梳理 RK3576 GMAC 与 TJA1103 / TJA1120 的硬件连接与接口配置；
- 讲解 PHY 寄存器、Auto-Negotiation（自动协商）等核心机制；
- 结合 Linux/Android 内核 phylib 驱动框架，说明 PHY 驱动的实现、移植与调试方法；
- 汇总开发过程中常见问题的定位与解决办法。

适用读者：

- 负责 RK3576 平台以太网功能开发、移植、调试的**驱动工程师**；
- 需要了解 PHY 硬件接口与外围电路设计的**硬件工程师**；
- 参与车载以太网链路联调与问题定位的**测试/系统工程师**。

阅读前提：建议读者对 C 语言、Linux 设备驱动模型、网络通信、I2C/MDIO 类总线通信有基本了解。

---

## 目录

1. [网络基础与 PHY 概述](#1-网络基础与-phy-概述)
   - 1.1 OSI 七层模型简介
   - 1.2 PHY 所处的层级
   - 1.3 PHY 与 MAC、MDIO 的关系
2. [RGMII 与 MDIO 接口](#2-rgmii-与-mdio-接口)
   - 2.1 RGMII 数据接口
   - 2.2 MDIO 管理接口
3. [RK3576 GMAC 设备树配置](#3-rk3576-gmac-设备树配置)
   - 3.1 节点概述与示例
   - 3.2 phy-mode
   - 3.3 PHY 复位配置（snps,reset-\*）
   - 3.4 phy-supply
   - 3.5 时钟配置
   - 3.6 pinctrl
   - 3.7 tx_delay / rx_delay
   - 3.8 参考文档
4. [TJA1120 pin strap 配置](#4-tja1120-pin-strap-配置)
   - 4.1 strap 引脚与功能映射
   - 4.2 三态 strap 与外部电阻
   - 4.3 采样检测逻辑
   - 4.4 PHY 地址配置（CONFIG0/1/2）
   - 4.5 xMII 模式配置（CONFIG3/4）
   - 4.6 时钟模式配置（CONFIG5）
   - 4.7 Leader/Follower 与 Autonomous（CONFIG6）
   - 4.8 strap 采样时机与替代功能
   - 4.9 strap 配置检查清单
5. [NXP C45 PHY 驱动结构（struct phy_driver nxp_c45_driver[]）](#5-nxp-c45-phy-驱动结构struct-phy_driver-nxp_c45_driver)
   - 5.1 文件与驱动框架定位
   - 5.2 struct phy_driver 数组结构
   - 5.3 字段分组详解
   - 5.4 两个表项（TJA1103 / TJA1120）对比
   - 5.5 driver_data 芯片私有数据
   - 5.6 注册与匹配（module_phy_driver）
6. [PHY 驱动架构与网卡 Link Up 流程](#6-phy-驱动架构与网卡-link-up-流程)
   - 6.1 分层架构总览
   - 6.2 核心对象与数据流
   - 6.3 phylib 状态机
   - 6.4 Link Up 完整流程
   - 6.5 结合本项目（TJA1120 / RK 改动）
7. [调试流程](#7-调试流程)
   - 7.1 检查硬件配置：晶振、电源、时钟
   - 7.2 MDIO 探测 PHY 与驱动匹配
   - 7.3 Link Up 检查
   - 7.4 Loopback 测试
   - 7.5 端对端连接测试
   - 7.6 RGMII Delay 调试
8. [常见问题](#8-常见问题)
   - 8.1 log 提示 PHY ID 不匹配
   - 8.2 提示 DMA 初始化失败
   - 8.3 RX 无法收包异常

---

# 1. 网络基础与 PHY 概述

## 1.1 OSI 七层模型简介

OSI（Open Systems Interconnection，开放系统互联）参考模型由 ISO 提出，将计算机网络通信划分为七个层次。每一层只负责其特定的功能，并为相邻的上一层提供服务，从而把复杂的网络通信问题分解为若干相对简单的子问题。

| 层号 | 名称 | 核心职责 | 典型协议/设备 |
|------|------|----------|--------------|
| 7 | 应用层 (Application) | 为用户提供网络应用服务接口 | HTTP、FTP、DNS、SMTP |
| 6 | 表示层 (Presentation) | 数据格式转换、加密、压缩 | TLS、JPEG、ASCII |
| 5 | 会话层 (Session) | 建立、管理、终止会话 | RPC、NetBIOS |
| 4 | 传输层 (Transport) | 端到端可靠传输，端口寻址 | TCP、UDP |
| 3 | 网络层 (Network) | 逻辑寻址（IP 地址）、路由选择 | IP、ICMP、路由器 |
| 2 | 数据链路层 (Data Link) | 帧封装/解封装、物理寻址（MAC）、差错检测 | Ethernet、网卡 MAC、交换机 |
| 1 | 物理层 (Physical) | 比特流的传输，电气/光/无线信号，介质与接口 | 网线、光模块、**PHY 芯片** |

发送数据时数据从上往下逐层封装；接收数据时从下往上逐层解封装。

> 注意：OSI 模型是理论参考模型，实际以太网遵循的是 TCP/IP 四层模型（应用层 / 传输层 / 网络层 / 网络接口层），但 OSI 模型用于理解"PHY 处于什么位置"仍然最清晰直观。

## 1.2 PHY 所处的层级

**PHY（Physical Layer Transceiver，物理层收发器）位于 OSI 模型的第一层——物理层。**

在以太网体系中，"网卡"在逻辑上由两部分组成：

- **MAC（Media Access Control，介质访问控制）**：位于数据链路层（第 2 层），负责帧的封装与解封装、MAC 寻址、CSMA/CD、流量控制等，通常集成在 SoC 内部。
- **PHY**：位于物理层（第 1 层），负责真正的"信号搬运"，把 MAC 发来的数据转换成能在物理介质（网线、光纤）上传输的电/光信号，并把从介质上收到的信号还原成数据交给 MAC。

通俗理解：**MAC 管"帧"，PHY 管"比特和信号"**。PHY 是 MAC 与物理介质之间的桥梁。

PHY 内部又按功能划分为多个子层（IEEE 802.3 定义）：

| 子层 | 全称 | 主要功能 |
|------|------|----------|
| PCS | Physical Coding Sublayer（物理编码子层） | 数据编码/解码，如 4B/5B、8B/10B 编码，负责从 MAC 传来的并行数据到串行比特流的转换 |
| PMA | Physical Medium Attachment（物理介质接入子层） | 串行化/解串行化，扰码/解扰，时钟恢复 |
| PMD | Physical Medium Dependent（物理介质相关子层） | 驱动/接收物理介质上的信号，完成最终的电/光信号发送与接收 |

PHY 芯片对外还需提供几个关键能力：

- **Auto-Negotiation（自动协商）**：上电后与对端设备协商出双方都支持的最高工作速率（10/100/1000 Mbps）和双工模式（半双工/全双工）。
- **Link 检测**：实时检测物理链路是否建立（对端是否在线、线路是否连通），并向 MAC 上报 Link 状态。
- **MDI/MDIX 自动交叉**：自动切换收发线对，使直通线与交叉线均可直接使用（注：车载 100/1000BASE-T1 为单对双绞线，其极性/线序处理与多对 802.3 以太网不同）。
- **隔离与驱动**：提供变压器/电平转换等，驱动差分信号在双绞线上传输。

## 1.3 PHY 与 MAC、MDIO 的关系

在 RK3576 等 SoC 的以太网系统中，典型连接关系如下：

```
                    ┌─────────────────────────────┐
                    │            SoC (RK3576)       │
                    │                              │
                    │   ┌─────────┐    ┌────────┐  │
                    │   │  GMAC   │    │  MDIO  │  │
                    │   │ (MAC)   │◄──►│ Master │  │
                    │   └────┬────┘    └───┬────┘  │
                    └────────┼─────────────┼───────┘
                             │ RGMII/SGMII  │ MDIO/MDC
                             │ (数据接口)    │ (管理接口)
                             ▼             ▼
                    ┌─────────────────────────────┐
                    │     TJA1103 / TJA1120       │
                    │     (车载以太网 PHY)         │
                    └─────────────┬───────────────┘
                                 │ 单对双绞线
                                 │ (100/1000BASE-T1)
                                 ▼
                        H-MTD 车载连接器
                                 │
                                 ▼
                          对端 ECU 设备
```

- **数据接口（RGMII / SGMII 等）**：负责 MAC 与 PHY 之间数据面（数据收发）的传输。本文档涉及的 TJA1103（100BASE-T1）/ TJA1120（1000BASE-T1）车载 PHY 典型配置为 RGMII 或 SGMII。
- **管理接口（MDIO/MDC，也称 SMI）**：由 MAC 作为 MDIO Master，通过读写 PHY 内部寄存器实现对 PHY 的配置与状态查询。PHY 寄存器是驱动开发的核心对象（详见后续章节）。

> 开发以太网 PHY 的本质工作就是：**通过 MDIO 操作 PHY 的寄存器，配合内核 phylib 驱动框架，使 PHY 完成初始化、自动协商、链路监测、数据收发等功能。**

---

（本节完）

---

# 2. RGMII 与 MDIO 接口

MAC 与 PHY 之间有**两条独立通路**：

- **数据通路**：承载网络数据帧的收发，即 RGMII / SGMII 等介质无关接口；
- **管理通路**：读写 PHY 寄存器，即 MDIO 接口。

本节分别介绍 RGMII 与 MDIO。

## 2.1 RGMII 数据接口

### 2.1.1 介质无关接口（MII）族概述

MAC 与 PHY 之间的数据接口统称为"介质无关接口"，因为它与具体的传输介质（双绞线、光纤、单对车载线缆）无关。常见接口对比如下：

| 接口 | 速率范围 | 数据宽度 | 采样方式 | 发送时钟 | 引脚数 |
|------|----------|----------|----------|----------|--------|
| MII | 10/100 Mbps | 4-bit | 单沿 (SDR) | 25/2.5 MHz | ~16 |
| RMII | 10/100 Mbps | 2-bit | 单沿 (SDR) | 50 MHz | ~8 |
| GMII | 1000 Mbps | 8-bit | 单沿 (SDR) | 125 MHz | ~24 |
| **RGMII** | **10/100/1000 Mbps** | **4-bit** | **双沿 (DDR)** | **125/25/2.5 MHz** | **~12** |
| SGMII | 10/100/1000 Mbps | 1-bit 串行 | 串行 | 1.25 GHz | ~4（差分对） |

RK3576 的 GMAC 与 TJA1103 / TJA1120 之间最常见的连接方式是 **RGMII**（千兆下用 4 位数据线、双沿采样，引脚数比 GMII 少一半）。

### 2.1.2 RGMII 引脚定义

RGMII 信号分为发送（TX）、接收（RX）和时钟三组：

| 信号 | 方向 (MAC 视角) | 说明 |
|------|------------------|------|
| TXD[3:0] | 输出 | 发送数据 |
| TX_CTL | 输出 | 发送控制：上升沿 = TX_EN（发送使能），下降沿 = TX_ER（发送错误） |
| GTX_CLK | 输出 | 发送时钟：1000M = 125 MHz，100M = 25 MHz，10M = 2.5 MHz |
| RXD[3:0] | 输入 | 接收数据 |
| RX_CTL | 输入 | 接收控制：上升沿 = RX_DV（接收数据有效），下降沿 = RX_ER（接收错误） |
| RX_CLK | 输入 | 接收时钟，由 PHY 从链路上恢复后提供给 MAC（125/25/2.5 MHz） |

> 发送数据线的名称在不同芯片资料中略有差异（如 MAC 侧可能写作 TXD，PHY 侧写作 RXD），判断方向时以"PHY 角度"为准即可。

### 2.1.3 RGMII 的传输原理

- **双沿采样（DDR）**：数据与 RX_CTL 在时钟的**上升沿和下降沿各采样一次**，4 位数据线每时钟周期等效传输 8 位，因此 125 MHz 时钟即对应 1000 Mbps 线速。
- **时钟由谁产生**：
  - 发送方向：**MAC 产生 GTX_CLK** 并随 TXD 一起送给 PHY；
  - 接收方向：**PHY 从接收到的串行数据中恢复出 RX_CLK**，连同 RXD 一起送给 MAC。
- **时钟/数据对齐（skew）问题**：RGMII 规范中数据相对时钟存在约 1.5~2 ns 的建立/保持时间约束。实际设计中常通过以下方式解决，否则高速下会出现错位误码：
  - 发送方向：由 **MAC 内部 TX delay**（如 RK3576 GMAC 的 tx-delay 配置）或 **PHY 内部 RX delay** 提供约 2 ns 延迟；
  - 接收方向：由 **MAC 内部 RX delay** 或 **PHY 内部 TX delay** 提供补偿；
  - 注意：MAC 与 PHY 两侧的 delay 只能**启用其中一侧**，若两边都加会超时、都不加则不满足时序。

> RK3576 平台在设备树中通过 `tx_delay` / `rx_delay` 属性（配合 PHY 内部寄存器）来配置该延迟，具体见第 4、5 章驱动与移植部分。

### 2.1.4 RK3576 GMAC 与 RGMII 注意事项

- RK3576 内部集成千兆 GMAC，RGMII 引脚一般为**复用引脚**，需在设备树 `pinctrl` 中正确配置。
- 如果 MAC 与 PHY 之间的 RGMII 走线较长，或使用了电平转换芯片，需要特别关注时钟与数据延迟匹配。
- 若选用 **SGMII** 连接，则引脚更少（串行差分对），但需要额外的时钟源或参考时钟配置，配置复杂度更高，常用于对引脚数量敏感的板卡。

## 2.2 MDIO 管理接口

### 2.2.1 概述

MDIO（Management Data Input/Output，管理数据输入输出，也称 **SMI**，Serial Management Interface）是 MAC 与 PHY 之间的**管理通路**，用于对 PHY 内部寄存器进行读写，实现速率/双工配置、自动协商控制、链路状态查询、中断状态读取等。**PHY 驱动开发的几乎所有操作都通过 MDIO 完成。**

MDIO 总线只有两根信号线：

| 信号 | 方向 | 说明 |
|------|------|------|
| MDC | MAC → PHY | 管理时钟，由 MAC 产生，Clause 22 规范下最高 2.5 MHz |
| MDIO | MAC ↔ PHY | 双向数据线，开漏输出，需要上拉电阻 |

MAC 侧的 MDIO 控制器称为 **MDIO Master（或 STA，Station Management Entity）**，PHY 作为 **MDIO Slave**。一条 MDIO 总线上可以挂接多个 PHY，通过 **5-bit PHY 地址**区分（0~31）。

### 2.2.2 标准帧格式（IEEE 802.3 Clause 22）

Clause 22 是最常用的 MDIO 帧格式，一帧固定读写 16-bit 寄存器：

```
| Preamble  | ST   | OP   | PHYAD  | REGAD  | TA    | DATA     |
| 32×"1"    | 2b01 | 2b   | 5 bit  | 5 bit  | 2 bit | 16 bit   |
```

| 字段 | 位宽 | 说明 |
|------|------|------|
| Preamble | 32 bit | 前导码，全 1；总线空闲超过一段时间后可缩短为 16 bit |
| ST | 2 bit | 起始码，固定 `01` |
| OP | 2 bit | 操作码：`10` = 读，`01` = 写 |
| PHYAD | 5 bit | PHY 地址，与硬件 strap 引脚一致 |
| REGAD | 5 bit | 寄存器地址，0~31 |
| TA | 2 bit | 转向时间：读操作时高阻释放（`Z0`），写操作为 `10` |
| DATA | 16 bit | 读/写的寄存器数据 |

Clause 22 只能访问 **32 个通用寄存器**，其中 0~15 为 IEEE 802.3 标准定义，16~31 为厂商自定义。

### 2.2.3 扩展帧格式（IEEE 802.3 Clause 45）

现代 PHY（包括车载以太网 PHY，如 TJA1103 / TJA1120）寄存器功能远超 32 个，因此引入 **Clause 45** 帧格式：增加 5-bit **DEVAD（设备地址，MMD）** 和 16-bit 寄存器地址，可访问更多寄存器空间（每个 DEV 下有 65536 个 16-bit 寄存器）。

```
| Preamble | ST   | OP   | DEVAD | REGAD     | TA    | DATA     |
| 32×"1"   | 2b00 | 2b   | 5 bit | 16 bit    | 2 bit | 16 bit   |
```

> **注意：Clause 45 帧中没有 PHYAD 字段**，这是 IEEE 802.3 的有意设计。Clause 45 面向**点到点 MDIO 连接**：每颗 PHY 使用独立的 MDC/MDIO 线与 MAC 相连，一段总线上只有一颗 PHY，因此帧内无需再携带"选择哪颗 PHY"的地址；帧中的 **DEVAD** 用于选择该 PHY 内部的 **MMD 寄存器块**。若确需多颗 Clause 45 PHY 共用一条总线，硬件上必须分多段（各自独立 MDC/MDIO）或加总线切换器，帧格式本身不支持同总线多 PHY。

Clause 45 的寄存器组织为 **MMD（MDIO Manageable Device）** 结构，如 MMD0（控制）、MMD1（状态）、MMD3（诊断）等。驱动中需要区分当前 PHY 使用 Clause 22 还是 Clause 45，TJA1103 / TJA1120 的核心配置寄存器主要位于 **MMD 空间**。

> Linux 的 phylib 中，Clause 45 访问通过 `mdio_bus` 的 `read_mmd` / `write_mmd` 回调实现；对老式仅支持 Clause 22 的 PHY 则不适用，需确认芯片数据手册。

### 2.2.4 PHY 地址的确定

PHY 地址不是软件随意指定的，而是由 **PHY 芯片外部引脚（strap 配置）** 在复位时采样决定的，常见取值如 `0x01`、`0x03` 等。驱动开发时必须：

1. 查看硬件原理图中 PHY 地址 strap 引脚的接法，确定实际地址；
2. 在设备树 / 驱动中填入一致的 PHY 地址；
3. 若一条 MDIO 总线上挂多个 PHY，地址必须互不相同。

### 2.2.5 与 Linux 驱动的对应关系

Linux 内核通过 **mii_bus（mdio_bus）** 框架管理 MDIO 总线与 PHY 设备：

- MAC 驱动的 `mdio_bus` 注册后，内核会自动扫描每个 PHY 地址，读取其寄存器进行枚举；
- 识别出的每个 PHY 对应一个 `phy_device`，再匹配对应的 **PHY 驱动**（phylib）做进一步初始化；
- 驱动中读写 PHY 寄存器的入口，最终都会落到 `mdiobus_read` / `mdiobus_write`（或 mmd 版本）。

### 2.2.6 读写mido的工具

用AI写了个可以读写mido接口工具，放在附件。

具体框架结构、驱动编写与调试，详见第 4、5 章。

---

（本章完）

---

# 3. RK3576 GMAC 设备树配置

RK3576 的 GMAC（内部集成 MAC）与外部 PHY 的初始化、引脚、时钟、复位等行为，全部通过设备树（Device Tree）节点配置。本节说明设备树中与 PHY 驱动开发直接相关的关键属性。

## 3.1 节点概述与示例

RK3576 平台通常提供 `gmac0`、`gmac1` 两个 GMAC 节点，使用时在板级设备树中使能（`status = "okay"`）并填充 PHY 相关配置。以 RGMII + TJA1103/1120 为例的典型节点如下：

```dts
&gmac0 {
	/* Use rgmii-rxid mode to disable rx delay inside Soc */
	phy-mode = "rgmii-id";
	clock_in_out = "output";

	snps,reset-gpio = <&gpio3 RK_PC7 GPIO_ACTIVE_HIGH>;
	snps,reset-active-high;
	snps,reset-delays-us = <0 20000 100000>;

	pinctrl-names = "default";
	pinctrl-0 = <&eth0m0_miim
		     &eth0m0_tx_bus2
		     &eth0m0_rx_bus2
		     &eth0m0_rgmii_clk
		     &eth0m0_rgmii_bus>;

	//tx_delay = <0x20>;
	//rx_delay = <0x3f>;

	phy-handle = <&rgmii_phy0>;

	status = "okay";
};

&mdio0 {
	rgmii_phy0: phy@1 {
		compatible = "ethernet-phy-ieee802.3-c45";
		reg = <0x1>;
		//clocks = <&cru REFCLKO25M_GMAC0_OUT>;
	};
};

```

> 注意：`eth0m0_miim` / `eth0m0_tx_bus2` 等 pinctrl 组名称、GPIO 号均为示意，具体以 RK3576 对应内核版本和硬件原理图为准。RMII 与 RGMII 的引脚组名也不同（见 3.6 节）。

## 3.2 phy-mode

`phy-mode` 用于告诉 MAC 与 PHY 之间的数据接口工作在哪种模式。按接口类型分：

- **`"rmii"`**：百兆 RMII 接口，引脚更少、时钟更高（50 MHz）。
- **`"rgmii"` / `"rgmii-id"` / `"rgmii-txid"` / `"rgmii-rxid"`**：千兆 RGMII 接口及其延迟变体，本文档涉及的 TJA1103（100BASE-T1）/ TJA1120（1000BASE-T1）通常采用 RGMII 系列。
- 其它：`sgmii` 等，按实际芯片支持选择。

### 3.2.1 RGMII 及其延迟变体的区别

RGMII 是 2.1 节介绍的双沿采样接口。**`rgmii` 与 `rgmii-id` / `rgmii-txid` / `rgmii-rxid` 的区别，本质是"时钟与数据之间约 2ns 的相位延迟由谁来加"**。

| phy-mode | PHY 侧（内部）延时 | MAC 侧延时（tx_delay/rx_delay） | 适用场景 |
|----------|---------------------|-------------------------------|----------|
| `"rgmii"` | 不加 | **必须由 MAC 侧配置**（`tx_delay` / `rx_delay` 属性，或外部延迟电路） | 由 MAC 寄存器控制延迟 |
| `"rgmii-id"` | **TX + RX 都加** | 不加（应不配/置 0，避免叠加） | PHY 内部同时补偿收发两侧 |
| `"rgmii-txid"` | **仅 TX 加** | RX 侧仍需 MAC 配置（`rx_delay`）或外部补偿 | 只由 PHY 补偿发送方向 |
| `"rgmii-rxid"` | **仅 RX 加** | TX 侧仍需 MAC 配置（`tx_delay`）或外部补偿 | 只由 PHY 补偿接收方向 |

要点：

- **`-id` / `-txid` / `-rxid` 后缀的含义是"该方向的延迟由 PHY 内部寄存器补"**。此时 MAC 对应方向**不应再加延迟**，否则两侧各加一遍，总延迟超限，导致高速收发错位、大量 CRC 错误。
- **`"rgmii"`（不带后缀）表示两侧都不默认加延迟**，必须显式配置其中一侧：常见做法是在 MAC 设备树节点里配 `tx_delay` / `rx_delay`（Rockchip MAC 寄存器实现），或在 PHY 寄存器里开内部延迟。
- 配置**必须与实际硬件/芯片能力匹配**：
  - 若 PHY 不支持内部延迟而配了 `rgmii-id`，驱动虽然会给 PHY 写延迟寄存器，但 PHY 不生效，需改由 MAC 侧配 `tx_delay`/`rx_delay`；
  - 反之若 PHY 已开内部延迟，MAC 侧又配了 `tx_delay`/`rx_delay`，则延迟叠加出错。
- **判断与排查**：link 能 up 但 ping 不通、报文 CRC 错误率高、速率上不去，通常就是延迟两侧配置矛盾或取值不当，是 RGMII 最典型的坑之一。

### 3.2.2 结合本平台示例

3.1 节的示例节点采用 **`"rgmii-id"`** 模式，由 PHY 内部完成收发两侧延迟补偿，因此 `tx_delay` / `rx_delay` 被注释掉：

```dts
&gmac0 {
	phy-mode = "rgmii-id";   /* PHY 内部补偿 TX+RX 延迟 */
	clock_in_out = "output";

	//tx_delay = <0x20>;     /* PHY 已加延迟，MAC 侧不再配置 */
	//rx_delay = <0x3f>;
	...
};
```

如果换成 **`"rgmii"`** 模式，则必须取消注释并调好 `tx_delay` / `rx_delay` 的取值。

**`phy-mode` 必须与硬件实际接法一致**，且与 pinctrl、时钟配置（3.5/3.6 节）配套，否则 MAC 与 PHY 间数据无法正常收发。

## 3.3 PHY 复位配置（snps,reset-\*）

设备树通过以下属性控制 PHY 的硬件复位：

| 属性 | 含义 |
|------|------|
| `snps,reset-gpio` | PHY 的硬件复位脚（GPIO），如 `<&gpio2 RK_PC6 GPIO_ACTIVE_LOW>` |
| `snps,reset-delays-us` | PHY 的复位时序，三个时间值分别表示 PHY 不同阶段的复位时序 |
| `snps,reset-active-low` | 低电平复位（Reset 脚低有效）。常见 PHY 复位脚多为低有效，此时**默认不写该属性反而需要谨慎**，实际以 GPIO flag 与芯片要求为准 |
| `snps,reset-active-high` | 高电平复位 |

**`snps,reset-delays-us` 的三个时间值含义与复位极性相关：**

- 配置 `snps,reset-active-low`（低有效复位）时，三个时间分别表示 Reset pin **拉高 → 拉低 → 再拉高**的时间：
  1. 第一个值：复位前 pin 保持高电平（非复位）的时间；
  2. 第二个值：pin 拉低（进入复位）持续的时间；
  3. 第三个值：pin 释放拉高（复位结束）后，等待 PHY 完成上电初始化稳定下来的时间。
- 配置 `snps,reset-active-high`（高有效复位）时，则相反：表示 **拉低 → 拉高 → 再拉低** 三个阶段的时间。

**不同 PHY 的复位时序要求不同**，必须参照对应 PHY 数据手册中的"复位时间/上电时序"章节填写，例如 TJA1103 / TJA1120 的上电复位要求（最小复位保持时间、复位释放后到可以访问 MDIO 的等待时间），通常该第三个值需要给得足够大（如 100 ms 量级）。

## 3.4 phy-supply

`phy-supply` 用于指定 PHY 的电源（regulator）：

- 如果 PHY 的电源是**常供方式**（一直上电），**可以不用配置**该属性；
- 否则，需要配置对应的 regulator（设备树中对应一个电源节点），驱动会在 PHY 初始化前/后按 regulator 的使能顺序完成上电/下电控制，保证 PHY 在上电稳定后再进行复位和初始化。

## 3.5 时钟配置

GMAC 的时钟配置决定两件事：**MAC 内核时钟是否就绪**，以及 **RGMII 参考时钟（GTX_CLK / REF_CLK）由谁产生、频率多少**。配置错乱的表现通常为：PHY 枚举不到、link 起不来、速率只能跑 100M 或 10M。

### 3.5.1 GMAC 节点涉及的时钟（clocks / clock-names）

以 RK3576 为例，GMAC 节点通常声明以下几类时钟（具体名称/个数随 SoC 与内核版本略有差异，以对应 dtsi 为准）：

| clock-name | 作用 |
|------------|------|
| `stmmaceth`（主时钟） | MAC 内核工作时钟，必须使能，否则 MAC 无法工作 |
| `pclk` | APB 从接口时钟，供 MAC 寄存器/MDIO 管理接口使用 |
| `ptp_ref` | 1588 PTP 时间戳参考时钟（可选，未用 PTP 可不关心） |
| `mac_ref` | RGMII/RMII 参考时钟：决定 `GTX_CLK` / `REF_CLK` 的来源与频率 |

### 3.5.2 clock_in_out：时钟方向（Rockchip 特有属性）

Rockchip 内核用 **`clock_in_out`** 指定参考时钟的方向，必须在设备树中显式配置：

- **`clock_in_out = "output"`**：参考时钟由 **MAC 产生并输出**给 PHY。RGMII 模式下即 MAC 输出 `GTX_CLK`（千兆 125 MHz，百兆 25 MHz，十兆 2.5 MHz）。
- **`clock_in_out = "input"`**：参考时钟由**外部（PHY 或独立晶振/时钟源）提供**，MAC 侧作为输入使用。

### 3.5.3 参考时钟频率与 assigned-clocks

- RGMII：千兆 = 125 MHz，百兆 = 25 MHz，十兆 = 2.5 MHz（由 MAC 按协商速率自动切换）；
- RMII：固定 50 MHz（`REF_CLK`）；
- 可用 `assigned-clocks` / `assigned-clock-rates` 强制指定时钟源与初始频率，例如给 `mac_ref` 分配 125 MHz，确保初始化时时钟正确。

### 3.5.4 结合 TJA1103 / TJA1120 的时钟要点

- TJA1103（100BASE-T1）/ TJA1120（1000BASE-T1）需要一个参考时钟。常见两种来源：
  1. **由 MAC 输出**（`clock_in_out = "output"`）：借用 RGMII 的 `GTX_CLK` 作为 PHY 时钟；
  2. **由 CRU 专用时钟提供**：如设备树中注释示例 `clocks = <&cru REFCLKO25M_GMAC0_OUT>`——由 SoC 的 CRU 直接给 PHY 输出一路 25 MHz 参考时钟（100BASE-T1 常用 25 MHz）。
- **PHY 的时钟主/从（Master/Slave）模式必须与上述来源匹配**：PHY 若配置为时钟主，则其自身提供时钟，MAC 应设 `clock_in_out = "input"`；若 PHY 配置为时钟从，则由 MAC/CRU 提供时钟，MAC 设 `clock_in_out = "output"`。主从不匹配同样会导致 link 不上或时钟冲突。
- 时钟、`phy-mode`、`pinctrl` 三者必须成套配置，不能只改其中一项。

## 3.6 pinctrl

`pinctrl-0` 配置 GMAC 各信号对应的 SoC 引脚复用。要点如下：

- **RGMII 与 RMII 模式下配置不一样**：两者的数据线、时钟、控制信号的引脚组不同（例如 RGMII 使用 `gmac1_rgmii_clk` / `gmac1_rgmii_bus`，RMII 则使用 RMII 对应的时钟与总线组）；
- **时钟引脚驱动强度**：对于**输出时钟**的引脚（例如 RMII 模式下 `ref_clk` pin 脚作为时钟输出时），为满足时序/信号质量，该引脚的**驱动强度一般需要配置得更大**（如 `drive-strength` 增大），具体数值参考硬件走线与信号完整性评估。

## 3.7 tx_delay / rx_delay

- **仅当使用 `"rgmii"` 模式（无 `-id` 后缀）时需要配置**，用于补偿时钟与数据之间的相位延迟（对应 2.1.3 节的 2ns 对齐问题）；若使用 `"rgmii-id"` / `"rgmii-txid"` / `"rgmii-rxid"`，延迟由 PHY 内部补偿，MAC 侧**不应再配**（详见 3.2 节）；
- 取值以 PHY 内部 delay 是否启用为依据：若 PHY 侧已启用 delay，则 MAC 侧相应方向需减小或置为 0，**两侧只能启用其中一边**；
- 具体取值需要结合 PCB 走线长度、PHY 寄存器设置综合调整，调整方法可参考 3.8 节的 Rockchip GMAC 配置指南，本文后续如有 "RGMII Delayline" 专题也会展开。

## 3.8 参考文档

由于不同芯片下的不同模式配置比较多，请参考另外一份文档：

**《Rockchip_Developer_Guide_Linux_GMAC_Mode_Configuration_CN.pdf》**

---

（本章完）

---

# 4. TJA1120 pin strap 配置

本章依据 NXP 应用笔记 **AN13663（TJA1120 Application Note）§3.10 Pin strapping** 编写，介绍 TJA1120 的硬件 strap 配置。TJA1120 在上电时通过 pin-strapping 完成基本硬件配置，strap 结果锁存在 SMI 可访问的寄存器中，**除 PHY 地址外均可通过 MDIO 读取并修改**。

## 4.1 strap 引脚与功能映射（Table 14）

TJA1120 使用 **CONFIG0 ~ CONFIG6** 共 7 个引脚，在复位上电时被采样，各引脚与功能对应关系如下：

| 引脚 | 封装 Pin | 功能 |
|------|---------|------|
| CONFIG0 | 17 | PHY 地址（最高位） |
| CONFIG1 | 19 | PHY 地址 |
| CONFIG2 | 20 | PHY 地址（最低位） |
| CONFIG3 | 21 | xMII 模式（RGMII / SGMII） |
| CONFIG4 | 23 | xMII 模式 |
| CONFIG5 | 24 | 时钟模式 |
| CONFIG6 | 25 | 1000BASE-T1 Leader/Follower、Autonomous 模式 |

> 注意：strap 采样完成后，CONFIG2 ~ CONFIG6 会复用为 RGMII 接口引脚（RXD3/RXD2/RXD1/RXD0/RX_CTL），见 4.8 节。

## 4.2 三态 strap 与外部电阻（Table 15）

每个 CONFIG 引脚有三种状态，通过**外部 10 kΩ 上下拉电阻**设定：

| Setting | 名称 | 上拉电阻（接 VDDIO） | 下拉电阻（接 GND） |
|---------|------|----------------------|--------------------|
| O | Open（悬空） | Open（不接） | Open（不接） |
| H | High | 10 kΩ | Open |
| L | Low | Open | 10 kΩ |

- 外部上/下拉电阻标准值为 **10 kΩ**；
- strap 采样期间，其它连接到该引脚的外围电路不得干扰，应避免与 strap 状态冲突的寄生阻抗。

## 4.3 采样检测逻辑（3.10.1）

strap 检测分两阶段进行：

1. 使能**内部下拉**（禁用上拉），采样引脚电平；
2. 使能**内部上拉**（禁用下拉），采样引脚电平。

两次采样结果解码如下（Table 16）：

| 下拉阶段采样 | 上拉阶段采样 | 解码结果 |
|:---:|:---:|:---:|
| 0 | 0 | 外部下拉 → **L** |
| 1 | 1 | 外部上拉 → **H** |
| 0 | 1 | 悬空 → **O** |
| 1 | 0 | 无效（Invalid） |

TJA1120 内部上/下拉电阻约 **40 kΩ ~ 62 kΩ**，因此对外部电阻的约束（Table 17）：

| 目标值 | 约束 |
|--------|------|
| H | 上拉电阻 < 17 kΩ |
| L | 下拉电阻 < 17 kΩ |
| O | 上/下拉电阻 > 145 kΩ |

> 同板使用多颗 TJA11xx PHY 时，**CONFIG 引脚不能互相连接**，否则各自的 strap 检测逻辑会互相干扰，导致检测错误。

## 4.4 PHY 地址配置（3.10.3，CONFIG0/1/2）

- TJA1120 的 PHY 地址由 **CONFIG0、CONFIG1、CONFIG2** 三根 strap 引脚组合决定，可选 **地址 1 ~ 27**；
- 完整编码见数据手册 **Table 19**（PHY address pin-strapping），常见示例：
  - **PHYADDR = 7**：CONFIG0 = H、CONFIG1 = O、CONFIG2 = H（对应 AN §3.10.2 示例）；
  - **PHYADDR = 2**：见 AN Figure 20 的接法示例；
- **PHY 地址用于 SMI 通信本身，无法通过 SMI/MDIO 修改**，只能改硬件 strap；
- 设备树 PHY 节点 `reg` 必须与 strap 得到的地址一致（如 `reg = <0x7>`）。

> 说明：PHY 地址的完整编码表因 PDF 文字提取存在乱码，此处未逐条展开，**务必以 NXP 数据手册 Table 19 为准**，并对照你们板子的实际 strap 组合确认。

## 4.5 xMII 模式配置（3.10.4，CONFIG3/4）

**CONFIG3、CONFIG4** 两脚决定 RGMII / SGMII 默认模式（Table 20）。TJA1120A 支持 CMOS 型 RGMII，TJA1120B 支持 SGMII：

| CONFIG3 | CONFIG4 | TJA1120A（RGMII 变体） | TJA1120B（SGMII 变体） |
|:---:|:---:|--------------------------|--------------------------|
| O | O | RGMII-ID（RXC） | SGMII-PHY |
| O | H | - | SGMII-MAC |
| O | L | 非法（禁止） | 非法（禁止） |
| H | O | 非法（禁止） | 非法（禁止） |
| H | H | RGMII | - |
| H | L | 非法（禁止） | 非法（禁止） |
| L | O | 非法（禁止） | 非法（禁止） |
| L | H | RGMII-ID（TXC/RXC） | - |
| L | L | JTAG | JTAG |

- **非法组合必须避免**；
- 与设备树 `phy-mode` 的对应关系：`RGMII` → `"rgmii"`，`RGMII-ID` → `"rgmii-id"` 等（见 3.2 节）；
- 本项目 TJA1120 使用 RGMII 时，需确认 strap 组合对应 `RGMII` 还是 `RGMII-ID`，再决定设备树 delay 配置（3.2、3.7 节）。

## 4.6 时钟模式配置（3.10.5，CONFIG5）

**CONFIG5** 控制时钟输入方式（Table 21）：

| CONFIG5 | CLK select |
|:---:|------------|
| O | differential（差分时钟） |
| H | XO mode（晶体振荡器） |
| L | single ended（单端时钟） |

注意：

- 时钟模式影响 **XI/XO 引脚允许的最大电压**：只有 single ended 模式允许信号电平达到 VDDIO，其它模式限制为 1.1 V；
- 还有一种允许 1.1 V 摆幅的单端模式，可通过寄存器选择（见 AN §3.6）；
- 与 3.5 节 `clock_in_out`、PHY 时钟主/从配置配套。

## 4.7 Leader/Follower 与 Autonomous 模式（3.10.6，CONFIG6）

**CONFIG6** 同时决定两件事：1000BASE-T1 链路的 **Leader/Follower 角色**，以及 PHY 启动时的 **Autonomous（自主）/ Managed（受控）模式**。两者都是车载以太网特有的概念，下面分别说明。

### 4.7.1 Leader/Follower 是什么

- 1000BASE-T1 是**点对点单对以太网**，一条链路只有**两端**（两个 PHY）；
- 建链时两端必须分出角色：一端为 **Leader**，另一端为 **Follower**。Leader 负责产生/控制链路主时钟并主导训练，Follower 同步到 Leader；
- **一条链路只能有一个 Leader**：
  - 两端同为 Leader → 时钟冲突，无法建链；
  - 两端同为 Follower → 无人主导，无法建链；
- 因此**部署时必须保证链路两端恰好一 Leader 一 Follower**。常见做法：上游（如交换机/中央网关侧）设为 Leader，下游 ECU 设为 Follower；也可按需互换；
- 角色可由 **CONFIG6 strap** 或 **寄存器（MMD1 中 BASE_T1_PMA_* 相关位）** 配置，strap 是上电默认值。

### 4.7.2 Autonomous（自主）与 Managed（受控）模式

区分的是** PHY 上电后要不要等主机控制**再启动：

- **Autonomous（自主）**：PHY 上电后**自己完成配置并自动建链**，无需主机（CPU）通过 MDIO 干预。适合主机尚未就绪、但链路需要先拉起来的场景；
- **Managed（受控）**：PHY 上电后停在 **Standby**，**暂停等待主机指令**，直到主机通过 MDIO 置位 **MMD30.DEVICE_CONTROL.START_OPERATION（30.0040h bit0）** 才继续进入工作。主机可以在此之前完成 Leader/Follower、节能等参数配置，再统一启动，便于完全掌控启动时序。

> 结合本驱动：`config_init` 末尾会调用 `nxp_c45_start_op()`（写 `VEND1_PHY_CONTROL` 的 `PHY_START_OP` 位）下发"开始运行"，并在初始化中置 `PHY_CONFIG_AUTO`。即驱动接管后主动把 PHY 启动起来，即使 strap 配成 Managed 模式，也会由驱动在配置完成后放开。

### 4.7.3 CONFIG6 strap 组合（Table 22）

| CONFIG6 | Leader/Follower | Autonomous 模式 | 含义 |
|:---:|-----------------|-----------------|------|
| O | 寄存器控制（默认 Follower） | 使能 | 自主启动；角色由软件通过寄存器设置 |
| H | Leader | 使能 | 自主启动，本端为 Leader |
| L | Follower | 关闭（Managed） | 受控启动，本端为 Follower；PHY 暂停直至 START_OPERATION |

### 4.7.4 本项目注意点

- **确认对端角色**：与对端 ECU/工装联调前，先确认链路两端恰好一个 Leader、一个 Follower，否则表现为"link 起不来"（见 7.3 节），容易误判为硬件/线缆问题；
- **Managed 模式要放行**：若 strap 为 CONFIG6 = L，确认驱动会下发 START_OPERATION（本项目由 `nxp_c45_start_op` 完成），否则 PHY 会一直停在 Standby；
- **极性校正差异**：与 TJA1103/4 相比，TJA1120 的极性校正（polarity correction）**不能通过 pin-strapping 设置**，默认使能，可通过寄存器修改。

## 4.8 strap 采样时机与替代功能（3.10.7）

- strap 状态在**上电或复位**时采样并锁存；
- 采样完成后引脚复用：
  - **CONFIG0/GPIO0**：默认作为 LED 输出。外接 LED 时，**LED 极性必须与 strap 电阻匹配**（LED 相当于一个电阻，会覆盖 strap 配置），或使用 FET 在 strap 阶段隔离负载；
  - **CONFIG1/GPIO2**：默认输入，strap 结束后呈高阻态；
  - **CONFIG2 ~ CONFIG6**：属于 RGMII 接口，strap 完成后配置为输出（RXD3/RXD2/RXD1/RXD0/RX_CTL），不会出现未定义的中间态；
- PCB 上 strap 引脚到电阻的**走线 stub 尽量短**，避免寄生阻抗影响检测。

## 4.9 strap 配置检查清单

- [ ] CONFIG0/1/2 的 H/L/O 组合与数据手册 Table 19 核对，得到 PHY 地址，且与设备树 `reg` 一致
- [ ] CONFIG3/4 的 xMII 模式与设备树 `phy-mode` 一致（RGMII / RGMII-ID / SGMII）
- [ ] CONFIG5 时钟模式（differential / XO / single ended）与 3.5 节 `clock_in_out` 匹配
- [ ] CONFIG6 的 Leader/Follower 与 Autonomous/Managed 模式符合应用需求；若为 Managed 模式，驱动需设置 MMD30 的 START_OPERATION
- [ ] 外部 10 kΩ strap 电阻、短 stub、LED 极性等符合 4.2/4.8 要求

---

（本章完）

---

# 5. NXP C45 PHY 驱动结构（struct phy_driver nxp_c45_driver[]）

本章讲解 RK3576 平台 TJA1103 / TJA1120 的 PHY 驱动核心数据结构 **`struct phy_driver nxp_c45_driver[]`**。这是驱动中"注册哪些 PHY 型号、每个型号挂哪些回调"的声明，是理解整个驱动的入口。

## 5.1 文件与驱动框架定位

- 文件：`kernel-6.1/drivers/net/phy/nxp-c45-tja11xx.c`（NXP 官方驱动 + RK 本地改动）
- 文件末尾用 `module_phy_driver(nxp_c45_driver)` 把该数组注册进内核 **phylib**（PHY 驱动框架）；
- 内核 MDIO 总线枚举到某颗 PHY 后，用 **PHY ID** 匹配本驱动，匹配成功即绑定并调用对应回调。

> 说明：本文件实际覆盖的是 **TJA1103 / TJA1120**（即"TJA11xx"），不是 TJA1102。驱动的核心就是下面的数组。

## 5.2 struct phy_driver 数组结构

驱动用**数组**形式同时注册多个 PHY 型号，一个元素对应一颗芯片：

```c
static struct phy_driver nxp_c45_driver[] = {
    {                       /* ── 第 1 个元素：TJA1103 ── */
        PHY_ID_MATCH_MODEL(PHY_ID_TJA_1103),   /* 0x001BB010 */
        .name           = "NXP C45 TJA1103",
        .get_features   = nxp_c45_get_features,
        .driver_data    = &tja1103_phy_data,
        .probe          = nxp_c45_probe,
        .soft_reset     = nxp_c45_soft_reset,
        .config_aneg    = genphy_c45_config_aneg,
        .config_init    = nxp_c45_config_init,
        .config_intr    = tja1103_config_intr,
        .handle_interrupt = nxp_c45_handle_interrupt,
        .read_status    = genphy_c45_read_status,
        .suspend        = genphy_c45_pma_suspend,
        .resume         = genphy_c45_pma_resume,
        .get_sset_count = nxp_c45_get_sset_count,
        .get_strings    = nxp_c45_get_strings,
        .get_stats      = nxp_c45_get_stats,
        .cable_test_start       = nxp_c45_cable_test_start,
        .cable_test_get_status  = nxp_c45_cable_test_get_status,
        .set_loopback   = genphy_c45_loopback,
        .get_sqi        = nxp_c45_get_sqi,
        .get_sqi_max    = nxp_c45_get_sqi_max,
        .remove         = nxp_c45_remove,
    },
    {                       /* ── 第 2 个元素：TJA1120 ── */
        PHY_ID_MATCH_MODEL(PHY_ID_TJA_1120),   /* 0x001BB031 */
        .name           = "NXP C45 TJA1120",
        .get_features   = nxp_tja1120_get_features,
        .driver_data    = &tja1120_phy_data,
        .probe          = nxp_c45_probe,
        .soft_reset     = nxp_c45_soft_reset,
        .config_aneg    = tja1120a_config_aneg,   /* RK 改动 */
        .config_init    = nxp_c45_config_init,
        .config_intr    = tja1120_config_intr,
        .handle_interrupt = nxp_c45_handle_interrupt,
        .read_status    = genphy_c45_read_status,
        .link_change_notify = tja1120_link_change_notify,  /* 仅 TJA1120 */
        .suspend        = genphy_c45_pma_suspend,
        .resume         = genphy_c45_pma_resume,
        .get_sset_count = nxp_c45_get_sset_count,
        .get_strings    = nxp_c45_get_strings,
        .get_stats      = nxp_c45_get_stats,
        .cable_test_start       = nxp_c45_cable_test_start,
        .cable_test_get_status  = nxp_c45_cable_test_get_status,
        .set_loopback   = genphy_c45_loopback,
        .get_sqi        = nxp_c45_get_sqi,
        .get_sqi_max    = nxp_c45_get_sqi_max,
        .remove         = nxp_c45_remove,
    },
};
```

## 5.3 字段分组详解

`struct phy_driver` 是 phylib 定义的回调集合，各字段按用途分组：

**① 匹配与标识**
| 字段 | 说明 |
|------|------|
| `PHY_ID_MATCH_MODEL(PHY_ID_TJA_1103)` | 宏，展开为 `.phy_id = 0x001BB010, .phy_id_mask = 0xFFFFFFF0`，用于按 PHY ID 精确匹配 |
| `PHY_ID_MATCH_MODEL(PHY_ID_TJA_1120)` | 同上，`0x001BB031` |
| `.name` | 驱动名，`dmesg`/`sysfs` 中可见 |

**② 探测与生命周期**
| 字段 | 说明 |
|------|------|
| `.probe` | 绑定成功后调用：分配 `struct nxp_c45_phy priv`，初始化 PTP 时钟、MACsec 等（`nxp_c45_probe`） |
| `.soft_reset` | 软复位 PHY：写 MMD DEVICE_CONTROL 复位位并轮询复位完成（`nxp_c45_soft_reset`） |
| `.suspend` / `.resume` | 电源管理挂起/恢复（C45 通用实现） |
| `.remove` | 卸载清理：注销 PTP 时钟、清空 skb 队列（`nxp_c45_remove`） |

**③ 能力与配置**
| 字段 | 说明 |
|------|------|
| `.get_features` | 上报 PHY 支持的速率能力，写入 `supported`（TJA1103 加 100baseT1_Full，TJA1120 加 1000baseT1_Full） |
| `.driver_data` | **指向芯片私有数据**，见 5.5 节 |
| `.config_init` | 初始化配置（`nxp_c45_config_init`）：使能配置、应用 errata、设 xMII 模式与 RGMII 延时、使能计数器、初始化 PTP |
| `.config_aneg` | 自动协商配置。TJA1103 用 C45 通用实现；**TJA1120 用 RK 改写的 `tja1120a_config_aneg`（直接返回 0，配合 `config_init` 中 `autoneg = AUTONEG_DISABLE` 关闭自协商）** |
| `.read_status` | 读取链路/速率/双工状态（C45 通用 `genphy_c45_read_status`） |
| `.link_change_notify` | 链路状态变化通知，仅 TJA1120 注册，用于 PCS 复位等 errata 处理 |

**④ 中断**
| 字段 | 说明 |
|------|------|
| `.config_intr` | 使能/禁止 PHY 中断。TJA1103 用 `tja1103_config_intr`（清理 FUSA 中断），TJA1120 用 `tja1120_config_intr`（多一个 BOOT_DONE 中断） |
| `.handle_interrupt` | 中断处理（`nxp_c45_handle_interrupt`）：读中断状态、上报链路事件、处理 PTP egress 时间戳 |

**⑤ ethtool 与诊断**
| 字段 | 说明 |
|------|------|
| `.get_sset_count` / `.get_strings` / `.get_stats` | 硬件统计的条数/名称/数值（如链路丢包、符号错误等，见 `*_hw_stats`） |
| `.cable_test_start` / `.cable_test_get_status` | 电缆（链路）测试 |
| `.set_loopback` | 回环测试（C45 通用） |
| `.get_sqi` / `.get_sqi_max` | 信号质量指数 SQI 及其最大值（1000BASE-T1 链路质量评估） |

## 5.4 两个表项（TJA1103 / TJA1120）对比

两颗芯片**绝大部分回调复用同一份函数**（probe、soft_reset、config_init、handle_interrupt、read_status、stats、cable_test 等），差异只在芯片相关的几处：

| 字段 | TJA1103 | TJA1120 |
|------|---------|---------|
| PHY ID | `0x001BB010` | `0x001BB031` |
| `.name` | NXP C45 TJA1103 | NXP C45 TJA1120 |
| `.get_features` | `nxp_c45_get_features`（100baseT1_Full） | `nxp_tja1120_get_features`（1000baseT1_Full） |
| `.driver_data` | `&tja1103_phy_data` | `&tja1120_phy_data` |
| `.config_aneg` | `genphy_c45_config_aneg`（通用） | `tja1120a_config_aneg`（RK 改动，关闭自协商） |
| `.config_intr` | `tja1103_config_intr` | `tja1120_config_intr` |
| `.link_change_notify` | 无 | `tja1120_link_change_notify` |

> RK 本地改动提示：本仓库这份代码里带调试打印（`pr_err("hj ...")`）、`tja1120a_config_aneg` 直接返回 0、`config_init` 中强制 `autoneg = AUTONEG_DISABLE`、以及 probe 里创建读取寄存器的调试线程等，属于调试期痕迹，非 NXP 原版。

## 5.5 driver_data 芯片私有数据

`.driver_data` 指向 `struct nxp_c45_phy_data`，把**两颗芯片的差异（寄存器映射、统计项、PTP 相关回调等）封装成数据**，使共用代码能按芯片区分行为：

```c
static const struct nxp_c45_phy_data tja1103_phy_data = { ... };  /* regmap=&tja1103_regmap, PTP_CLK_PERIOD_100BT1 等 */
static const struct nxp_c45_phy_data tja1120_phy_data = { ... };  /* regmap=&tja1120_regmap, PTP_CLK_PERIOD_1000BT1 等 */
```

- `.regmap`：指向 `struct nxp_c45_regmap`，保存 PTP、LTC、外部触发等寄存器的 **MMD 地址映射**（两芯片不同，如 PTP 时钟周期寄存器 TJA1103 为 0x1104，TJA1120 为 0x1020）；
- `.stats` / `.n_stats`：芯片各自的硬件统计表；
- `.ptp_clk_period`：PTP 时钟周期（100BASE-T1 为 15 ns，1000BASE-T1 为 8 ns）；
- 其余为 PTP/中断相关的函数指针（get_egressts、get_extts、ptp_init、ptp_enable、nmi_handler 等）。

驱动通过 `nxp_c45_get_data(phydev)` 取得 `phydev->drv->driver_data` 后，即可用一套代码访问不同芯片的寄存器。

## 5.6 注册与匹配（module_phy_driver）

```c
module_phy_driver(nxp_c45_driver);                       /* 注册整个驱动数组 */

static struct mdio_device_id nxp_c45_tbl[] = {           /* MODULE 设备表，用于热插拔/自动加载 */
    { PHY_ID_MATCH_MODEL(PHY_ID_TJA_1103) },
    { PHY_ID_MATCH_MODEL(PHY_ID_TJA_1120) },
    { /* sentinel */ },
};
MODULE_DEVICE_TABLE(mdio, nxp_c45_tbl);
```

工作流程：

1. 启动时 `module_phy_driver()` 把 `nxp_c45_driver` 注册进 phylib；
2. 内核 MDIO 总线（mii_bus）扫描各 PHY 地址，读到 PHY 的 ID（0x001BB010 / 0x001BB031）并据此创建 `phy_device`；
3. phylib 用 `phy_id` + `phy_id_mask` 与各驱动匹配，命中本驱动后绑定，依次调用 `probe → soft_reset → config_init → config_aneg → read_status ...`，链路建立后正常运行；
4. 若编译为模块，`nxp_c45_tbl` 提供 MODULE 设备表供自动加载。

---

（本章完）

---

# 6. PHY 驱动架构与网卡 Link Up 流程

本章从整体角度讲解：Linux 下 PHY 驱动在整条网络数据通路中的位置、phylib 如何驱动 PHY 工作，以及网卡从 `ifconfig eth0 up` 到链路（link）建立的全过程。结合本项目（RK3576 + TJA1120）落地讲解。

## 6.1 分层架构总览

Linux 以太网数据通路分层如下，PHY 驱动处于最底层：

```
┌───────────────────────────────────────┐
│ 网络协议栈（TCP/IP、socket）            │
├───────────────────────────────────────┤
│ net_device（eth0）                    │
│   ndo_start_xmit / ndo_set_rx_mode 等 │
├───────────────────────────────────────┤
│ MAC 驱动（stmmac/dwmac，RK3576 GMAC）  │
│   .adjust_link() 回调                 │
├───────────────────────────────────────┤
│ phylib（PHY 驱动框架）                 │
│   phy_device / phy_driver / 状态机     │
├───────────────────────────────────────┤
│ MDIO 总线（mii_bus）                  │
│   mdiobus_read / mdiobus_write        │
├───────────────────────────────────────┤
│ PHY 芯片（TJA1103 / TJA1120）         │
└───────────────────────────────────────┘
```

- **MAC 驱动（dwmac/stmmac）**：管理 GMAC 内核、RGMII 引脚、MDIO 控制器；把"链路状态/速率/双工"通过 `.adjust_link()` 配置进 GMAC。
- **phylib**：通用 PHY 框架，负责 PHY 的枚举、匹配、状态机调度，屏蔽了各 PHY 的差异。
- **PHY 驱动（第 5 章的 `nxp_c45_driver`）**：具体芯片（TJA1103/1120）的回调实现，最终通过 MDIO 读写 PHY 寄存器。

## 6.2 核心对象与数据流

| 对象 | 作用 |
|------|------|
| `mii_bus`（mdio bus） | 一条 MDIO 总线；MAC 驱动注册后扫描 PHY 地址（0~31）枚举 PHY |
| `phy_device` | 代表一颗具体 PHY，保存地址、ID、接口模式、link/speed/duplex、中断等状态 |
| `phy_driver` | PHY 驱动回调集合（第 5 章），挂到 `phy_device->drv` |
| `phy_device->driver_data` | 第 5.5 节的芯片私有数据 |
| `phy_state_machine` | 内核工作队列中的状态机，周期性驱动 PHY 状态迁移 |

数据流：MAC 驱动注册 `mii_bus` → 总线扫描到 PHY → 创建 `phy_device` → 按 PHY ID 匹配 `phy_driver` → 绑定并执行初始化回调 → 状态机持续运行维护链路状态 → 通过 `adjust_link`/`netif_carrier_*` 通知上层。

## 6.3 phylib 状态机

phylib 有一个内核线程（工作队列）`phy_state_machine()`，周期（默认约 1 s，有中断则被立即唤醒）执行一次，是 **link 状态维护的核心**。

主要状态及迁移（关键部分）：

| 状态 | 含义 | 主要动作 |
|------|------|----------|
| `PHY_READY` | 已初始化，等待启动 | `phy_start()` 后进入 `PHY_UP` 或 `PHY_AN` |
| `PHY_AN` | 自动协商进行中 | 调用 `.config_aneg` 配置协商；`.read_status` 查是否完成 |
| `PHY_UP` | 无需自协商，等待 link | `.read_status` 读 link |
| `PHY_RUNNING` | **link 已建立**，正常工作 | `.read_status` 监视；link 掉 → `PHY_NOLINK` |
| `PHY_NOLINK` | 无链路 | `.read_status` 监视；link 起 → `PHY_RUNNING` |
| `PHY_CABLETEST` | 电缆测试中 | 调用 `.cable_test_*` 回调 |

状态机每次跑一圈都会调用 `.read_status`，`genphy_c45_read_status` 会去读 PHY 的 C45 状态寄存器，得到 link / speed / duplex 并写入 `phydev`。链路从 0→1 时调用 `phy_link_change()`，进而触发 MAC 的 `adjust_link()` 并 `netif_carrier_on()`。

**两种链路监测方式**：

- **中断驱动**：PHY 有中断脚接到 SoC 时（本项目可配 GPIO INT），`config_intr` 使能 PHY 中断，link 事件触发 `.handle_interrupt` → `phy_trigger_machine()` 立即唤醒状态机，响应快；
- **轮询**：无中断时，状态机按周期轮询 `read_status`，实现简单但有延迟。

## 6.4 Link Up 完整流程

以本项目（RK3576 + TJA1120，RGMII）为例，从开机到链路建立：

```
上电
 └─ 1. MAC 驱动 probe
 │    注册 mii_bus，扫描 PHY 地址
 │    读到 PHY ID 0x001BB031 → 匹配 nxp_c45_driver
 │    → .probe (nxp_c45_probe)：分配 priv、初始化 PTP/MACsec
 └─ 2. .soft_reset：写 MMD DEVICE_CONTROL 复位 PHY，等待复位完成
 └─ 3. .config_init (nxp_c45_config_init)
 │    使能配置；应用 TJA1120 errata；
 │    设置 xMII 模式=RGMII 及内部 delay（rxd/txd）；
 │    使能计数器、初始化 PTP；autoneg = DISABLE（RK 改动）
 └─ 4. ifconfig eth0 up → MAC 驱动调用 phy_start()
 │    状态机进入 PHY_UP（自协商已关闭）
 └─ 5. PHY 硬件自动完成 1000BASE-T1 link training（Master/Slave 建立）
 └─ 6. 状态机 .read_status → 读到 link=1, speed=1000, duplex=full
 └─ 7. phy_link_change() → MAC 的 adjust_link() 回调
 │    按 speed/duplex 配置 GMAC 时钟与工作模式
 │    → netif_carrier_on()，eth0 网络可用
```

运行期：链路断开 → 状态机由 `PHY_RUNNING` → `PHY_NOLINK` → `netif_carrier_off()`；恢复后回到 `PHY_RUNNING`。若使用 PHY 中断，跳变瞬间即被感知，否则最多延迟一个轮询周期。

## 6.5 结合本项目（TJA1120 / RK 改动）

- **自协商被关闭**：本驱动 `config_init` 中强制 `autoneg = AUTONEG_DISABLE`，`config_aneg`（`tja1120a_config_aneg`）直接返回 0。1000BASE-T1 的 Master/Slave 建立由 **PHY 硬件自动完成**，驱动不参与配置，只需用 `read_status` 上报结果；
- **Link 依赖 PHY 硬件**：因为软件不发起协商，链路能否建立主要取决于对端设备与线缆/连接器质量；SQI（`get_sqi`）可用于评估链路质量；
- **链路变化通知**：TJA1120 注册了 `link_change_notify`（PCS 复位 errata），在链路断开时复位内部 PCS，保证恢复后时间戳功能正常；
- **建议核对**：若使用 PHY 中断加速 link 检测，需确认设备树 `phy-handle` 对应的 PHY 节点有 `interrupts` 配置、`config_intr` 使能、且 SoC GPIO 能收到中断；否则退化为轮询。

---

（本章完）

---

# 7. 调试流程

本章给出 RK3576 + TJA1103/1120 以太网从"硬件上电"到"网络可用"的**自底向上调试流程**。按顺序排查，能把问题快速定位到硬件、总线、驱动、还是链路/线缆层面。

```
7.1 硬件（晶振/电源/时钟）
  └─ 7.2 MDIO 探测到 PHY 且驱动匹配
       └─ 7.3 Link Up
            └─ 7.4 Loopback 测试（内部通路）
                 └─ 7.5 端对端连接测试
```

## 7.1 检查硬件配置：晶振、电源、时钟

PHY 无法工作，先确认最底层硬件是否就绪。

**检查项与工具：**

| 检查项 | 如何检查 | 合格标准 |
|--------|----------|----------|
| 电源 | 万用表量 PHY 电源引脚 | VDDA（3.3 V）、VDDIO（1.8/3.3 V）电压正确且稳定 | 检查INH脚是否拉高避免进入休眠模式 |
| 时钟 | 示波器量 XI/XO 或时钟输入脚 | 频率正确（25 MHz），幅度满足芯片要求 | 检查RXC输出时钟频率和幅值 (125M/25M/2.5M， 3v3/1v8);
| 时钟来源/模式 | 对照原理图 + 4.6 节 CONFIG5 strap | strap 时钟模式与时钟来源（晶振/CRU/MAC 输出）一致 |
| 复位时序 | 示波器量 RESET 脚 | 复位释放后满足上电时序（见 3.3 节 `snps,reset-delays-us`） |
| strap 配置 | 万用表量 CONFIG0~CONFIG6 | 上/下拉状态符合 4.2 节预期（H/L/O） |

**常见问题：**

- 晶振不起振 / 频率不对 → pin strap时钟模式配置不对/时钟源或负载电容问题；
- 电源纹波大或上电顺序不对 → PHY 初始化异常；
- strap 被 LED/其它电路干扰（见 4.8 节）→ strap 值不对，PHY 地址/模式错误。

> 7.1 全部通过后，才进入 7.2。

## 7.2 MDIO 是否探测到 PHY 以及驱动是否匹配

MDIO 能读到 PHY 寄存器，是驱动工作的前提。

**检查方法（目标机上执行）：**

```bash
# 1) 查看 PHY 枚举与驱动绑定日志
dmesg | grep -iE "phy|mdio|nxp"

# 2) 查看 mdio 总线上枚举出的设备
ls /sys/bus/mdio_bus/devices/

# 3) 查看 PHY ID（应读到 0x001BB031 / 0x001BB010）
cat /sys/bus/mdio_bus/devices/*/phy_id
```

**判断是否匹配：**

- 若能枚举出 PHY 设备且 PHY ID 正确（TJA1120 = `0x001BB031`，TJA1103 = `0x001BB010`），说明 MDIO 通路正常；
- 若驱动已绑定，`dmesg` 中会出现 "NXP C45 TJA1120" 以及 `probe`/`config_init` 等调试打印；结合第 5 章各回调的打印可确认执行到哪一步；
- 本驱动 `probe` 还会创建一个线程持续打印 SIGNAL_QUALITY / MSE / EPHY 等寄存器，可辅助判断 PHY 是否正常工作。

**常见问题：**

| 现象 | 可能原因 |
|------|----------|
| 探测到PHY，probe函数已经运行，但是最终匹配失败 | 驱动 get_features 未获取到对应带宽速率，部分内核的genphy_c45_pma_read_abilities接口可能读取不全，需要手动添加
| mdio_bus 下无设备 | PHY 地址（`reg`）与 strap 不一致（见 3.1、4.4）；MDC/MDIO 无时钟（见 3.5）；pinctrl 未配置 |
| 有设备但 PHY ID 全 0 / 全 F | PHY 未上电或未复位完成；MDIO 无上拉；地址扫错 |
| PHY ID 不对 | strap 地址配置错、器件版本不同 
| 驱动未匹配 | `phy_driver` 未注册（模块未加载）；PHY ID 与 `PHY_ID_MATCH_MODEL` 不符 |

> 7.2 通过后，说明"驱动 ↔ PHY"通路 OK，进入 7.3。

## 7.3 是否 Link Up

确认 PHY 已与对端建立物理链路。

**检查方法：**

```bash
ifconfig eth0                    # 看 eth0 是否 UP、是否有 IP
ethtool eth0                     # 看 "Link detected: yes/no"、speed、duplex
cat /sys/class/net/eth0/carrier  # 1 = link up，0 = link down
cat /sys/class/net/eth0/operstate
dmesg | grep -iE "link|carrier"
```

**按 6.4 流程定位：**

- 若 **never link up**：检查 7.1（时钟方向/strap）、对端是否在线、线缆/连接器是否接好、`phy-mode` 与 strap 的 xMII 模式是否一致（见 4.5）；
- 若 **link 有、但速率不对**：核对 `read_status` 读到的 speed，检查 RGMII 延时配置（3.2、3.7 节）；
- 本项目自协商关闭（见 6.5），link 由 PHY 硬件自动建立，`read_status` 只是读取上报——若 7.2 正常但始终无 link，重点查 **PHY 硬件/对端/线缆**，而非驱动配置。

## 7.4 Loopback 测试

用 PHY **内部回环**验证"MAC → PHY → MAC"数据通路是否正常，把问题与外部链路（线缆/对端）隔离开。

**目的：** 内部回环能通，说明 GMAC、RGMII、PHY 寄存器通路、驱动收发都正常；若回环不通，问题在板内（MAC/PHY 接口、delay 配置）。

**方法：**

## 7.4.1 进入loopback模式

使用mido工具写寄存器进入 internal loopback模式
mdio mmd_write ethX 3 0x0 0x4000 (bit 14位置1)

查询等待连接状态可用
mido mmd_read ethX 30 0x8102  #LINK_AVAILABLE=1

## 7.4.2 测试
设置网卡IP
ifconfig ethX 192.168.1.1 netmask 255.255.255.0 up

设置网卡静态ARP:绑定IP地址与MAC地址(ip指令不支持neigh的话，可以用arp命令)
ip neigh add 192.168.1.2 lladdr 00:55:7b:b5:7d:f7(网卡MAC地址) dev ethX nud permanent

通过网卡ping
ping  -I ethX 192.168.1.2

抓包
tcpdump -i ethX -n -e

连通性正常会抓到序号相同的ICMP包2次:

22:18:18.641561 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 2, length 64
22:18:18.641787 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 2, length 64
22:18:19.669600 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 3, length 64
22:18:19.669688 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 3, length 64
22:18:20.689576 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 4, length 64
22:18:20.689682 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 4, length 64
22:18:21.713544 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 5, length 64
22:18:21.713692 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 5, length 64
22:18:22.737555 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 6, length 64
22:18:22.737645 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 6, length 64
22:18:23.761612 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 7, length 64
22:18:23.761723 00:55:7b:b5:7d:f7 > 00:55:7b:b5:7d:f7, ethertype IPv4 (0x0800), length 98: 192.168.10.1 > 192.168.10.2: ICMP echo request, id 25, seq 7, length 64


**判读：**

- 回环通过 → 板内数据通路 OK，进入 7.5；
- 回环失败 → 优先检查 RGMII 信号质量与时序（2.1.3 节的 delay 问题）、GMAC 配置、寄存器读写（可用 7.2 的 mdio 读写工具验证寄存器）。

## 7.5 端对端连接测试

连接真实对端（对端 ECU 或测试工装），验证实际通信质量。

**方法：**

可以使用车载以太网转换器，将T1的端口转换成RJ45的端口与电脑连接进行端对端测试

```bash
# 1) 连通性
ping <对端IP>

# 2) 吞吐与稳定性
iperf3 -c <对端IP>          # 或 iperf，观察带宽与丢包

# 3) 统计/错误计数（含 PHY 硬件统计，见 5.3 ethtool 回调）
ethtool -S eth0

# 4) 信号质量 SQI（get_sqi，1000BASE-T1 链路质量）
#    驱动已打印 SIGNAL_QUALITY/MSE，或通过 ethtool 相关统计查看
```

**判读：**

- **ping 通、iperf 无丢包** → 链路正常，调试完成；
- **能 link、能 ping 但丢包/CRC 错误多** → 优先查 RGMII delay 配置（3.2、3.7）、线缆/连接器质量、对端协商速率；
- **SQI/MSE 异常** → 物理链路质量差，查线缆、连接器、板端阻抗匹配。

## 7.6 RGMII Delay 调试

RGMII 是 2.1 节介绍的双沿采样接口，**时钟与数据之间必须保持约 2ns 的相位关系（skew）**。这个相位由 PHY 或 MAC 侧的可调延迟（delay line）补偿，配错或取值不当会导致"能 link 但数据错乱"。本节讲如何定位和调整 RGMII delay。

### 7.6.1 为什么需要调 delay

**RGMII 是双沿采样（DDR）接口**：数据线 `TXD[3:0]` / `RXD[3:0]` 在时钟的**上升沿和下降沿都采样**。千兆时时钟为 125 MHz、周期 8 ns，相邻两个采样边沿间隔 4 ns，数据每 4 ns 更新一次。

为了保证接收端在采样边沿上能稳定读到位，要求**数据跳变落在相邻两个采样边沿之间的中间位置**，也就是数据边沿相对采样时钟边沿错开约 **2 ns**——这就是 RGMII 规范要求的时钟/数据相位关系（skew），也是需要 delay line 补偿的原因。

但这个 2 ns 相位不会自动成立，skew 主要来自三处：

1. **PCB 走线长度差**：时钟线与数据线走线长度不一致，信号到达时间有差异；
2. **芯片内部路径差**：MAC / PHY 内部，时钟路径与数据路径经过的逻辑和缓冲器不同，固有传播延迟不一致；
3. **工艺与温度（PVT）漂移**：芯片制造偏差、温度、电压变化会让固有 skew 在一定范围内变化。

因此两侧都提供**可调延迟（delay line）**，把时钟相对数据"移"到采样窗口中心，让数据在采样边沿两侧保留足够的**建立/保持时间余量**。MAC 侧（RK3576 GRF）和 PHY 侧（内部寄存器）都有，具体由哪侧补、补多少，就是 7.6 节要调的内容。

**不调或调错的后果**：数据跳变离采样边沿太近，采样余量变小——轻则偶发错包（CRC 错误、重传），重则完全采不到（link 通但 ping 不通、错误暴增）。速率越高、采样周期越短对时序越敏感，所以千兆比百兆更容易出问题。

**为什么必须"实测调整"而不是一次配好**：最佳 delay 值取决于具体的 PCB 走线、芯片批次与当前温区，无法仅靠理论精确算出，必须实测（7.6.4 节）找出错误为零的取值区间。

**什么现象会提醒你要调 delay：**

| 现象 | 说明 |
|------|------|
| link 能 up，但 ping 高丢包 / 不通 | 时序错位导致采样错误，最典型 |
| 千兆下大量 CRC / 对齐错误 | `ethtool -S eth0` 的 `rx_crc_errors`、`ifconfig` 的 RX errors 持续增长 |
| 低速正常、高速异常 | 速率越高、双沿周期越短（125 MHz），时序窗口越紧 |
| 批量/温度变化后出现 | 走线长度、器件批次差异会影响 skew，同一配置在不同板卡上表现不同 |

**排查顺序**：这类问题先按 7.6.2 确认 delay 由谁提供，再按 7.6.3 看当前配置，最后按 7.6.4 调整。

### 7.6.2 调之前：确认 delay 由谁提供（避免两侧叠加）

RGMII 的 2ns 相位补偿只能由**其中一侧**完成，否则两侧各加一遍，总延迟超限。先确认 `phy-mode`（3.2 节）：

| phy-mode | PHY 侧补偿 | MAC 侧（RK3576）该配什么 |
|----------|-----------|--------------------------|
| `rgmii` | 无 | **`tx_delay` 和 `rx_delay` 都要配** |
| `rgmii-id` | TX + RX | 都不配（`tx_delay`/`rx_delay` 注释掉） |
| `rgmii-txid` | 仅 TX | 只配 `rx_delay` |
| `rgmii-rxid` | 仅 RX | 只配 `tx_delay` |

> 本平台参考板（RK3576 EVB 系列）常用 **`rgmii-rxid`**：`tx_delay = <0x20>;` 生效、`rx_delay` 注释掉，注释明确写"Use rgmii-rxid mode to disable rx delay inside Soc"——即 RX 侧延迟交给 PHY 内部，SoC 只补 TX。

### 7.6.3 查看/设置当前 delay 配置

MAC侧读取
cat /sys/class/net/eth0/device/rgmii_delayline
tx delayline: 0xffffffff, rx delayline: 0xffffffff

MAC侧设置
echo "0x20 0x20" > /sys/class/net/eth0/device/rgmii_delayline

> 0xffffffff 表示未设置delay

PHY侧读取
mdio mmd_read eth0 0x30 0xAFCC
0x8012
mdio mmd_read eth0 0x30 0xAFCD
0x8012

PHY侧设置
mdio mmd_write eth0 0x30 0xAFCC 0x8012
mdio mmd_write eth0 0x30 0xAFCD 0x8012

> 0xAFCC是TX delay, 0xAFCD是 RX delay ,其中bit15表示使能delay，bit[4:0]表示delay的相位

### 7.6.4 调试步骤

**方法 A：软件观测法（先做，快速粗调）**

1. 建立压力：`iperf3 -c <对端IP>`，或 `ping -f -s 1472 -c 1000 <对端IP>`；
2. 记录基线错误：`ifconfig eth0` 与 `ethtool -S eth0`（盯 `rx_crc_errors` 等）；
3. 改设备树 `tx_delay` / `rx_delay`（取值 0x00~0x7F），重编 dtb、重启；
4. 重复测量，找出**错误为零的取值区间**，取中间值留裕量（考虑温度/批次）；
5. **一次只调一个方向**（先 TX 后 RX），两个方向别同时改，便于定位。

**方法 B：示波器测量法（精调/验证）**

1. **发送方向**：测 `GTX_CLK` 与 `TXD[3:0]`（或 TX_CTL）边沿。期望**数据跳变位于时钟周期的中心附近**，即数据边沿相对时钟边沿约 **2ns**；
2. **接收方向**：测 `RX_CLK` 与 `RXD[3:0]` 边沿，同样期望数据居中于时钟周期；
3. 调对应方向 delay 直到两侧采样点（setup/hold）居中且两端板卡都满足；若一侧没达到就换值，直到眼图余量最大。

> MAC侧调整也可以参考<Rockchip_Developer_Guide_Linux_GMAC_RGMII_Delayline_CN.pdf>

### 7.6.5 RK3576 delay 实现细节（供对照）

- **设备树属性**：`tx_delay` / `rx_delay`（u32）。dwmac-rk 读到就写入对应 GRF；**不写 = -1 = 禁用**（使能位不置）；
- **取值范围**：0x00~0x7F（7 bit）。全范围约覆盖 2ns 量级，**值越大延迟越大**；具体每 LSB 对应多少 ps 以 RK3576 手册的 delay line 参数为准；
- **参考起点**：RK3576 EVB 系列 gmac0 用 `tx_delay = <0x20>/<0x21>`（rgmii-rxid 模式），industry 板用 `0x1a/0x1b`。新板可先抄同 SoC 参考板值，再实测微调；
- 若想不改 dts 快速试值，可临时用 devmem 直写 GRF（注意 Rockchip GRF 写入格式：低 16 位=数据、高 16 位=写掩码），验证后固化到 dts。

### 7.6.6 注意点

- **两侧只允许一边加 delay**：若 PHY 已开内部延迟（`rgmii-id` 等），MAC 侧必须不配或置 0，否则叠加错乱（3.2.1 节）；
- **调 MAC delay 无效时**：怀疑 PHY 侧 delay 或链路本身，用 7.4 loopback 隔离本板通路再判断；
- **不要只看 link**：delay 问题通常 link 正常、数据错误，判断必须看错误计数和吞吐，不是看 link 状态；
- 取值确定后**留裕量**：不要贴边界值，避免温度/批次变化后劣化。

---

（本章完）

---

# 8. 常见问题

本章按 **现象 → 可能原因 → 排查步骤** 组织，收集开发调试中实际遇到的典型问题。各小节内容待补充。

## 8.1 log 提示 PHY ID 不匹配

[   35.377879] rk_gmac-dwmac 2a220000.ethernet eth0: __stmmac_open: Cannot attach to PHY (error: -22)

查看log nxp_c45_probe 函数是否被调用：
如果nxp_c45_probe没有调用，说明MDIO总线没有检测到PHY排查硬件，参考7.1节。
如果nxp_c45_probe被调用，查看log(dmesg | grep TJA )是否有异常 如PHY信息寄存器读取失败，可能原因是nxp_c45_soft_reset复位后需要加延时。
如果nxp_c45_probe被调用，log无其他异常，可能内核genphy_c45_pma_read_abilities获取PHY能力失败，需要手动添加设置	linkmode_set_bit(ETHTOOL_LINK_MODE_1000baseT1_Full_BIT, phydev->supported);

## 8.2 提示 DMA 初始化失败

gmac驱动打印: DMA engine initialization failed

一般可能是PHY的RXC时钟异常，有无，频率，幅值。
也可能是GMAC内部时钟配置异常。

参考文档<Rockchip_Developer_Guide_linux_GMAC_CN>

## 8.3 RX 收包异常

现象：端对端抓包发现PHY的TX包对端都可以收到，对端发送的包，偶尔可以收到，进行internal loopback测试，发现也是偶尔可以收到相同序号的ICMP包，无论怎么调整双方delay都无法改善。

最后排查发现是PHY的RX信号幅值较小，VDDIO用的是1V8而，SOC用的是3V3，修改硬件后，解决这一问题。
同样硬件用VDDIO 1V8 ，TJA1103 SOC初始化时不会报错8.2节的DMA错误，TJA1120就会报，漏过了排查这一点。

---

（本章完）


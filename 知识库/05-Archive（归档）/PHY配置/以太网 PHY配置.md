

SMI：串行管理接口（Serial Management Interface），通常直接被称为MDIO接口（Management Data Input/Output Interface）管理数据输出/输入。MDC（Management Data Clock）：管理数据时钟，由MAC（或管理实体）发出，为MDIO上的数据传输提供时钟参考。主要被应用于以太网的MAC和PHY层之间，用于MAC层器件通过读写寄存器来实现对PHY层器件的操作与管理。

MDIO帧格式（64bit）：
前导码：32bit 
标志位：2bit 必须是01 标志该数据帧开始。
操作码：2bit 10为读操作，01为写操作。
PHY地址：5bit 表示所访问的PHY地址，一个MDIO总线最大支持32个PHY。
寄存器地址：5bit 表示所访问的寄存器的地址，共32个寄存器。
翻转标志位：2bit 固定为10 该标志位为PHY芯片地址传输和数据传输处理预留处理时间，同时防止总线存在冲突。
数据：16bit 操作符为读操作，该数据为对于地址PHY的特定寄存器的数值；操作符为写时，该数据为对该寄存器写入的数值。

扩展帧格式（一个总线只有一个PHY芯片，增加一个DEVAD MMD ）：（间接寻址：完整的读写操作需要发送两帧）64bit 
前导码（32）+ 帧起始（00）+ 操作码（xx）（00：地址帧  01：写数据帧 11：读数据帧 10：读后地址自增帧）+ 端口地址（xxxxx）+ 设备地址（xxxxx）+ 周转（xx）+ 寄存器地址/数据（16）

MMD(MDIO Manageable Device)结构，如MMD0（控制）、MMD1（状态）、MMD3（诊断）等。驱动中需要区分当前 PHY 使用 Clause 22 还是 Clause 45，TJA1103 / TJA1120 的核心配置寄存器主要位于 MMD 空间。


PHY(Physical Layer Transceiver)：物理层收发器，负责将MAC（媒体访问控制器）传来的数字信号转化为在网线上传输的模拟信号。

/home/ssd12/huangziye/rk3576-linux/kernel/drivers/net/phy/nxp-c45-tja11xx.c
内核源码中有nxp tja1120 1103 的源码 我们主要是配置，正确的编写设备树。


百兆速率接口（10/100Mbps）
MII (Media Independent Interface)：最原始的“媒体独立接口”，是后续所有接口的基础。它有16根信号线，数据总线为4位。
RMII (Reduced MII)：为减少引脚而设计，是MII的简化版。它将数据总线从4位减为2位，总信号线降至7根。它需要一个独立的50MHz外部时钟源。
SMII (Serial MII)：由思科（Cisco）定义，是一种串行接口，数据总线仅为1位，可进一步减少引脚。

千兆速率接口 (1000 Mbps)
GMII (Gigabit MII)：千兆版的MII，数据总线扩展为8位，时钟频率提升至125MHz，但代价是引脚数增至24根。
RGMII (Reduced GMII)：正如我们上次讨论的，它是GMII的简化版。通过将数据总线减为4位并采用双沿采样（DDR） 技术，在125MHz时钟下实现了千兆速率。（1103 也能使用，可以向下兼容）
SGMII (Serial GMII)：一种串行千兆接口，使用SerDes（串行器/解串器）技术。它仅需两对差分信号线（收和发），引脚极少，适合高速、长距离传输。
TBI (Ten-Bit Interface)：一种10位接口，主要用于连接MAC与PHY中的SerDes部分。

更高速率及特殊接口 (≥10 Gbps)
QSGMII (Quad SGMII)：可以理解为4路SGMII的集合。它将4个千兆端口的数据汇聚成一个5 Gbps的串行数据流，通过一对差分线传输，非常适合高密度交换机设计。
XGMII (10-Gigabit MII)：万兆以太网的并行接口，数据总线宽度达32位，但引脚数也高达74个。
XAUI (XGMII Attachment Unit Interface)：XGMII的串行化版本，目的是减少XGMII庞大的引脚数。它将32位并行数据分为4条SerDes通道，每条速率3.125 Gbps，总带宽达到10 Gbps。
更高速接口：对于40G/100G及更高速率，还有XLGMII、XLAUI、CAUI等接口。


TX_CLK：由PHY芯片提供给MAC芯片（百兆十兆）

GTX_CLK：MAC芯片提供给PHY芯片 （千兆）

为了满足芯片内部建立时间（Setup Time）和保持时间（Hold Time）的物理电气要求，如果不配或者配错现象往往是 链路显示up，但是ping不通，或者丢包率极高。
原因：双边沿采样，如果数据和时钟严格对齐，接收方在上升沿去抓数据时，数据恰好“跳变到一半”，根本抓不到稳定值，直接就抓到亚稳态了，导致 CRC 校验错误。延时导致采样避开了跳变区域。

相位延迟由谁来加：
rgmii：PHY 侧（内部）延时  不加 必须由mac侧配置延时（tx_delay/rx_delay）
rgmii-id：PHY侧 TXRX都加  MAC侧不加 
rgmii-txid：PHY侧仅加TX  MAC侧仅加rx_delay
rgmii-rxid：PHY侧仅加RX  MAC侧仅加tx_delay

要点：-id  -txid -rxid 后缀的含义是"该方向的延迟由 PHY 内部寄存器补"。此时 MAC 对应方向不应再加延迟，否则两侧各加一遍，总延迟超限，导致高速收发错位、大量 CRC 错误。
配置必须与实际硬件/芯片能力匹配，若PHY不支持内部延迟但是却配置了rgmii-id，驱动虽然会给PHY写延迟寄存器，但是PHY不生效，需改由MAC端配置“tx_delay rx_delay ”


RK3576 内部集成千兆 GMAC，RGMII 引脚一般为**复用引脚**，需在设备树 `pinctrl` 中正确配置。

兼容模式是历史遗留做法，不建议这样做
snps,reset-gpio = <&gpio2 RK_PC6 0>; 
snps,reset-active-low;

现代内核5.10+的标准做法
snps,reset-gpio = <&gpio2 RK_PC6 GPIO_ACTIVE_LOW>;


”clock_in_out“ = “output” 就是MAC给PHY时钟 GTX_CLK  input就是PHY或者晶振提供，mac作为输入使用。

时钟、`phy-mode`、`pinctrl` 三者必须成套配置，不能只改其中一项。

GMAC/MAC -> PCS（物理编码子层）->PMA（物理介质相关子层）-> PMD（物理介质相关子层）->物理介质（网线/双绞线）
MMD 1（PMA/PMD）  MMD3 PCS

数据手册：
![[Pasted image 20260901160842.png]]

![[Pasted image 20260901161214.png]]

![[Pasted image 20260901161234.png]]


![[Pasted image 20260901193450.png]]



![[Pasted image 20260901194113.png]]
这个Dh：4：0指的是800Dh寄存器里面的低5位指向的MMD
这个寄存器写入的地址指的就是MMD里面的地址
![[Pasted image 20260901195058.png]]
如果0x0D（TABLE 7 ）14 15位是00 这个寄存器就是存的就是地址 反之是数据 

![[Pasted image 20260901195441.png]]
![[Pasted image 20260901195454.png]]
bit 8  serdes链路  SGMII_LS_EVENT   	SGMII（串行千兆媒体独立接口）与外部 MAC/交换机之间的链路可用性发生翻转（插拔光纤或 SerDes 失锁）。

bit 7 PTP 同步时钟（硬件时间戳） SYNC_TS_IRQ   检测到外部输入的 SYNC 脉冲信号（如 1PPS），且硬件已将时间戳存入 PTP_SYNC_TS 缓冲区，等待读取。

bit 6  PTP 接收时间戳 INGRESS_TS_IRQ 收到了带 PTP（IEEE 1588）协议的以太网帧，且硬件已捕获该帧的精确接收时刻，存入入口缓冲区。

bit 5	 PTP 发送时间戳 INGRESS_TS_IRQ  发送了 PTP 帧，硬件捕获了该帧离开 PHY 的精确时刻，存入出口缓冲区。

bit 4  电源管理 / 唤醒 WAKE_SLEEP_EVENT PHY 的 Wake-Sleep 状态位发生变化（常用于汽车待机模式、远程唤醒（Wake-on-LAN）或本地唤醒）。

bit 3 物理链路层（核心） LINK_AVAILABLE_EVENT 这是最常看的中断——100BASE-T1 双绞线物理链路的 Link_available 状态发生改变（网线插拔、协商成功/失败）。

bit 2 保留

bit 1 端口级汇总（只读）PORT_IRQ  只要这个 PORT 内有任何未处理的中断，该位就为 1。MCU 中断服务程序（ISR）通常先读这个位来判断是否是本端口触发。

bit 0  全局汇总（只读） GLOBAL_IRQS 只要芯片内部全局中断树有任何挂起中断，该位就为 1。若芯片支持多端口，可通过此位快速跳过无中断的端口扫描。


![[Pasted image 20260901201858.png]]
bit 15 WU_SMI SMI总线远程唤醒使能，SOC通过MDIO写1，可以将处于休眠状态的PHY芯片唤醒
bit 14：5 保留
bit 4 REFCLK_WARN 参考时钟频率异常警告。 置 1 表示 SoC 提供给 PHY 的参考时钟频率偏离了有效范围（需在 12.5MHz - 37.5MHz 之间），此时 MDIO 通信可能不可靠
bit 3 CORE_SUPPLY_WARN 内核供电告警记录。 置 1 表示检测到内核电源发生过故障（需写 1 清除此记录）
bit2 CORE_SUPPLY_STATUS 内核供电实时状态（只读）。 1 = 内核供电正常，所有寄存器均可访问  0 = 内核供电异常，SoC 无法正常读写大部分寄存器 
bit1 LIMITED_ACCESS  当前访问权限状态（只读）。 该位含义跟随供电状态动态变化：当供电异常（Bit2=0）时：置 1 表示只允许访问 0x1F 和 0x801F，其余寄存器被硬件隔离锁死。当供电正常（Bit2=1）时：置 1 表示所有寄存器可读，但只有 0x1F 和 0x801F 允许写入（其他配置寄存器被写保护）。
bit 0  INHIBIT_STATUS INHIBIT 引脚电平状态透传（只读）。 1 表示 PHY 当前正将 INHIBIT 引脚驱动为高电平（用于向 SoC 或外部电路指示禁止发送/上电状态）。

CL45：MMD  0为保留 1为PMA/PMD 3为PCS 30为NXP专用   其他都为保留





PHY 地址配置（3.10.3，CONFIG0/1/2） 地址1~27  但是未查看到PHY address pin-strapping 数据手册，只能通过别的办法看reg值 
![[Pasted image 20260902143031.png]]
硬件图表示config0 1 都是open的 就是o   config 2 也是open  是o  但是找不到映射表只能想别的办法

找到了映射表，在另一个数据手册中。
![[Pasted image 20260903100140.png]]


非必要不要清理output  

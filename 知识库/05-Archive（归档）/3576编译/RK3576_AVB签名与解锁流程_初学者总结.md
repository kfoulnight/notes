---
title: RK3576 AVB 签名与解锁流程：初学者总结
cssclasses:
  - archive-page
---

# RK3576 AVB 签名与解锁流程：初学者总结

原始文档：<https://kb.cvte.com/pages/viewpage.action?pageId=553292704>

这篇笔记是给初学者看的。重点不是背每一条命令，而是先理解 AVB 是什么、为什么要签名、设备启动时怎么验签、解锁流程又是在解决什么问题。

> [!summary] 核心主线
> AVB 用签名和证书链保证设备启动的镜像没有被篡改。设备启动时先验证信任链和 vbmeta，再验证 boot 和 system。如果需要解锁，也必须通过授权密钥和 challenge 签名证明这个解锁请求是可信的。

## 一、这篇文档在讲什么

这篇文档讲的是 RK3576 平台上的 Android AVB 流程，主要分成两部分。

### AVB 签名与验签

固件发布前，要对 boot.img、system.img、vbmeta.img 等镜像做签名。设备启动时，Bootloader 和 Kernel 会检查这些镜像有没有被篡改。验证通过才允许继续启动。

### AVB 解锁

正常安全状态下，设备不允许启动被篡改或未授权的镜像。解锁流程用于研发调试、售后维修、工厂处理等授权场景。

但解锁不是随便跳过安全机制。设备仍然会通过证书链和 challenge 签名确认：这个解锁请求确实被授权。

> [!quote] 一句话理解
> AVB 是 Android 的启动安全机制，用来保证设备启动的系统镜像是厂家授权且没有被篡改的；解锁流程是在特定授权条件下，让设备进入可调试或可修改状态。

## 二、先理解几个核心概念

### AVB

AVB 全称是 Android Verified Boot，中文可以理解为 Android 启动验证。

它解决的问题是：

1. 防止别人修改 boot.img，比如替换内核或 ramdisk。
2. 防止别人修改 system.img，比如植入恶意系统文件。
3. 防止 A 产品线的固件刷到 B 产品线上。
4. 防止旧版本、低安全版本的密钥或固件被回滚使用。

AVB 的基本思想很简单：

> 固件发布前先签名；设备启动时再验签。签名对得上，就认为可信；对不上，就拒绝启动。

### 签名和验签

可以把签名理解成厂家盖章。

厂家用私钥给镜像签名。设备里保存对应的公钥，或者保存能验证到这个公钥的信任链。设备启动时用公钥验证签名。如果镜像被改过，签名就对不上。

### 私钥

厂家保管的秘密钥匙，用来签名，不能泄露。

### 公钥

可以放到设备里，用来验证签名。

### 签名

用私钥对数据生成的可信证明。

### 验签

用公钥确认签名是否正确。

### vbmeta.img

vbmeta.img 是 AVB 中非常关键的顶级验证镜像。

它本身通常不存放系统文件，而是存放验证其他镜像所需的信息，例如：

- boot.img 的哈希摘要。
- system.img 的哈希树根哈希。
- dm-verity 的内核参数。
- 用于验证签名的公钥信息。
- ATX 证书链元数据。

可以这样理解：

> vbmeta.img 像一张验证清单。Bootloader 先验证这张清单是真的，然后再按清单去验证 boot、system 等分区。

### Hash Footer 和 Hashtree Footer

文档里有两个常见命令：add_hash_footer 和 add_hashtree_footer。

它们都是给镜像追加 AVB 校验信息，但适用场景不同。

#### Hash Footer

常用于 boot.img。boot.img 比较小，可以对整个镜像算一个整体 hash。

#### Hashtree Footer

常用于 system.img。system.img 很大，运行时会读取不同 block，所以需要哈希树逐块验证。

> [!tip] 简单记法
> boot 小，用整体 hash。
> system 大，用 hashtree。

### dm-verity

dm-verity 是 Linux 内核里的块设备完整性校验机制。

它的作用是：系统运行时，每次读取 system 分区的某个 block，内核都会根据哈希树检查这个 block 有没有被改。如果 hash 不匹配，就说明数据被篡改。

AVB 和 dm-verity 的关系是：

- AVB 在启动阶段验证 vbmeta.img 和 boot.img。
- vbmeta.img 里包含 system.img 的 root hash 和 dm-verity 参数。
- 内核启动后，根据这些参数继续验证 system 分区。

### ATX

文档里的 ATX 是 Android Things Extension 的 AVB 扩展机制。这里主要用它实现更复杂的信任链和产品绑定。

普通理解就是：

> ATX 让 AVB 不只是验证一个公钥，还能验证“根密钥 -> 中间密钥 -> 产品密钥”的证书链，并且绑定 Product ID。

## 三、关键文件和它们的作用

### PRK、PIK、PSK 三把核心签名密钥

文档中先生成三对 RSA 4096 密钥。

> [!info] 三把核心签名密钥速查
> | 文件 | 缩写 | 作用 | 类比 |
> |---|---|---|---|
> | testkey_atx_prk.pem | PRK | 根认证密钥，最高信任锚点 | 根 CA |
> | testkey_atx_pik.pem | PIK | 中间密钥，由 PRK 签发 | 中间 CA |
> | testkey_atx_psk.pem | PSK | 产品签名密钥，实际签 boot/system/vbmeta | 业务证书 |

它们的关系是：

1. PRK 根密钥授权 PIK。
2. PIK 授权 PSK。
3. PSK 给 boot.img、system.img、vbmeta.img 签名。

> [!warning] 最重要的是
> 量产时这些私钥必须放在安全的 PKI 环境中，不能直接放到普通编译机或代码仓库里。

### Product ID

atx_product_id.bin 是产品 ID 文件，长度是 16 字节。

它的作用是绑定产品线。

A 产品的固件只能给 A 产品使用。B 产品的设备如果拿到 A 产品固件，Product ID 校验会失败。

Product ID 会出现在两个地方：

1. atx_permanent_attributes.bin 中。
2. PSK 或 PUK 证书的 subject 中。

设备启动或解锁时，会比较两边的 Product ID 是否一致。

### atx_metadata.bin

atx_metadata.bin 是 ATX 元数据，里面打包了 PIK 证书和 PSK 证书。

它会被嵌入 vbmeta.img。

设备启动时，Bootloader 会从 vbmeta.img 里取出 atx_metadata.bin，然后按这个顺序验证证书链：

1. 用设备里的 PRK 公钥验证 PIK 证书。
2. 用 PIK 公钥验证 PSK 证书。
3. 用 PSK 公钥验证 vbmeta.img 签名。

### atx_permanent_attributes.bin

atx_permanent_attributes.bin 是设备端的永久属性文件，包含版本号、Product ID 和 PRK 根公钥。

它会被烧录到设备的安全存储区域，例如 RK3576 中的 RPMB / secure 分区。

文档里还提到，会把根公钥 hash 烧录到 OTP 中。

> [!danger] 这里要特别注意
> OTP 是一次性烧录区域，一旦写入通常不可更改。它用于保证设备信任锚点不能被随便替换。

### vbmeta.img

vbmeta.img 是顶级验证镜像。

里面主要包含：

- boot 分区的 hash descriptor。
- system 分区的 hashtree descriptor。
- dm-verity cmdline。
- PSK 公钥。
- ATX metadata，也就是 PIK / PSK 证书链。
- vbmeta.img 自己的签名。

Bootloader 会优先验证 vbmeta.img。只要 vbmeta.img 被验证可信，里面记录的 boot 和 system 验证信息也就可信。

## 四、AVB 签名前置准备

签名前要先准备信任链和设备端信任锚点。

### 整体过程

1. 生成 PRK、PIK、PSK 三对 RSA 密钥。
2. 创建 PIK 证书需要的空 subject 占位文件 temp.bin。
3. 用 PRK 签发 PIK 证书。
4. 用 PIK 签发 PSK 证书。
5. 把 PIK 证书和 PSK 证书打包成 atx_metadata.bin。
6. 生成 atx_permanent_attributes.bin，里面包含 Product ID 和 PRK 根公钥。
7. 把永久属性烧录到设备安全区域。

> [!abstract] 可以这样记
> 先生成 PRK / PIK / PSK。
> PRK 签发 PIK 证书。
> PIK 签发 PSK 证书。
> PIK 证书和 PSK 证书组成 atx_metadata.bin。
> Product ID 和 PRK 公钥组成 atx_permanent_attributes.bin。

完成后的结果：

- 设备端有了不可轻易篡改的信任锚点。
- 固件侧有了可以嵌入 vbmeta.img 的证书链。
- 后续可以用 PSK 对镜像进行签名。

## 五、AVB 签名流程

### boot.img 签名

boot.img 使用 avbtool add_hash_footer 签名。

签名后，boot.img 后面会追加这些内容：

- VBMeta Blob。
- Hash Descriptor。
- PSK 公钥。
- RSA 签名。
- AVB Footer。

> [!note] 简单理解
> boot 镜像尾部被加了一段“校验说明和签名”。启动时 Bootloader 可以根据这些信息确认 boot 是否被改过。

partition_size 要注意三点：

1. 必须大于原始镜像加 AVB 元数据空间。
2. 必须 4K 对齐。
3. 不能超过真实物理分区大小。

### system.img 签名

system.img 使用 avbtool add_hashtree_footer 签名。

它和 boot 的区别是：system 分区很大，不能只靠启动时整体 hash 验证，需要构建哈希树，让内核运行时逐块验证。

签名后，system.img 里会追加这些内容：

- Hash Tree。
- 可选 FEC 数据。
- VBMeta Blob。
- Hashtree Descriptor。
- AVB Footer。

> [!note] 简单理解
> system 镜像被加上了一棵哈希树，内核之后可以靠这棵树检查每个数据块有没有被篡改。

### 生成 vbmeta.img

最后用 avbtool make_vbmeta_image 生成 vbmeta.img。

它会做这些事：

1. 从 boot.img 提取 hash descriptor。
2. 从 system.img 提取 hashtree descriptor。
3. 从 system.img 生成 dm-verity cmdline。
4. 嵌入 ATX metadata，也就是证书链。
5. 用 PSK 私钥给 vbmeta.img 本身签名。

> [!info] 最终结构
> vbmeta.img 里有自己的签名，也有 boot.img 的验证信息、system.img 的验证信息、dm-verity 参数和 ATX 证书链。

结果是：

> 设备启动时只要先验证 vbmeta.img 可信，就可以继续信任里面记录的 boot 和 system 验证信息。

## 六、AVB 验签流程

设备启动时，验证分成两个阶段。

### Bootloader 阶段

Bootloader 负责验证启动早期必须可信的内容。

大致顺序是：

1. 读取 atx_permanent_attributes.bin。
2. 用 OTP 中的 hash 确认 permanent attributes 没被改。
3. 读取 vbmeta.img。
4. 从 vbmeta.img 中取出 ATX metadata。
5. 用 PRK 公钥验证 PIK 证书。
6. 用 PIK 公钥验证 PSK 证书。
7. 检查 Product ID 是否匹配。
8. 用 PSK 公钥验证 vbmeta.img 签名。
9. 从 vbmeta.img 中取 boot hash descriptor。
10. 验证 boot.img 是否完整。
11. 启动内核，并传入 dm-verity 参数。

> [!danger] 任何一步失败
> 如果这里任何一步失败，设备通常会拒绝启动。

### Kernel 阶段

内核启动后，主要负责验证 system 分区。

流程是：

1. Bootloader 把 dm-verity 参数传给内核。
2. 内核以 dm-verity 模式挂载 system 分区。
3. 每次读取一个 4KB block，内核都检查它的 hash。
4. hash 一直向上验证到 root hash。
5. root hash 来自已经被 Bootloader 验证过的 vbmeta.img。

所以内核信任这个 root hash。

> [!note] 结果
> 即使系统启动后，有人试图改 system 分区中的某个 block，读取时也会被 dm-verity 发现。

### 验签失败会怎样

> [!danger] 失败点与结果
> | 失败点 | 可能原因 | 结果 |
> |---|---|---|
> | permanent attributes 校验失败 | RPMB 数据被改，或 OTP hash 不匹配 | 拒绝启动 |
> | PIK / PSK 证书验证失败 | 证书不是合法上级签发 | 拒绝启动 |
> | Product ID 不匹配 | 固件和设备产品线不一致 | 拒绝启动 |
> | vbmeta 签名失败 | vbmeta.img 被改过或签名不对 | 拒绝启动 |
> | boot hash 失败 | boot.img 被改过 | 拒绝启动 |
> | system dm-verity 失败 | system 分区某些 block 被改过 | 拒绝启动或进入错误状态 |

## 七、AVB 解锁流程

### 为什么需要解锁

AVB 正常工作时，设备只信任官方签名镜像。

但研发和调试时，可能需要刷入临时调试镜像、修改 boot 或 system、排查启动和驱动问题。

这时就需要解锁。

不过解锁不能变成安全漏洞，所以文档设计了一套授权解锁流程。

### PUK

解锁流程会生成一把新的密钥：PUK，也就是 Product Unlock Key。

- testkey_atx_puk.pem：产品解锁私钥。
- puk_certificate.bin：PUK 公钥证书，由 PIK 签发。

PUK 证书有一个关键用途参数：

`--usage=com.google.android.things.vboot.unlock`

它说明这张证书的用途是“解锁”，不是普通启动签名。

> [!warning] 这里要记住
> PSK 用于签固件，PUK 用于授权解锁，两者用途不同，不应该混用。

### 解锁整体过程

解锁过程可以理解为：设备出题，PC 端用授权私钥答题。

整体步骤：

1. 设备进入 fastboot。
2. 设备生成 unlock challenge。
3. PC 读取 challenge。
4. PC 校验 Product ID，并提取随机数。
5. PC 用 PUK 私钥对随机数签名。
6. 生成 unlock_credential.bin。
7. 通过 fastboot stage 上传到设备。
8. 设备验证证书链和签名。
9. 验证通过后解锁成功。

### raw_unlock_challenge.bin

raw_unlock_challenge.bin 是设备生成的 challenge，长度是 52 字节。

它的结构是：

- 前 4 字节是 challenge 版本号。
- 中间 32 字节是 SHA-256(atx_product_id.bin)。
- 最后 16 字节是随机数，或者 CPUID。

PC 端脚本会做两件事：

1. 验证 challenge 里的 Product ID hash 是否正确。
2. 提取最后 16 字节作为 unlock_challenge.bin。

### unlock_credential.bin

unlock_credential.bin 是最终上传到设备的解锁凭证。

里面包含：

- PIK 证书。
- PUK 证书。
- PUK 私钥对 challenge 的签名。

设备端验证时会检查：

1. PRK 是否能验证 PIK 证书。
2. PIK 是否能验证 PUK 证书。
3. PUK 公钥是否能验证 challenge 签名。
4. challenge 是否和设备之前生成的一致。
5. Product ID 是否匹配。

都通过才解锁。

### 解锁最容易踩的坑

> [!danger] 不能断电
> 获取 challenge 后，到执行 fastboot oem at-unlock-vboot 前，设备不能断电、关机、重启。

原因是：默认模式下 challenge 里包含随机数。随机数保存在设备当前 fastboot 会话状态中。如果设备重启，随机数会丢失或变化，之前生成的 unlock_credential.bin 就失效了。

所以默认随机数模式下，解锁凭证通常是一次性的。

### OTP / CPUID 模式

如果内核开启 CONFIG_MISC=y 和 CONFIG_ROCKCHIP_OTP=y，challenge number 可以从随机数变成设备 CPUID。

> [!example] 两种解锁模式对比
> | 对比项 | 默认随机数模式 | OTP / CPUID 模式 |
> |---|---|---|
> | challenge 来源 | 每次新随机数 | 固定 CPUID |
> | 凭证能否重复使用 | 不能 | 可以 |
> | 断电后是否失效 | 会失效 | 不会失效 |
> | 安全性 | 更高 | 较低 |

## 八、最终结果

### 签名与启动验证结果

完成签名后，会得到这些关键文件。

> [!success] 签名与启动验证产物
> | 文件 | 结果 |
> |---|---|
> | boot.img | 已追加 Hash Footer，可被 Bootloader 验证 |
> | system.img | 已追加 Hashtree Footer，可被 dm-verity 验证 |
> | vbmeta.img | 顶级验证镜像，记录 boot 和 system 的验证信息 |
> | atx_metadata.bin | 证书链元数据，嵌入 vbmeta.img |
> | atx_permanent_attributes.bin | 设备端信任锚点，烧录到安全区域 |

设备启动时可以做到：

1. 确认永久属性没被改。
2. 确认证书链可信。
3. 确认 vbmeta.img 没被改。
4. 确认 boot.img 没被改。
5. 通过 dm-verity 确认 system.img 运行时没被改。

### 解锁结果

解锁流程会生成并使用这些文件。

> [!tip] 解锁产物文件
> | 文件 | 结果 |
> |---|---|
> | testkey_atx_puk.pem | 解锁私钥 |
> | puk_certificate.bin | 解锁证书 |
> | raw_unlock_challenge.bin | 设备生成的 challenge |
> | unlock_challenge.bin | 从 challenge 提取的随机数或 CPUID |
> | unlock_credential.bin | 最终上传到设备的解锁凭证 |

设备验证通过后，会进入解锁状态。之后可以进行研发调试或刷入特定镜像。

## 九、完整主线

如果只记一条 AVB 签名和启动验证主线，记这个：

1. 厂家先生成根密钥 PRK。
2. PRK 授权 PIK。
3. PIK 授权 PSK。
4. PSK 签名 boot、system、vbmeta。
5. 设备里烧录 PRK 公钥和 Product ID。
6. 设备启动时验证 PRK -> PIK -> PSK -> vbmeta -> boot / system。
7. 全部通过才启动。

解锁主线是：

1. PIK 授权 PUK。
2. 设备生成 challenge。
3. PC 用 PUK 私钥签 challenge。
4. 设备验证 PRK -> PIK -> PUK -> challenge 签名。
5. 验证通过后解锁。

## 十、学习顺序建议

不要一开始就陷进命令参数。建议按这个顺序学：

1. 先理解 AVB 的目的：防止启动镜像和系统分区被篡改。
2. 再理解 vbmeta.img：它是顶级验证清单。
3. 再理解信任链：PRK -> PIK -> PSK。
4. 再理解 Product ID：防止固件跨产品线使用。
5. 再理解 boot 和 system 的区别：boot 用整体 hash，system 用哈希树。
6. 再理解 Bootloader 和 Kernel 分工：Bootloader 验启动链，Kernel 验 system block。
7. 最后再看解锁：设备出 challenge，PC 用 PUK 私钥签名，设备验签后解锁。

## 十一、容易混淆的点

> [!faq]- 混淆点速查（点击展开）
> | 容易混淆的问题 | 正确理解 |
> |---|---|
> | PRK、PIK、PSK 都是干什么的 | PRK 是根，PIK 是中间，PSK 是实际签固件的产品密钥 |
> | atx_metadata.bin 和 atx_permanent_attributes.bin 的区别 | 前者放证书链并嵌入 vbmeta，后者放设备端永久信任锚点 |
> | vbmeta.img 是不是系统镜像 | 不是，它主要是验证信息清单 |
> | boot 和 system 为什么签名方式不同 | boot 小用整体 hash，system 大用 hashtree 支持运行时逐块验证 |
> | Product ID 的作用 | 绑定产品，防止固件刷错产品线 |
> | 解锁是不是跳过所有安全 | 不是，解锁本身也需要证书链和 PUK 签名授权 |
> | 默认解锁凭证为什么不能复用 | 因为 challenge 是随机数，重启或断电后会变化或丢失 |

## 十二、安全注意点

### 私钥不能泄露

PRK、PIK、PSK、PUK 都是敏感私钥。量产环境必须放在 PKI 或安全签名服务中。

### OTP 烧录不可逆

根公钥 hash 一旦烧录到 OTP，通常不能更改。烧录前必须确认密钥、Product ID 和流程没有问题。

### Product ID 要规划好

每条产品线应该有清晰、唯一的 Product ID。否则后面可能出现固件无法启动或跨产品混乱。

### partition_size 不能乱填

必须满足镜像空间、4K 对齐、不能超过物理分区大小。填错可能导致签名失败或刷机后启动失败。

### 解锁配置必须确认

文档提到 U-Boot 中需要开启严格安全校验：

`CONFIG_RK_AVB_LIBAVB_ENABLE_ATH_UNLOCK=y`

如果不开启，可能出现输入 fastboot oem at-unlock-vboot 就能直接解锁的风险。

## 最后记住

> [!quote] 一句话总结
> RK3576 上的 AVB 通过 PRK -> PIK -> PSK 证书链建立固件信任关系。
> vbmeta.img 负责统一记录 boot 和 system 的验证信息。
> 设备启动时，Bootloader 先验证信任链和 boot，Kernel 再通过 dm-verity 验证 system。
> 如果需要解锁，则通过 PIK 授权的 PUK 对设备 challenge 签名。设备验证通过后，才允许解锁。

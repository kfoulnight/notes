---
title: Android 编译、打包与镜像生成基础流程
tags:
  - android
  - rockchip
  - rk3576
  - build
  - embedded
aliases:
  - Android 编译流程
  - build.sh 笔记
created: 2026-07-31
platform: Rockchip RK3576 / Android 14
cssclasses:
  - archive-page
---

# Android 编译、打包与镜像生成基础流程

这篇笔记是给初学者看的，重点不是记住每一个参数，而是先理解 Android 固件是怎么从源码变成 `update.img` 的。

> [!summary] 核心主线
> 先加载编译环境，再选择产品配置，然后 `build.sh` 按阶段编译 U-Boot、Kernel、Android，最后整理镜像并打包成 Rockchip 可以烧录的 `update.img`。

## 一、整体流程

1. `source build/envsetup.sh`：加载 Android 编译环境，让 `lunch`、`make` 等命令可用。
2. `lunch <product>-<variant>`：选择要编译的产品和版本类型，例如 `rk3576s_u-userdebug`。
3. `get_build_var`：`build.sh` 通过它读取当前产品的配置，例如 DTS、Kernel config、U-Boot config、AB 开关等。
4. `./build.sh -U -K -A -u`：根据参数执行不同阶段的编译。
5. 编译 U-Boot：生成 bootloader 相关镜像。
6. 编译 Kernel：生成 Kernel Image、DTS、驱动模块和 `resource.img`。
7. 编译 Android：生成 `boot.img`、`system.img`、`vendor.img`、`product.img`、`super.img`、`vbmeta.img` 等分区镜像。
8. `mkimage.sh`：把 `out` 目录里的镜像整理到 `rockdev/Image-<product>/`。
9. `mkupdate.sh`：把多个分区镜像打包成最终的 `update.img`。

> [!success] 最终目标
> 最终最常用的烧录文件是 `rockdev/Image-<product>/update.img`。

## 二、关键目录

> [!info] 关键产物目录
> | 路径 | 含义 |
> |---|---|
> | `out/target/product/<product>/` | Android make 的原始产物目录，包含分区镜像和中间产物 |
> | `rockdev/Image-<product>/` | Rockchip 整理后的最终镜像集合，烧录单分区时从这里找 |
> | `rockdev/Image-<product>/update.img` | Rockchip 整包烧录固件，给 RKDevTool 等工具使用 |

## 三、配置从哪里来

`build.sh` 里很多变量不是脚本自己写死的，而是来自 `lunch` 选中的产品配置。

常见配置目录：`device/rockchip/rk3576/<customer>/`

### 客户目录三件套

> [!info] 三个核心文件
> | 文件 | 作用 |
> |---|---|
> | `AndroidProducts.mk` | 决定 `lunch` 能选择哪些产品 |
> | `BoardConfig.mk` | 板级硬件配置，例如 DTS、Kernel config、U-Boot config、分区大小、AB 开关、传感器 |
> | `<product>.mk` | 产品级配置，例如预装 app、overlay、API level、`PRODUCT_NAME` |

### 常见变量

> [!info] get_build_var 常见变量
> | 变量 | 作用 | 示例 |
> |---|---|---|
> | `TARGET_PRODUCT` | 当前产品名 | `rk3576s_u` |
> | `TARGET_BUILD_VARIANT` | 当前编译类型 | `userdebug` |
> | `PRODUCT_KERNEL_DTS` | Kernel 使用的 DTS | `rk3576s-tablet-v10` |
> | `PRODUCT_KERNEL_CONFIG` | Kernel 配置片段 | `pcie_wifi.config` |
> | `PRODUCT_UBOOT_CONFIG` | U-Boot 配置 | `rk3576_defconfig` |
> | `BOARD_USES_AB_IMAGE` | 是否启用 AB 分区 | `true` / `false` |
> | `BOARD_BUILD_GKI` | 是否启用 GKI | `false` |

> [!danger] 编译前一定先 lunch
> 如果没有执行 `lunch`，很多 Rockchip 相关变量会是空的，U-Boot 和 Kernel 很容易编译失败。
>
> 推荐顺序：
>
> `source build/envsetup.sh`
>
> `lunch <product>-userdebug`

## 四、编译类型 variant

> [!info] 三种编译类型
> | variant | 适用场景 | 特点 |
> |---|---|---|
> | `user` | 正式量产版本 | 权限最严格，默认无 root，适合出货 |
> | `userdebug` | 日常开发调试 | 有调试能力，可以 `adb root`，最常用 |
> | `eng` | 工程开发版本 | 限制最少，调试工具最多，但不适合正式版本 |

> [!tip] 选择建议
> 日常开发用 `userdebug`。正式出货用 `user`。除非明确需要，否则少用 `eng`。

## 五、build.sh 的几个阶段

### U-Boot 阶段（`-U`）

这一阶段负责生成 bootloader 相关镜像。

大致过程是：进入 `u-boot` 目录，清理旧配置，根据 `UBOOT_DEFCONFIG` 推导 board 名，然后执行 `./make.sh <board>`。

> [!info] 常见产物
> | 产物 | 说明 |
> |---|---|
> | `u-boot/uboot.img` | U-Boot 主镜像 |
> | `u-boot/spl/u-boot-spl.bin` | SPL，早期启动阶段 |
> | `u-boot/tpl/u-boot-tpl.bin` | TPL，比 SPL 更早的启动阶段 |
> | `loader.bin` | Rockchip loader |
> | `trust.img` | 安全固件相关镜像 |

> [!note] loader / trust 来源
> `loader.bin`、`trust.img` 等通常来自 `rkbin` 的预编译文件，再由 Rockchip 脚本打包，不完全是源码直接编译出来的。

### Kernel 阶段（`-K` / `-C`）

这一阶段负责编译 Linux Kernel、DTS、外部驱动和 Rockchip 资源镜像。

主要做这些事：

1. 用 `rockchip_defconfig` 加产品配置片段生成最终 Kernel `.config`。
2. 根据 `PRODUCT_KERNEL_DTS` 编译对应 DTS。
3. 编译外部 Wi-Fi 驱动和 rvcam 驱动。
4. 打包 `resource.img`，里面可能包含充电图、开机 logo 等。

> [!info] 常见产物
> | 产物 | 位置 |
> |---|---|
> | Kernel 原始 Image | `kernel-6.1/arch/arm64/boot/Image` |
> | 拷贝后的 Kernel | `out/kernel` |
> | DTB 文件 | `kernel-6.1/arch/arm64/boot/dts/rockchip/*.dtb` |

### Android 阶段（`-A`）

这一阶段负责编译 Android 源码树。

它通常会先执行 `make installclean`，再执行 `make -jN`。`make installclean` 会清理上一次安装产物，但保留很多中间产物，所以比 `make clean` 快。

主要产物目录：`out/target/product/<product>/`

常见镜像：`boot.img`、`system.img`、`vendor.img`、`product.img`、`odm.img`、`super.img`、`vbmeta.img`。

> [!tip] OTA
> 如果 `build.sh` 带 `-o` 参数，还会生成 OTA 包。

### 镜像整理阶段（`mkimage.sh`）

Android 编出来的镜像还不是最终 Rockchip 烧录目录结构。`mkimage.sh` 会把 `out` 目录里的镜像整理到 `rockdev/Image-<product>/`。

> [!info] 打包脚本
> | 项目类型 | 使用脚本 |
> |---|---|
> | 非 AB | `./mkimage.sh` |
> | AB | `./mkimage_ab.sh` |

这一阶段还会生成或整理 `parameter.txt`，它是 Rockchip 烧录时使用的分区表。

### 整包阶段（`-u`）

这一阶段会调用 Rockchip 打包工具，把 `rockdev/Image-<product>/` 下面的多个分区镜像打包成一个 `update.img`。

`update.img` 是最终给 RKDevTool 等烧写工具使用的一键烧录固件。

## 六、常见分区镜像

> [!info] 分区镜像速查
> | 镜像 | 说明 |
> |---|---|
> | `boot.img` | 启动镜像，主要包含 kernel 和 ramdisk，启动 Linux 内核和第一阶段 rootfs |
> | `dtbo.img` | Device Tree Overlay，覆盖或补充主 DTS，适配不同板子硬件差异 |
> | `resource.img` | Rockchip 特有资源镜像，常放充电图、开机 logo，U-Boot 阶段使用 |
> | `uboot.img` / `idblock` | bootloader 相关镜像，负责设备早期启动 |
> | `trust.img` / `loader` | 安全固件相关镜像，可能包含 ATF、TOS 等 |
> | `super.img` | 动态分区容器，里面放 system、vendor、product、odm 等逻辑分区 |
> | `system.img` | Android 系统核心分区，包含 framework、系统 app、基础库等 |
> | `vendor.img` | 厂商分区，包含 HAL、驱动、芯片私有内容，和硬件强相关 |
> | `product.img` | 产品定制分区，常放预装 app、产品 overlay 等 |
> | `odm.img` | ODM 定制层，比 vendor 更偏下游客户或硬件定制 |
> | `system_ext.img` | 系统扩展分区，常用于 GMS 或可复用的系统组件 |
> | `userdata.img` | 用户数据分区，保存 app 数据和用户文件，可写 |
> | `cache.img` | 缓存分区，可能用于 OTA、recovery 日志等，可写 |
> | `vbmeta.img` | Verified Boot 元数据镜像，保存 AVB 校验信息 |
> | `customer.img` | Rockchip 给客户预留的客户分区，常放客户私有数据或配置，可写 |

> [!abstract] 为什么要拆这么多分区
> 1. 安全：只读分区可以被 AVB 或 dm-verity 校验。
> 2. 升级：OTA 可以只更新变化的分区。
> 3. 复用：同一套 system 可以搭配不同 vendor 或 product。
> 4. 维护：AOSP、芯片厂商、ODM、客户定制可以分层管理。

## 七、boot.img 和 super.img 的理解

### boot.img

`boot.img` 可以理解为启动 Linux 内核用的镜像。

- 非 AB 传统结构里，`boot.img` 通常包含 kernel 和 ramdisk。
- AB 且无独立 recovery 的结构里，recovery 可能合并进 boot，部分 ramdisk 可能放到 `init_boot.img`。
- Rockchip 平台上，device tree 可能放在 `resource.img` 或 `dtbo.img`，kernel 走 `boot.img`。

### super.img

`super.img` 可以理解为动态分区的大容器。

Android 10 以后引入动态分区。`super.img` 是一个大的物理分区，里面再划分出多个逻辑分区。

常见逻辑分区：`system_a / system_b`、`vendor_a / vendor_b`、`product_a / product_b`、`odm_a / odm_b`。

> [!tip] 动态分区的好处
> 分区大小不再完全写死在分区表里，后续 OTA 和产品维护会更灵活。

## 八、AB 与非 AB

> [!info] AB 与非 AB 对比
> | 项 | 非 AB | AB |
> |---|---|---|
> | 分区数量 | system、vendor 等各一份 | system_a / system_b、vendor_a / vendor_b 双槽位 |
> | 更新方式 | 通常需要进 recovery 刷 | 后台写入另一槽，重启后切换 |
> | 失败回滚 | 一般没有自动回滚 | 失败后可自动回滚到旧槽 |
> | recovery | 通常有独立 recovery 分区 | 通常合并进 boot |
> | 打包脚本 | `mkimage.sh` | `mkimage_ab.sh` |

> [!example] RK3576 实践
> 部分客户项目会启用 AB，例如 `BOARD_USES_AB_IMAGE := true`。

## 九、GKI 和 customer 分区

### GKI

GKI 是 Generic Kernel Image，也就是 Google 推的统一内核方案。

它的思路是：核心 kernel 保持统一、只读并签名，厂商驱动以 `.ko` 模块形式放到 `vendor_dlkm` 或 `system_dlkm`。

在 RK3576 SDK 中，常见默认值是 `BOARD_BUILD_GKI := false`。如果开启 GKI，通常还需要额外脚本把内核模块拷到对应分区位置。

### customer 分区

`customer.img` 是 Rockchip 给客户预留的独立分区，常用于存放客户私有数据或配置。

常见配置项：

- `TARGET_CUSTOMER_IMAGE ?= device/rockchip/rk3576/<cust>/customer/customer.img`
- `BOARD_WITH_CUSTOMER_PARTITIONS := customer:64M`

> [!tip] customer 分区特点
> 它通常是可写分区，并且不随普通 OTA 覆盖，适合保存客户希望长期保留的数据。

## 十、常用命令

> [!info] 常用编译命令
> | 目标 | 命令 |
> |---|---|
> | 加载环境 | `source build/envsetup.sh` |
> | 选择产品 | `lunch rk3576s_u-userdebug` |
> | 全量编译并打包 | `./build.sh -AUCKu` |
> | 只编 Android | `./build.sh -A` |
> | 出 OTA 包 | `./build.sh -A -o` |
> | 只重编 boot | `./build.sh` |
> | 归档镜像、patch 和 commit | `./build.sh -A -p` |

### build.sh 参数速查

> [!info] 参数速查
> | 参数 | 作用 |
> |---|---|
> | `-U` | 编译 U-Boot |
> | `-C` | 用 Clang 编译 Kernel |
> | `-K` | 编译 Kernel |
> | `-A` | 编译 Android |
> | `-B` | 编译 AB Image |
> | `-p` | 打包 IMAGE 和 patch stub |
> | `-o` | 生成 OTA 包 |
> | `-u` | 生成 `update.img` |
> | `-v` | 指定 `user` 或 `userdebug` |
> | `-V` | 指定版本号 |
> | `-d` | 指定 Kernel DTS |
> | `-J` | 指定并发数，默认 32 |
> | `-S` | spl-new |

## 十一、产物去哪里找

> [!info] 产物速查
> | 想要 | 路径 |
> |---|---|
> | 烧录整包 | `rockdev/Image-<product>/update.img` |
> | 单分区镜像 | `rockdev/Image-<product>/` |
> | Kernel Image | `out/kernel` 或 `kernel-6.1/arch/arm64/boot/Image` |
> | DTS / DTB | `kernel-6.1/arch/arm64/boot/dts/rockchip/*.dtb` |
> | U-Boot | `u-boot/uboot.img`、`u-boot/spl/u-boot-spl.bin` |
> | OTA 包 | `out/target/product/<product>/<product>-ota-*.zip` |
> | 编译命令记录 | `Image/<...>_<date>/build_cmd_info.txt`（由 `-p` 生成） |

## 十二、常见问题

> [!danger] 问题 1：没有 lunch
> 现象：`TARGET_PRODUCT` 变成 `aosp_arm`，Kernel / U-Boot 配置为空，编译报错。
>
> 处理：先执行 `source build/envsetup.sh`，再执行 `lunch <product>-userdebug`。

> [!warning] 问题 2：改了 build.sh 不生效
> 原因：`android/build.sh` 是符号链接，真正文件在 `device/rockchip/common/build/rockchip/build.sh`。

> [!warning] 问题 3：重复符号链接错误
> 现象：多个驱动里出现同名全局变量，例如 elan / himax 都有 `private_ts`。
>
> 处理：如果变量不需要导出，可以把其中一个改成 `static`。
>
> 相关记录：[[elan_ts.c private_ts 重复符号修复]]

> [!warning] 问题 4：U-Boot `.config not found`
> 常见原因是 `UBOOT_DEFCONFIG` 为空，也就是没有 `lunch`，或者没有给 `./make.sh` 传 board 参数。

> [!tip] 问题 5：分区大小不够
> 检查 `BoardConfig.mk` 里的 `BOARD_*_PARTITION_SIZE`，以及 Rockchip 使用的 `parameter.txt`。

> [!tip] 问题 6：OTA 不包含某个分区
> 检查 `BoardConfig.mk` 的 AB 开关，以及 `mkimage_ab.sh` 是否把对应分区打进去了。

## 十三、相关路径

> [!info] 相关路径速查
> | 内容 | 路径 |
> |---|---|
> | 客户配置目录 | `device/rockchip/rk3576/` |
> | build.sh 真正位置 | `device/rockchip/common/build/rockchip/build.sh` |
> | U-Boot make.sh | `u-boot/make.sh` |
> | Kernel 基础 defconfig | `kernel-6.1/arch/arm64/configs/rockchip_defconfig` |

## 最后记住

> [!quote] 一句话总结
> Android 编译不是一个单独的 `make`，而是一条流水线：选产品配置 → 编 bootloader → 编 kernel → 编 Android → 整理镜像 → 打包 `update.img`。
>
> 分区拆分是为了安全、升级、复用和分层维护。







实例：[[rk3576_u_AB_build_summary]]
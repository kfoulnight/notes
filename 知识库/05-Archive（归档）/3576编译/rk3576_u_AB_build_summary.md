---
title: rk3576_u AB 编译产物与分区镜像总结
tags:
  - android
  - rockchip
  - rk3576
  - AB
  - partition
  - build
aliases:
  - AB 编译总结
  - 分区镜像作用
created: 2026-07-31
product: rk3576_u
variant: userdebug
features: AB 开 / AVB 关 / GKI 关 / 动态分区开 / customer 开
related: "[[android_build_flow]]"
cssclasses:
  - archive-page
---

# rk3576_u AB 编译产物与分区镜像总结

> [!info] 本文背景
> 这份笔记基于一次实际编译：`lunch rk3576_u-userdebug` + `./build.sh -ABUCKuop`。
>
> 固件形态：AB 双槽、AVB 关闭、GKI 关闭、动态分区开启、customer 分区开启。
>
> 所有分区名都来自本次生成的 `parameter.txt`，不是泛泛而谈。通用编译流程见 [[android_build_flow]]。

> [!summary] 一句话看懂这份固件
> 这是一个 rk3576_u 的 AB 双槽固件：bootloader、boot、dtbo、vbmeta 等启动相关分区使用 `_a` / `_b` 双槽，system、vendor 等逻辑分区放在一个 `super.img` 容器内，userdata 和 customer 分区负责保存可写数据。

## 一、整体流程

1. 加载 Android 编译环境。
2. 执行 `lunch rk3576_u-userdebug`，选择产品和编译类型。
3. 通过 `get_build_var` 读取 DTS、Kernel config、U-Boot config 和 AB 配置。
4. 执行 `./build.sh -ABUCKuop`。
5. 编译 U-Boot，使用 `rk3576-ab-car` board 配置。
6. 编译 Kernel、DTS、Wi-Fi / rvcam 驱动和 `resource.img`。
7. 编译 Android，并生成 OTA 所需的 target-files。
8. 执行 `mkimage_ab.sh ota`，整理 `parameter.txt`、`super.img` 和各分区镜像。
9. 执行 `mkupdate.sh`，把所有镜像打包成 `update.img`。

> [!success] 产物落点
> | 路径 | 含义 |
> |---|---|
> | `out/target/product/rk3576_u/` | Android `make` 的原始产物 |
> | `rockdev/Image-rk3576_u/` | `mkimage_ab.sh` 整理后的镜像集合 |
> | `rockdev/Image-rk3576_u/update.img` | 整包烧录固件，大小约 2.0G |

## 二、build.sh 五个阶段

### U-Boot 阶段（`-U`）

典型过程：清理旧配置，根据 `UBOOT_DEFCONFIG` 推导 board 名，再执行 `./make.sh rk3576-ab-car`。

> [!info] U-Boot 配置链
> | 项目 | 本次配置 |
> |---|---|
> | `UBOOT_DEFCONFIG` | `rk3576_defconfig rk3576-ab-car.config` |
> | 最终 board | `rk3576-ab-car` |
> | 基础配置 | `CONFIG_BASE_DEFCONFIG="rk3576_defconfig"` |
> | 配置过程 | 基础 defconfig + `rk3576-ab-car.config` -> `.config` -> 编译 |

> [!info] U-Boot 产物
> | 产物 | 说明 |
> |---|---|
> | `uboot.img` | U-Boot 主镜像 |
> | `MiniLoaderAll.bin` | Rockchip loader |
> | `trust.img` | 安全固件，已并入 U-Boot |
> | `resource.img` | 充电图、开机 logo 等资源 |

### Kernel 阶段（`-C -K`）

本次 Kernel 编译会把多个配置片段合并，再根据指定 DTS 生成镜像。

> [!info] Kernel 配置和产物
> | 项目 | 本次内容 |
> |---|---|
> | defconfig | `rockchip_defconfig` + `pcie_wifi.config` + `rk3576_vehicle.config` |
> | DTS | `rk3576-vehicle-evb-v20` |
> | 外部 Wi-Fi 驱动 | `external/wifi_driver` |
> | 外部 rvcam 驱动 | `hardware/rockchip/rvcam/drivers` |
> | Kernel 原始产物 | `kernel-6.1/arch/arm64/boot/Image` |
> | 流程拷贝产物 | `out/kernel` |

### Android 阶段（`-A`，带 `-o` OTA）

本次 Android 阶段主要执行以下操作：

1. `make installclean`：清理安装产物。AB 切换时必须清理，避免旧产物干扰。
2. `make -jN`：编译 Android 源码树。
3. `make dist -jN`：生成 OTA 使用的 target-files。

### `mkimage_ab.sh ota` 阶段

这个阶段负责整理 AB 固件镜像。

主要动作：

1. 因为 `BOARD_USES_AB_IMAGE=true`，从 `$OUT/parameter.txt` 拷贝分区表到 Image 目录。
2. 拷贝 `super.img`。
3. 从 target-files 中拷贝 `boot.img`、`vbmeta.img`、`dtbo.img` 等镜像。
4. 生成与 OTA 匹配的量产 `super.img` 固件。

> [!danger] 之前遇到的坑
> 之前使用 `-B` 强制 AB，但产品配置实际是非 AB，导致 `mkimage_ab.sh` 因 `BOARD_USES_AB_IMAGE=false` 跳过 parameter 拷贝，最后报 `parameter.txt not found`。
>
> 现在 `BoardConfig.mk` 已设置 `BOARD_USES_AB_IMAGE := true`，命令参数和产品配置保持一致，问题消失。

### `mkupdate.sh` 整包阶段

打包过程可以理解为：

1. 把 `rockdev/Image-rk3576_u/` 下的文件拷贝到 Rockchip 打包工具的 Image 目录。
2. 执行 `mkupdate.sh rk3576 Image <flash_type>`。
3. 按 `parameter.txt` 的分区顺序拼接镜像。
4. 添加文件头、CRC / MD5 等信息。
5. 生成最终的 `update.img`。

## 三、实际分区表

本次分区表中包含这些分区：

`security`、`uboot_a`、`uboot_b`、`trust_a`、`trust_b`、`misc`、`dtbo_a`、`dtbo_b`、`vbmeta_a`、`vbmeta_b`、`boot_a`、`boot_b`、`backup`、`cache`、`metadata`、`frp`、`baseparameter`、`super`、`customer`、`userdata:grow`

> [!info] 分区表怎么看
> 带 `_a` / `_b` 的是 AB 双槽分区。`super` 是一个物理容器，内部再放 system、vendor、product、odm 等逻辑分区。`userdata:grow` 表示 userdata 会使用剩余空间增长。

## 四、分区镜像作用

### 启动链

下面按设备上电后的大致执行顺序排列。

> [!info] 启动链分区
> | 镜像 | 片上分区 | 内容 | 谁生成 | 作用 |
> |---|---|---|---|---|
> | `MiniLoaderAll.bin` | `loader` | bootloader 一阶段 | U-Boot 编译 | 最早执行，初始化 DDR，加载后续代码 |
> | `uboot.img` | `uboot_a` / `uboot_b` | U-Boot 主体 | U-Boot 编译 | 引导加载，读取 boot 并传递 ramdisk |
> | `trust.img` | `trust_a` / `trust_b` | ATF（BL31）+ TOS | rkbin 打包 | 安全固件，已并入 U-Boot |
> | `resource.img` | 并入 U-Boot | 充电图、开机 logo | U-Boot pack_resource | U-Boot 阶段显示 |
> | `baseparameter.img` | `baseparameter` | 基础参数 | 预置 | 保存屏参等基础显示参数 |

### Kernel 与设备树

> [!info] Kernel 相关分区
> | 镜像 | 分区 | 内容 | 作用 |
> |---|---|---|---|
> | `boot.img` | `boot_a` / `boot_b` | kernel + ramdisk + recovery | 启动 Kernel 和第一阶段 rootfs；AB 模式下 recovery 合并在这里 |
> | `dtbo.img` | `dtbo_a` / `dtbo_b` | Device Tree Overlay | 运行时叠加或修正主 DTS，适配不同板子 |

> [!note] recovery 说明
> 本次是 AB 固件，没有独立的 `recovery.img`，recovery 功能合并在 `boot.img` 中。

### 校验相关

> [!warning] AVB 当前是关闭的
> | 镜像 | 分区 | 内容 | 本次实际作用 |
> |---|---|---|---|
> | `vbmeta.img` | `vbmeta_a` / `vbmeta_b` | Verified Boot 元数据 | AVB 关闭，因此是约 4KB 空壳，不执行实际校验 |
> | — | `security` | Rockchip 安全信息 | 预留安全相关数据 |

### 动态分区：super 容器内部

> [!info] super 与逻辑分区
> | 镜像 | super 内逻辑分区 | 内容 | 作用 |
> |---|---|---|---|
> | `super.img` | `super` 物理容器 | 动态分区容器 | Android 10+ 使用，内部切分逻辑分区 |
> | `system.img` | `system` | AOSP 框架核心 | framework、系统 app、libc、运行时 |
> | `vendor.img` | `vendor` | 厂商 HAL、驱动、私有内容 | 和芯片硬件相关，例如 `himax_mmi.ko`、`mymem.ko` |
> | `product.img` | `product` | 预装 app、产品定制 | 从 system 中拆出 |
> | `odm.img` | `odm` | ODM 定制层 | 比 vendor 更下游 |
> | `system_ext.img` | `system_ext` | 跨产品共享的系统扩展 | 共享系统组件 |
> | `vendor_dlkm.img` | `vendor_dlkm` | 厂商 Kernel 模块 | 存放 `.ko` 模块 |
> | `system_dlkm.img` | `system_dlkm` | 系统 Kernel 模块 | 存放 `.ko` 模块 |
> | `odm_dlkm.img` | `odm_dlkm` | ODM Kernel 模块 | 存放 `.ko` 模块 |

> [!note] AB 与 super 的关系
> AB 模式下，system、vendor 等逻辑分区在 super 内也有双槽，例如 `system_a` / `system_b`。这些逻辑分区物理上共用一个 `super` 容器。

### 数据与杂项

> [!info] 可写分区和系统杂项
> | 镜像 | 分区 | 可写 | 作用 |
> |---|---|:---:|---|
> | `misc.img` | `misc` | 是 | Bootloader 通信区，保存 BCB、启动目标和 recovery 指令 |
> | `cache.img` | `cache` | 是 | OTA / recovery 缓存 |
> | — | `metadata` | 是 | 元数据，和加密相关 |
> | — | `frp` | — | Factory Reset Protection，谷歌防盗刷 |
> | `userdata.img` | `userdata:grow` | 是 | 用户数据，使用剩余空间增长 |
> | `customer.img` | `customer` | 是 | 客户私有分区，64M，通常不随 OTA 覆盖 |
> | — | `backup` | — | AB 备份和恢复 |

## 五、为什么要拆这么多分区

> [!abstract] 四个主要原因
> | 原因 | 说明 |
> |---|---|
> | 安全 | `vbmeta` 可以对只读分区做 AVB / dm-verity 校验；本次 AVB 关闭，因此不生效 |
> | 升级 | OTA 只更新变化的分区，AB 双槽可以后台写另一槽、重启切换、失败回滚 |
> | 复用 | 同一套 `system.img` 可以搭配不同 `vendor.img` / `product.img` 适配多种板型 |
> | 分层 | AOSP 放在 system，芯片厂放在 vendor，ODM 放在 odm，产品定制放在 product |

## 六、AB 双槽工作原理

AB 的核心是：当前系统运行在 A 槽时，OTA 在后台写入 B 槽，写完后重启切换到 B。这样升级失败时仍然可以回到 A。

1. 当前系统运行在槽 A。
2. OTA 在后台写入 `boot_b`、`system_b` 等 B 槽分区，不影响正在运行的 A 槽。
3. 系统写入 `misc` 分区的 BCB，标记下次启动 B 槽，并记录“尝试启动”。
4. 设备重启，Bootloader 尝试启动 B 槽。
5. 如果 B 槽启动成功，就把 B 标记为成功。
6. 如果连续启动失败，Bootloader 自动回滚到 A 槽。

> [!success] 分区规律
> `boot`、`uboot`、`trust`、`dtbo`、`vbmeta` 等启动相关分区都使用 `_a` / `_b` 成对结构。
>
> `super` 仍然是一个物理容器，但内部的 system、vendor、product、odm 等逻辑分区可以有 A / B 两套。

## 七、本次固件形态速查

> [!info] rk3576_u 固件配置
> | 特性 | 状态 | 体现 |
> |---|---|---|
> | AB 双槽 | 开启 | `parameter.txt` 里分区以 `_a` / `_b` 成对出现，没有独立 recovery |
> | AVB | 关闭 | `vbmeta.img` 是约 4KB 空壳，日志出现 `use default vbmeta.img` |
> | GKI | 关闭 | `BOARD_BUILD_GKI := false` |
> | 动态分区 | 开启 | 存在 `super.img`，内部包含 system、vendor、product、odm |
> | customer | 开启 | `customer` 分区大小为 64M，存在 `customer.img` |
> | 整包 | 已生成 | `rockdev/Image-rk3576_u/update.img`，大小约 2.0G |

## 八、相关路径

> [!info] 相关文件
> | 内容 | 路径 |
> |---|---|
> | 通用编译流程 | [[android_build_flow]] |
> | 客户配置目录 | `../Android/android/device/rockchip/rk3576/rk3576_u/` |
> | 分区表 | `../Android/android/rockdev/Image-rk3576_u/parameter.txt` |
> | 整包固件 | `../Android/android/rockdev/Image-rk3576_u/update.img` |

## 最后记住

> [!quote] 一句话总结
> `lunch` 选产品，`build.sh` 编 U-Boot / Kernel / Android，`mkimage_ab.sh` 整理 parameter 和 super，`mkupdate.sh` 打包 update.img。
>
> 分区按 `parameter.txt` 顺序拼成整包；boot 包含 Kernel 和 ramdisk；super 内含 system、vendor、product、odm 等逻辑分区；userdata 和 customer 可写；AB 双槽靠 `_a` / `_b` 成对分区实现后台升级与失败回滚。

---
title: 今日构建问题复盘：根因分析与长期排障方法论
tags:
  - android
  - rockchip
  - rk3576
  - debugging
  - build
  - selinux
  - partition
aliases:
  - Android 构建问题方法论
  - rk3576 构建排障复盘
created: 2026-07-31
product: rk3576_u
related:
  - "[[rk3576_u_AB_build_summary]]"
  - "[[android_build_flow]]"
cssclasses:
  - archive-page
---

# 今日构建问题复盘：根因分析与长期排障方法论

> [!info] 本文范围
> 本文复盘今天实际遇到的构建、链接、镜像打包和配置问题，并将它们抽象成一套可以迁移到其它 Android / Rockchip 项目的排障方法。
>
> 本次产品：`rk3576_u`
>
> 最终固件形态：AB 开启 / AVB 关闭 / GKI 关闭 / 动态分区开启 / customer 分区开启。

> [!summary] 核心主线
> 今天的问题来自不同层级：源码链接、构建环境、脚本接口、模式一致性和归档工具链。排障的方法论是：先分层定位，再看配置快照，最后验证产物是否匹配。

## 一、今天遇到的问题总览

> [!info] 问题总览
> | 编号 | 现象 | 所属层级 | 最终原因 | 状态 |
> |:---:|:---|:---|:---|:---:|
> | 1 | `duplicate symbol: private_ts` | Kernel 链接 | elan 与 Himax 驱动定义了同名全局符号 | 已修复 |
> | 2 | U-Boot `ERROR: No .config` | U-Boot 配置 | 没有正确选择产品，且 `make.sh` 没拿到 board 参数 | 已修复 |
> | 3 | `No found parameter` | 镜像打包 | 用 `-B` 强行走 AB，但产品配置仍是非 AB | 已修复 |
> | 4 | `.repo/repo/repo: No such file` | 归档打包 | `-p` 依赖 repo 工具，当前 SDK 路径缺少 repo 可执行文件 | 不影响 update.img |
> | 5 | `trust.img not found` | U-Boot/镜像整理 | U-Boot trust 产物未在预期路径生成或未被拷贝 | 需要单独确认 |
> | 6 | AVB/AB 状态混淆 | 产品配置 | BoardConfig、命令行参数、打包脚本存在多个状态来源 | 已确认 |

## 二、问题一：Kernel 链接阶段 duplicate symbol

### 现象

```text
ld.lld: error: duplicate symbol: private_ts
>>> defined at elan_ts.c:45
>>> defined at himax_common.c:102
```

这发生在 `LD vmlinux.o` 阶段，不是编译某个 `.c` 文件时失败，而是所有 `.o` 已经生成，在最终链接 `vmlinux` 时失败。

### 根因定位

两个驱动分别定义了同名的外部链接符号：

```c
// elan_ts.c
struct elan_ts_data *private_ts;

// himax_common.c
struct himax_ts_data *private_ts;
EXPORT_SYMBOL(private_ts);
```

虽然两个变量的类型不同，但 C 链接器只按符号名区分，不按 C 类型区分。因此两个目标文件同时参与内核链接时，产生重复符号。

同时，内核配置确实把两个驱动都打开了：

```text
CONFIG_TOUCHSCREEN_ELAN5515=y
CONFIG_TOUCHSCREEN_HIMAX_COMMON=y
```

### 处理方式

将只在 elan 源文件内部使用的变量改为文件私有：

```c
static struct elan_ts_data *private_ts;
```

修改文件：`kernel-6.1/drivers/input/touchscreen/elan/elan_ts.c`

### 为什么这个修复合理

`elan_ts.c` 中所有引用都发生在同一个源文件内，且没有 `EXPORT_SYMBOL(private_ts)`。因此加 `static`：

- 不改变 elan 驱动的运行逻辑；
- 不影响其它源文件访问；
- 将符号从全局命名空间移到文件作用域；
- 避免与 Himax 的导出符号冲突。

### 长期规避方案

**源码规范**

驱动私有状态默认写成：

```c
static struct xxx_ts_data *g_ts;
```

只有确实需要跨文件访问时，才放到头文件并使用唯一前缀：

```c
struct elan_ts_data *elan_private_ts;
EXPORT_SYMBOL_GPL(elan_private_ts);
```

**配置规范**

同一块板上通常只应启用实际使用的触摸芯片。可以在 board 的 kernel config fragment 中明确关闭不使用的驱动：

```text
# CONFIG_TOUCHSCREEN_ELAN5515 is not set
```

**排障命令**

```bash
grep -Rnw kernel-6.1/drivers -e 'private_ts'

# 查看最终内核配置
grep -E 'TOUCHSCREEN_(ELAN|HIMAX)' kernel-6.1/.config

# 查看目标文件是否包含同名符号
nm -A kernel-6.1/drivers/input/touchscreen/*/*.o | grep private_ts
```

> [!tip] 经验
> `duplicate symbol` 首先查：谁定义了这个符号、哪些配置把它编进来了、它是否应该是 `static`。不要一开始就盲目清理整个 out 目录。

## 三、问题二：U-Boot ERROR: No .config

### 现象

```text
ERROR: No .config

Usage:
    ./make.sh [board|sub-command]
```

前面同时出现：

```text
TARGET_PRODUCT=aosp_arm
TARGET_BUILD_VARIANT=eng
start build uboot:
```

### 根因一：没有正确选择 Rockchip 产品

没有先执行正确的 `lunch` 时，Android 构建环境退回到默认产品 `TARGET_PRODUCT=aosp_arm`。

此时：

- `PRODUCT_UBOOT_CONFIG` 为空；
- `PRODUCT_KERNEL_DTS` 为空；
- `KERNEL_VERSION` 为空；
- build.sh 无法推导 U-Boot board；
- 最终执行类似无参数的 `./make.sh`。

正确初始化：

```bash
source build/envsetup.sh
lunch rk3576_u-userdebug
```

### 根因二：原 build.sh 没有把 board 传给 make.sh

原逻辑类似：

```bash
make $UBOOT_DEFCONFIG
./make.sh $NEW_SPL
```

当 `$NEW_SPL` 为空时，`./make.sh` 没有 board 参数。U-Boot 的 `make.sh` 设计为 `./make.sh <board>`，由它自己完成 defconfig、生成 `.config` 和后续编译。

### 处理方式

在 build.sh 中从 `UBOOT_DEFCONFIG` 推导 board，并传给 `make.sh`：

```bash
UBOOT_BOARD=$(echo $UBOOT_DEFCONFIG | awk '{print $NF}')
case "$UBOOT_BOARD" in
    *.config)
        UBOOT_BOARD="${UBOOT_BOARD%.config}"
        ;;
    *_defconfig)
        UBOOT_BOARD="${UBOOT_BOARD%_defconfig}"
        ;;
esac

./make.sh $UBOOT_BOARD $NEW_SPL
```

对于 AB 产品，`UBOOT_DEFCONFIG = rk3576_defconfig rk3576-ab-car.config`，最终 board 为 `rk3576-ab-car`。`make.sh` 再根据 `CONFIG_BASE_DEFCONFIG` 链式应用基础 defconfig。

### 长期规避方案

编译脚本开头增加环境校验，比等到 U-Boot 深处才失败更好：

```bash
if [ -z "$TARGET_PRODUCT" ] || [ "$TARGET_PRODUCT" = "aosp_arm" ]; then
    echo "ERROR: Please lunch a Rockchip product first"
    echo "Example: lunch rk3576_u-userdebug"
    exit 1
fi

if [ -z "$UBOOT_DEFCONFIG" ]; then
    echo "ERROR: PRODUCT_UBOOT_CONFIG is empty"
    exit 1
fi
```

并在真正编译前打印配置快照：

```bash
echo "TARGET_PRODUCT=$TARGET_PRODUCT"
echo "TARGET_BUILD_VARIANT=$BUILD_VARIANT"
echo "UBOOT_DEFCONFIG=$UBOOT_DEFCONFIG"
echo "KERNEL_VERSION=$KERNEL_VERSION"
echo "KERNEL_DTS=$KERNEL_DTS"
echo "BOARD_USES_AB_IMAGE=$BUILD_AB_IMAGE"
```

> [!tip] 方法论
> 遇到构建失败，先看最早的配置输出。`TARGET_PRODUCT=aosp_arm`、变量为空等信号，通常说明还没进入真正的源码问题。

## 四、问题三：parameter.txt 不存在

### 现象

```text
Error:No found parameter!
cat: Image/parameter.txt: No such file or directory

Error:<ParseParamFile> open file failed,
file=./Image/parameter.txt!
```

### 当时的命令

`./build.sh -ABUCKuop`，其中 `-B` 表示强制 build.sh 按 AB Image 处理。

### 根因：两个 AB 状态不一致

当时的产品 `rk3576_u` 没有设置 `BOARD_USES_AB_IMAGE := true`，于是出现两个不同的状态：

> [!warning] 状态不一致
> | 位置 | 状态 |
> |:---|:---:|
> | build.sh 的 `BUILD_AB_IMAGE` | `-B` 强制为 true |
> | mkimage_ab.sh 重新读取的 `BOARD_USES_AB_IMAGE` | false |

build.sh 因为 `-B` 进入 AB 分支，调用 `./mkimage_ab.sh ota`。但 mkimage_ab.sh 内部又判断：

```bash
if [ "$BOARD_USES_AB_IMAGE" = "true" ]; then
    cp $OUT/parameter.txt $IMAGE_PATH/parameter.txt
fi
```

由于产品配置仍为 false，这个复制逻辑没有执行。最终 `mkupdate.sh` 找不到 `parameter.txt`。

### 正确处理

在客户的 BoardConfig.mk 中明确设置 `BOARD_USES_AB_IMAGE := true`。

文件：`device/rockchip/rk3576/rk3576_u/BoardConfig.mk`

这样：

- build.sh 读取到 true；
- mkimage_ab.sh 读取到 true；
- Android 生成 AB 分区表；
- `parameter.txt` 被拷贝到 `rockdev/Image-rk3576_u/`；
- mkupdate.sh 正常生成 update.img。

### 最终验证方法

不要只看编译命令，要检查实际产物：

`get_build_var BOARD_USES_AB_IMAGE`

`grep 'CMDLINE:' rockdev/Image-rk3576_u/parameter.txt`

应看到以下成对分区：`uboot_a/uboot_b`、`boot_a/boot_b`、`dtbo_a/dtbo_b`、`vbmeta_a/vbmeta_b`。

AB 版本通常没有独立 recovery：

`ls rockdev/Image-rk3576_u/recovery.img`

### 长期规避方案

不要让命令行参数和产品配置各自维护一套状态。推荐：

1. `BOARD_USES_AB_IMAGE` 作为唯一事实来源；
2. build.sh 只读取它，不使用 `-B` 覆盖产品状态；或
3. 如果保留 `-B`，必须在脚本启动时把最终状态导出给子脚本；
4. 打包前强校验：

```bash
if [ "$BUILD_AB_IMAGE" = "true" ] && [ "$BOARD_USES_AB_IMAGE" != "true" ]; then
    echo "ERROR: AB mode mismatch"
    exit 1
fi
```

> [!warning] 重要
> AB 不是单纯的打包选项。它会影响 Android 分区表、U-Boot 配置、OTA 逻辑、boot/recovery 关系和 super 内部布局。切换 AB 后必须重新生成 Android、U-Boot、parameter 和 super，不能只拿旧镜像重新打包。

## 五、问题四：repo 缺失导致 -p 归档失败

### 现象

```text
./build.sh: line 371: .repo/repo/repo: No such file or directory
cp: cannot stat 'out/commit_id.xml': No such file or directory
```

### 根因

`-p` 不只是复制镜像，它还会：

```bash
.repo/repo/repo forall -c ".../gen_patches_body.sh"
.repo/repo/repo manifest -r -o out/commit_id.xml
```

这要求 SDK 中存在 `.repo/repo/repo`，当前环境没有这个路径或文件。

### 影响范围

这个错误发生在 `update.img` 已经生成之后，因此：

- Android 编译成功；
- 镜像整理成功；
- update.img 已成功生成；
- 失败的是 patch/manifest 归档，不是烧录固件本身。

### 处理方式

如果不需要归档，不要带 `-p`：

`./build.sh -ABUCKuo`

如果需要归档，先确认 repo 工具存在：

`which repo`

`ls -l .repo/repo/repo`

如果 SDK 的 repo 工具路径确实缺失，应按团队规定安装或恢复 repo，避免直接随意改动 `.repo`。

### 长期规避方案

在 `-p` 分支开始前预检查：

```bash
if [ "$BUILD_PACKING" = true ] && [ ! -x .repo/repo/repo ]; then
    echo "ERROR: -p requires .repo/repo/repo"
    exit 1
fi
```

同时建议让归档阶段失败时明确标注：

```text
Build image: SUCCESS
Generate update.img: SUCCESS
Generate patch archive: FAILED
```

而不是让用户误以为整个构建失败。

## 六、问题五：trust.img not found 警告

### 现象

```text
u-boot/trust.img not fount! Please make it from u-boot first!
```

### 可能原因

这个问题与 `parameter.txt` 不同，不能仅凭一行日志确定唯一原因。常见可能性有：

1. U-Boot 的 trust 已合并到 `uboot.img`；
2. `trust.img` 实际由 `rkbin` 或其它脚本生成在不同目录；
3. U-Boot 没有完整编译；
4. `mkimage` 仍然尝试按旧路径寻找独立 `trust.img`。

本项目配置中存在 `BOARD_ROCKCHIP_TRUST_MERGE_TO_UBOOT := true`，因此不能只因为独立文件不存在，就断言固件不可用。

### 建议验证

```bash
ls -lh u-boot/uboot.img
ls -lh u-boot/trust.img
ls -lh rockdev/Image-rk3576_u/uboot.img

# 查看 U-Boot 配置
grep -E 'TRUST|FIT|ARM64' u-boot/.config

# 查看最终 update.img 是否已经包含 uboot
ls -lh rockdev/Image-rk3576_u/update.img
```

如果量产工具明确要求独立 trust 分区，应进一步检查：

- `BOARD_ROCKCHIP_TRUST_MERGE_TO_UBOOT`；
- U-Boot 打包脚本；
- `parameter.txt` 是否存在 `trust_a/trust_b`；
- `package-file` 是否包含 `trust.img`。

> [!note] 本次结果
> 这次最终 `update.img` 已生成并包含 `uboot.img`。但如果要做量产验证，仍建议确认目标板的启动链是否要求独立 trust 镜像。

## 七、AVB 与 AB 的配置混淆

### 当前最终状态

> [!info] 固件配置矩阵
> | 功能 | 状态 | 证据 |
> |:---|:---:|:---|
> | AB 双槽 | 开 | `BOARD_USES_AB_IMAGE := true`；parameter 中有 `_a/_b` |
> | AVB | 关 | 日志 `BOARD_AVB_ENABLE=false`；vbmeta 为默认 4KB 镜像 |
> | GKI | 关 | `BOARD_BUILD_GKI := false` |
> | 动态分区 | 开 | 有 `super.img` |
> | customer 分区 | 开 | parameter 中有 `customer` |

### 概念区分

**AB**

解决的是升级可靠性：A 槽运行时 OTA 写 B 槽，重启切换到 B，B 启动失败则回滚 A。

**AVB**

解决的是启动完整性与签名校验：bootloader 校验 vbmeta，vbmeta 描述 boot/system/vendor 等镜像的哈希或签名，锁定设备通常要求签名链可信。

AB 和 AVB 是两个独立维度，可以出现所有组合：AB 开 + AVB 关、AB 关 + AVB 开、AB 开 + AVB 开、AB 关 + AVB 关。本次属于第一种。

### 长期规避方案

每次编译前输出配置矩阵：

```text
PRODUCT=rk3576_u
VARIANT=userdebug
AB=true
AVB=false
GKI=false
DYNAMIC_PARTITIONS=true
DTS=rk3576-vehicle-evb-v20
```

编译后再根据实际产物验证，不要只相信命令行参数。

## 八、底层构建问题通用定位框架

### 先判断失败在哪一层

Android/Rockchip 构建链可以分成：产品配置 → 构建环境/lunch → U-Boot 配置与编译 → Kernel 配置、编译与链接 → Android make/Soong/Ninja → 分区镜像生成 → Rockchip 镜像整理 → update.img/OTA/归档。

> [!info] 不同层的排查入口
> | 层 | 典型关键词 | 第一检查点 |
> |:---|:---|:---|
> | 环境 | `aosp_arm`、变量为空 | `source`、`lunch`、`get_build_var` |
> | U-Boot | `.config not found` | board 参数、defconfig、`u-boot/.config` |
> | Kernel 编译 | `error:`、头文件缺失 | Kconfig、Makefile、依赖 |
> | Kernel 链接 | `duplicate symbol`、`undefined symbol` | `nm`、符号定义、配置是否同时开启 |
> | Android 编译 | Ninja/Soong error | 第一处 error，不看最后一行 |
> | 镜像生成 | `mkfs`、`sparse`、空间不足 | out 镜像大小、分区 size、文件系统 |
> | 打包 | `parameter`、`package-file` | parameter、package-file、Image 目录 |
> | 烧录 | layout/校验/启动失败 | parameter 与镜像槽位是否匹配 |
> | 启动/权限 | mount、avc denied | fstab、SELinux context、sepolicy |
> | 归档 | `repo`、manifest、patch | 工具链和 Git 仓库状态 |

### 永远先看第一处真正错误

构建日志末尾通常只是连锁失败，往往不是根因。应该从上往下找第一处：`error:`、`fatal:`、`No such file`、`undefined symbol`、`duplicate symbol`、`cannot open`、`permission denied`。

### 看配置快照，而不是猜

```bash
source build/envsetup.sh
lunch rk3576_u-userdebug

get_build_var TARGET_PRODUCT
get_build_var TARGET_BUILD_VARIANT
get_build_var BOARD_USES_AB_IMAGE
get_build_var BOARD_AVB_ENABLE
get_build_var BOARD_BUILD_GKI
get_build_var PRODUCT_KERNEL_DTS
get_build_var PRODUCT_KERNEL_CONFIG
get_build_var PRODUCT_UBOOT_CONFIG
get_build_var TARGET_DEVICE_DIR
```

### 比较三份配置

排查板卡配置时，要同时看：客户 BoardConfig.mk → include → 芯片公共 BoardConfig.mk → include → 最终 `get_build_var` 结果。

不能只看某个 `.mk` 文件就认为它一定生效，因为 `?=` 不会覆盖已有值、include 顺序会影响变量、命令行参数可能覆盖脚本变量、子脚本可能重新读取环境得到与父脚本不同的状态。

## 九、分区挂载问题的方法论

虽然今天没有出现实际的 mount 失败，但以后遇到 `mount failed`、`unknown filesystem`、`failed to mount /vendor`、`Unable to find logical partition` 时，建议按以下顺序定位。

### 先确认物理分区还是逻辑分区

`boot`、`dtbo`、`vbmeta`、`userdata`、`customer` 通常是物理分区。`system`、`vendor`、`product`、`odm` 可能在 super 内的逻辑分区。

### 检查分区表

`grep 'CMDLINE:' rockdev/Image-rk3576_u/parameter.txt`

确认分区是否存在、AB 是否有 `_a/_b`、分区大小是否足够、super 是否存在、userdata 是否 `grow`。

### 检查 fstab 与启动槽位

重点对比 `vendor/etc/fstab.*`、`vendor/etc/init/hw/init*.rc`、`system/core/rootdir/etc/fstab.*`。检查 fstab 使用的是正确的设备节点、逻辑分区名称、`slotselect`、`logical`、`avb`、`wait` 等 flag。

### AB 常见错误

- `boot_a` 与 `boot_b` 不匹配
- fstab 没有 slotselect
- vbmeta 槽位不一致
- super 的逻辑分区名称与 fstab 不一致

长期方案是让 `BoardConfig → parameter.txt → super metadata → fstab → bootloader` 使用同一套 AB 定义，不要手工混用非 AB 镜像。

## 十、SELinux 权限问题的方法论

今天没有实际遇到 `avc: denied`，但客户定制中已有 `device/rockchip/rk3576/rk3576_u/sepolicy/genfs_contexts`。以后如果遇到权限问题，按“现象 → 主体 → 对象 → 操作”分析。

### 收集 AVC 日志

```bash
adb logcat -b all | grep -i 'avc: denied'
adb shell dmesg | grep -i 'avc: denied'
```

典型日志：

```text
avc: denied { read } for pid=1234
scontext=u:r:my_service:s0
 tcontext=u:object_r:vendor_file:s0
 tclass=file
```

### 四元组

> [!info] AVC 四元组
> | 字段 | 含义 |
> |:---|:---|
> | `scontext` | 谁在访问，即主体 domain |
> | `tcontext` | 访问什么对象，即目标 type |
> | `{ read }` | 需要什么权限 |
> | `tclass` | 对象类型，如 file、dir、socket 等 |

### 定位文件标签

```bash
adb shell ls -Z /vendor/bin/my_service
adb shell ls -Z /vendor/etc/my_config
```

比较文件实际路径、`file_contexts` 标签、`genfs_contexts` 对 sysfs/procfs 的标签、服务 domain 是否正确。

### 正确修复顺序

1. 确认服务是否应该访问该资源；
2. 确认文件/节点标签是否错误；
3. 优先修正 `file_contexts` / `genfs_contexts`；
4. 再添加最小化 allow 规则；
5. 不要直接使用宽泛 `permissive` 或 `allow * * *`。

### 长期规避

- userdebug 允许调试，但量产 user 必须在 enforcing 下验证；
- 每个新增服务定义独立 domain；
- 每条 allow 规则都应有对应的业务原因；
- 用 `audit2allow` 只做候选参考，不能直接无审查提交。

## 十一、分区镜像打包异常的方法论

### 先确认输入是否齐全

`ls -lh rockdev/Image-rk3576_u/`

重点检查：`parameter.txt`、`package-file`、`MiniLoaderAll.bin`、`uboot.img`、`boot.img`、`super.img`、`vbmeta.img`、`dtbo.img`。

### 再确认模式是否一致

> [!warning] 模式一致性检查
> | 项目 | 必须一致 |
> |:---|:---|
> | 产品 AB 配置 | `BOARD_USES_AB_IMAGE` |
> | build.sh 模式 | `BUILD_AB_IMAGE` |
> | mkimage 脚本 | `mkimage.sh` / `mkimage_ab.sh` |
> | parameter | 单槽 / 双槽 |
> | U-Boot | car / ab-car |
> | OTA 包 | 非 AB / AB |

### 最后检查 package-file

`package-file` 决定最终打包哪些文件。如果镜像在 Image 目录存在，但 package-file 没引用，仍可能不会进入 update.img。

`cat RKTools/linux/Linux_Pack_Firmware/rockdev/package-file`

### 产物级验收

```bash
# update.img 存在且时间是本次构建
stat rockdev/Image-rk3576_u/update.img

# parameter 与产品匹配
grep 'CMDLINE:' rockdev/Image-rk3576_u/parameter.txt

# 检查关键镜像
for f in parameter.txt MiniLoaderAll.bin uboot.img boot.img super.img vbmeta.img dtbo.img; do
    test -s "rockdev/Image-rk3576_u/$f" && echo "OK $f" || echo "BAD $f"
done
```

> [!tip] 经验
> "Android build completed successfully" 只说明 Android 产物完成；"Make firmware OK" 说明固件打包完成；`update.img` 生成成功后，还要做分区表和关键文件验收。

## 十二、版本兼容冲突的方法论

常见冲突不只是一种版本号：Android 版本、Kernel 版本、U-Boot 版本、DTS/DTBO、GKI/非 GKI、AB/非 AB、AVB key/vbmeta、工具链/JDK/Clang。

### 建立兼容矩阵

```text
Android: 14 / API 34
Kernel: 6.1
SoC: RK3576
Arch: arm64
DTS: rk3576-vehicle-evb-v20
AB: true
AVB: false
GKI: false
JDK: 8
Clang: clang-r487747c
```

### 重点检查边界

> [!info] 版本边界与典型问题
> | 边界 | 典型问题 |
> |:---|:---|
> | Android ↔ Kernel | boot header、模块 ABI、DTS 不匹配 |
> | Kernel ↔ DTS | 驱动已开但节点不存在，或节点存在驱动没开 |
> | U-Boot ↔ parameter | 槽位、分区名、启动地址不一致 |
> | AB ↔ OTA | OTA 写错槽位或没有 slot metadata |
> | AVB ↔ bootloader | 签名 key 不受信、vbmeta 不匹配 |
> | GKI ↔ vendor modules | 模块放错分区或 ABI 不匹配 |
> | JDK/Clang ↔ Android | 编译器报错、Soong/Ninja 行为异常 |

### 长期方案：保存构建元数据

每次可交付构建都保存：product/variant、repo manifest、git commit id、BoardConfig 最终变量、kernel `.config`、U-Boot `.config`、parameter.txt、package-file、编译命令、工具链版本。

你当前脚本的 `-p` 正是尝试保存这些信息，但它依赖 `.repo/repo/repo`。应把归档工具链也纳入构建环境检查。

## 十三、推荐的标准排障流程

### Step 1：确认环境

```bash
pwd
source build/envsetup.sh
lunch rk3576_u-userdebug
```

### Step 2：打印最终配置

```bash
get_build_var TARGET_PRODUCT
get_build_var TARGET_BUILD_VARIANT
get_build_var BOARD_USES_AB_IMAGE
get_build_var BOARD_AVB_ENABLE
get_build_var BOARD_BUILD_GKI
get_build_var PRODUCT_KERNEL_DTS
get_build_var PRODUCT_UBOOT_CONFIG
```

### Step 3：确认构建模式

AB、AVB、GKI、动态分区、DTS、U-Boot board 是否与预期一致。

### Step 4：按层执行

`U-Boot → Kernel → Android → mkimage → mkupdate`

不要在 Android 还没成功时直接分析打包；也不要在 parameter 不正确时分析设备启动。

### Step 5：抓第一处错误

将日志保存：

```bash
./build.sh ... 2>&1 | tee build-$(date +%Y%m%d-%H%M).log
```

### Step 6：做产物验收

文件是否存在、大小是否合理、时间是否为本次构建、parameter 是否匹配、AB 槽位是否正确、package-file 是否引用、update.img 是否成功生成。

### Step 7：再上板验证

上板后按顺序检查：U-Boot 日志 → kernel 启动 → 分区挂载 → SELinux AVC → framework 启动 → OTA/回滚。

## 十四、最终结论

今天的问题不是一个单独的“编译失败”，而是发生在不同层级的六类问题：

1. 源码链接问题：两个驱动把不同类型的变量放进了同一个全局符号命名空间；
2. 构建环境问题：没有 lunch，导致客户变量全部为空；
3. 脚本接口问题：U-Boot `make.sh` 需要 board 参数，但上层脚本没有正确传递；
4. 构建模式一致性问题：`-B` 强制 AB，但产品 BoardConfig 仍是非 AB；
5. 归档工具链问题：`-p` 依赖 repo，但环境缺少 repo；
6. 配置认知问题：AB、AVB、GKI 是独立能力，不能通过同一个“安全/量产”概念混为一谈。

## 最后记住

> [!quote] 排障口诀
> 先分层，后定位；先看第一处错误，再看配置快照；先验证输入，再验证产物；让产品配置、构建脚本、打包脚本和最终分区表保持同一个事实来源。

### 本次问题的长期改进清单

- [ ] build.sh 启动时检查 `lunch` 和关键变量是否为空
- [ ] 统一 AB 状态来源，避免 `-B` 与 `BOARD_USES_AB_IMAGE` 分离
- [ ] U-Boot 编译前明确打印并校验 board 参数
- [ ] Kernel 驱动私有全局变量默认使用 `static` 和厂商前缀
- [ ] `-p` 开始前检查 `.repo/repo/repo`
- [ ] 编译结束后自动检查 parameter、package-file、关键镜像
- [ ] 保存 Android / Kernel / U-Boot 最终配置和版本信息
- [ ] 在 user 版本上验证 SELinux enforcing、AVB 和 OTA 回滚
- [ ] 将 AB/AVB/GKI/DTS/工具链写入构建 manifest

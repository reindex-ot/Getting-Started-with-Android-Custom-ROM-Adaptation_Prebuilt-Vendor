# Android カスタム ROM アダプテーションの開始 - ベンダーのプレビルド
Android カスタム ROM アダプテーションの開始 - Vendor の事前ビルド

- [快速上手 Android Custom ROM 适配 - Prebuilt Vendor](https://blog.lynnrin.moe/posts/ROM-bringup-guide-prebuilt/#%E5%AE%8C%E5%96%84-device-tree-%E5%87%86%E5%A4%87%E5%BC%80%E5%A7%8B%E7%BC%96%E8%AF%91-android)

[<img src=""
     alt=""
     height="80">]()

---    
* 快速上手 Android Custom ROM 适配 - Prebuilt Vendor
    * 什么是 Prebuilt Vendor, 为什么要用 Prebuilt Vendor
    * 需要准备的东西
    * 准备开始
        * 安装所需依赖
        * 同步 LineageOS 源码
    * 开始适配
        * 解包 MIUI
        * 提取所需文件
            * 提取 vendor 以及 odm
            * 提取 kernel
        * 开始 Bring up device tree
            * 初始化 device tree 并编译 recovery
                * Android.mk
                * Android.bp
                * AndroidProducts.mk
                * lineage_thyme.mk
                * BoardConfig.mk
                * device.mk
                * 导入 recovery 所需基本文件
                * 开始编译 recovery
            * 完善 device tree 准备开始编译 Android
                * proprietary-files.txt, extract-files.sh 和 setup-makefiles.sh
                    * proprietary-files.txt
                    * extract-files.sh
                    * setup-makefiles.sh
                * FCM 或 manifest.xml 或 compatibility_matrix.device.xml
                * bootctrl 以及 gpt-utils 或 mtk_plpath_utils
                * VoLTE
                * Overlay
                * SELinux
            * 硬件功能的修复
                * 屏下指纹
                    * Android 12 以下系统
                * Android 12 及以上系统
                * 蓝牙音频
                * Android 12 及以下系统
                * Android 13
            * 声音
        * debug 指南
            * 设备卡在 oem logo (卡一屏)
                * data 分区未正确格式化
                * Sepolicy rules 的错误配置
                * 未正确配置 vendor image
                * 未正确配置 ramdisk
                * 未正确配置 kernel 或 prebuilt kernel 不可用
                * BPF 加载失败
                * 获取 logcat 或 pstore 日志
                    * 获取 logcat 日志
                    * 获取 pstore 日志
            * 设备卡在开机动画 (卡二屏)
            * logcat 日志中出现 linker 错误
            * logcat 日志中出现 Permission denied 错误
            * logcat 日志中出现 fatal signal 11 (SIGSEGV), code 1 (SEGV_MAPERR) 错误
---

# Android カスタム ROM ビルドの適用を始める - ベンダーのプレビルド

    ⚠️ 注：私はプロのAndroid開発者ではありません、この記事は参考用です。もし間違いがあれば、遠慮なく訂正してください！
    ⚠️ 注：私はプロのAndroid開発者ではありません、この記事は参考用です。もし間違いがあれば、遠慮なく訂正してください！
    ⚠️ 注：私はプロのAndroid開発者ではありません、この記事は参考用です。もし間違いがあれば、遠慮なく訂正してください！

この投稿はGKI、VNDKバージョン30と互換性のないVABデバイスであるXiaomi 10Sの適用についてです。

コンパイルサーバーシステム: Ubuntu 22.04

# プレビルドベンダーとは何か、何故ベンダーのプレビルドなのか?

プレビルドベンダーとは、その名の通りでベンダーにコンパイル済みのベンダーを使用してカスタムの適用を行う事です。 

これにより、適用の難易度とデバッグの時間が大幅に削減されます。

GoogleはAndroid 8.0でProject Trebleを公開し、AndroidカスタムROMの適用とデバッグの難易度を大幅に下げました。

今年にはGoogleはGoogle Requirements Freeze、またの名を「Vendor Freeze」を導入し、適用の難易度を更に下げました。

Android 8.0以前では、カスタムROMを適用するために、チップやハードウェアメーカーのAOSP、CAFや他のオープンソースコードを通して、デバイスハードウェアが必要とするランタイムライブラリ、HALやドライバをコンパイルする必要があり、Androidがメジャーバージョンにアップグレードされるたびに再びコンパイルする必要がありました。新しいバージョンのAndoridは、動作やコンパイルに問題が起きる可能性が非常に高く、ソースコードの更新がされるまで待つか、自分でソースを修正する必要があります。 

Project TrebleとGoogle Requirements Freezeの導入するとベンダーを純正ROMで直接使用することができるようになり、カーネルとシステムをコンパイルする必要があるだけでなく、Google Requirements Freezeの導入により、ベンダーは少なくとも3つの主要バージョンのAndroidアップデートと互換性があります。 (例:Xiaomi 12はAndroid 12、VNDKバージョン32でこのバージョンのベンダーイメージは少なくともAndroid 15、VNDK 35と互換性がある)

# 準備するもの

1. AndroidをコンパイルできるPCまたはサーバー
2. MIUI公式ファームウェア
3. インターネット接続
4. Xiaomi 10S

# 準備開始

必要なライブラリパッケージのインストール
安装编译依赖	
```
sudo apt install bc bison build-essential ccache curl flex g++-multilib gcc-multilib git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev libelf-dev liblz4-tool libncurses5 libncurses5-dev libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev git
```
	
配置 repo	
```
sudo curl https://storage.googleapis.com/git-repo-downloads/repo > /usr/bin/repo	
sudo chmod a+x /usr/bin/repo
```

# LineageOSソースコードの同期

デバイス適応の初期段階では、LineageOSを使ってデバイスを立ち上げることをお勧めする。

新建文件夹用于存放源码
```
mkdir lineage	
```	
初始化 repo
```
repo init -u https://github.com/LineageOS/android.git -b lineage-19.1 # 同步完整仓库, 带提交历史, 占用空间大	
repo init -u https://github.com/LineageOS/android.git -b lineage-19.1 --depth=1 # 仅拉取最新提交, 不带提交历史, 占用空间小	
```
开始同步

```
repo sync	
```

# 適用開始

MIUIを抽出する

IMSのようないくつかの機能を動作させるためには、ストックシステムにあるいくつかのファイルが必要です。

この操作では、ストックシステムを展開する必要があります。
下载解包工具	
```
git clone https://github.com/ShivamKumarJha/android_tools --depth=1	
cd android_tools
```
	
初始化工具所需环境	
```
chmod +x setup.sh	
sudo bash setup.sh
```
	
开始解包	
```
./tools/rom_extract.sh MIUI_PACKAGE.zip
```

# 必要なファイルの抽出

# vendorとodmの抽出

prebuilt vendorなので、元となるシステムからvendor、odmイメージを抽出する必要があります。
```
unzip MIUI_PACKAGE.zip	
./../android_tools/tools/Firmware_extractor/tools/update_payload_extractor/extract.py payload.bin --output_dir images/
```

次に、imagesディレクトリからvendor.imgとodm.imgをdevice/xiaomi/thym-prebuilt/にコピーする。


# kernelの抽出

適応プロセスの初期段階なので、適応プロセスを簡略化するために、prebuilt kernelを使用する。
下载 magiskboot 用于解包 boot.img 以及 vendor_boot.img	
```
wget https://github.com/TeamWin/external_magisk-prebuilt/raw/android-12.1/prebuilt/magiskboot_x86	
chmod a+x magiskboot_x86
```
	
解包 boot.img 以及 vendor_boot.img	
```
./magiskboot_x86 unpack boot.img	
./magiskboot_x86 unpack vendor_boot.img
```

次に、展開したkernel、dtb、dtbo.imgをdevice/xiaomi/thyme-prebuilt/にコピーする。

# デバイスツリーの構築を始める

# デバイスツリーを初期化し、リカバリーをコンパイルする

デバイスツリーを初期化し、リカバリーをコンパイルしてカーネルの可用性を検証する必要があります。

# Android.mk

Android makeコンパイラシステムにおけるコンパイル設定ファイルです。

Androidコンパイラシステムは、デバイスフォルダ内のAndroid.mkを含め、ソースディレクトリ内のすべてのAndroid.mkファイルを含みます。 カレントフォルダ内のすべてのmakefileファイルをこのファイルに含ませる必要があります。Androidコンパイラは、デバイスのフォルダ内の他のmakefileは含みません。
```
LOCAL_PATH := $(call my-dir) # 设置一个 LOCAL_PATH 变量, 并将当前文件夹的路径赋值给它	
ifeq ($(TARGET_DEVICE),thyme) # 如果 TARGET_DEVICE 变量等于 thyme	
subdir_makefiles=$(call first-makefiles-under,$(LOCAL_PATH)) # 设置一个 subdir_makefiles 变量, 并将当前文件夹下的所有 makefile 文件赋值给它	
$(foreach mk,$(subdir_makefiles),$(info including $(mk) ...)$(eval include $(mk))) # 遍历 subdir_makefiles 变量中的所有 makefile 文件, 并 include 进来	
endif # 结束 if 语句
```

# Android.bp

soongコンパイルシステム導入後のAndroidのコンパイル設定ファイルです。Androidのコンパイルシステムは、ソースディレクトリにある全てのAndroid.bpファイルをインクルードし、デバイスフォルダにあるAndroid.bpもインクルードします。 現時点では、外部のsoongモジュールをコンパイルする必要はありません。
```
soong_namespace { // soong 编译系统的命名空间	
    imports: [], // 导入的 soong 模块路径	
}	
```

# AndroidProducts.mk

ここではデバイスのコンパイル構成を定義して、複数のデバイスや同じデバイスのバリエーションをコンパイルするために、複数のデバイス構成を定義することができます。
PRODUCT_MAKEFILES := \	
    $(LOCAL_DIR)/lineage_thyme.mk # 要使用的设备编译配置文件	
	
COMMON_LUNCH_CHOICES := \ # 定义可用的 lunch 选项	
    lineage_thyme-eng \ # eng 版本	
    lineage_thyme-userdebug \ # userdebug 版本	
    lineage_thyme-user # user 版本	

# lineage_thyme.mk

これはデバイスのコンパイル・コンフィギュレーション・ファイルで、デバイスの基本情報とコンパイル・コンフィギュレーションを定義する必要がある。

```
Inherit common products	
$(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit.mk) # 继承 core_64_bit.mk 编译配置	
$(call inherit-product, $(SRC_TARGET_DIR)/product/full_base_telephony.mk) # 继承 full_base_telephony.mk 编译配置	
	
Inherit device configurations	
$(call inherit-product, device/xiaomi/thyme/device.mk) # 继承 device/xiaomi/thyme/device.mk 编译配置	
	
Inherit common ArrowOS configurations	
$(call inherit-product, vendor/arrow/config/common.mk) # 继承 vendor/arrow/config/common.mk 编译配置	
	
PRODUCT_CHARACTERISTICS := nosdcard # 产品特性, 这里表示不支持 sd 卡	
	
PRODUCT_BRAND := Xiaomi # 产品品牌	
PRODUCT_DEVICE := thyme # 产品设备名	
PRODUCT_MANUFACTURER := Xiaomi # 产品制造商	
PRODUCT_MODEL := M2102J2SC # 产品型号	
PRODUCT_NAME := lineage_thyme # 产品名	
	
PRODUCT_GMS_CLIENTID_BASE := android-xiaomi # 产品 GMS 客户端 ID	
```

# BoardConfig.mk

デバイスのボードレベル構成ファイルで、デバイスのハードウェア構成を定義する必要があります。
```
DEVICE_PATH := device/xiaomi/thyme # 定义一个 DEVICE_PATH 变量,     指向设备文件夹
```

# A/B
```
AB_OTA_UPDATER := true # 开启 A/B OTA 更新

AB_OTA_PARTITIONS += \ # A/B OTA 更新的分区
    boot \
    dtbo \
    product \
    system \
    system_ext \
    vbmeta \
    vbmeta_system
```

# アーキテクチャ
```
TARGET_ARCH := arm64 # 指定目标架构为 arm64
TARGET_ARCH_VARIANT := armv8-a # 指定目标架构变体为 armv8-a
TARGET_CPU_ABI := arm64-v8a # 指定目标 CPU ABI 为 arm64-v8a
TARGET_CPU_ABI2 := # 指定目标 CPU ABI 2 为空
TARGET_CPU_VARIANT := generic # 指定目标 CPU 变体为 generic
TARGET_CPU_VARIANT_RUNTIME := kryo385 # 指定目标 CPU 变体运行时为 kryo385

TARGET_2ND_ARCH := arm # 指定第二目标架构为 arm
TARGET_2ND_ARCH_VARIANT := armv8-a # 指定第二目标架构变体为 armv8-a
TARGET_2ND_CPU_ABI := armeabi-v7a # 指定第二目标 CPU ABI 为 armeabi-v7a
TARGET_2ND_CPU_ABI2 := armeabi # 指定第二目标 CPU ABI 2 为 armeabi
TARGET_2ND_CPU_VARIANT := generic # 指定第二目标 CPU 变体为 generic
TARGET_2ND_CPU_VARIANT_RUNTIME := kryo385 # 指定第二目标 CPU 变体运行时为 kryo385
```

# ブートローダー
```
TARGET_BOOTLOADER_BOARD_NAME := kona # 指定目标 bootloader 名为 kona, 该值可以从 MIUI 的 vendor/build.prop 中获取
```

# カーネル
```
BOARD_BOOT_HEADER_VERSION := 3 # 指定内核头版本为 3, 内核头版本从 3 开始支持 vendor_boot 分区
BOARD_KERNEL_BASE := 0x00000000 # 指定内核基地址为 0x00000000
BOARD_KERNEL_BINARIES := kernel # 指定内核二进制文件名为 kernel
BOARD_KERNEL_CMDLINE := console=ttyMSM0,115200n8 androidboot.hardware=qcom  # 指定内核命令行
BOARD_KERNEL_CMDLINE += androidboot.console=ttyMSM0 androidboot.memcg=1
BOARD_KERNEL_CMDLINE += lpm_levels.sleep_disabled=1
BOARD_KERNEL_CMDLINE += msm_rtb.filter=0x237 service_locator.enable=1
BOARD_KERNEL_CMDLINE += androidboot.usbcontroller=a600000.dwc3 swiotlb=0
BOARD_KERNEL_CMDLINE += loop.max_part=7 cgroup.memory=nokmem,nosocket
BOARD_KERNEL_CMDLINE += pcie_ports=compat loop.max_part=7
BOARD_KERNEL_CMDLINE += iptable_raw.raw_before_defrag=1
BOARD_KERNEL_CMDLINE += ip6table_raw.raw_before_defrag=1
BOARD_KERNEL_CMDLINE += androidboot.selinux=permissive
BOARD_KERNEL_IMAGE_NAME := Image # 指定内核镜像名为 Image
BOARD_KERNEL_PAGESIZE := 4096 # 指定内核页大小为 4096
BOARD_KERNEL_SEPARATED_DTBO := true # 指定内核分离 DTBO
BOARD_MKBOOTIMG_ARGS := --header_version $(BOARD_BOOT_HEADER_VERSION) # 指定 mkbootimg 参数, 这里指定了内核头版本
KERNEL_LD := LD=ld.lld # 指定内核链接器为 ld.lld
TARGET_KERNEL_ADDITIONAL_FLAGS := DTC_EXT=$(shell pwd)/prebuilts/misc/linux-x86/dtc/dtc # 指定使用外部 DTC
TARGET_KERNEL_APPEND_DTB := false # 指定不追加 DTB
BOARD_INCLUDE_DTB_IN_BOOTIMG := true # 指定在 boot.img 中包含 DTB
TARGET_KERNEL_ARCH := arm64 # 指定内核架构为 arm64
TARGET_KERNEL_HEADER_ARCH := arm64 # 指定内核头架构为 arm64

ifeq ($(TARGET_PREBUILT_KERNEL),) # 如果没有使用预编译内核
  TARGET_KERNEL_CONFIG := thyme_defconfig # 指定内核配置文件为 thyme_defconfig
  TARGET_KERNEL_CLANG_COMPILE := true # 指定使用 Clang 编译内核
  TARGET_KERNEL_SOURCE := kernel/xiaomi/thyme # 指定内核源码路径为 kernel/xiaomi/thyme
endif
```

# メタデータ
```
BOARD_USES_METADATA_PARTITION := true # 使用 metadata 元数据加密
```

# パーティション
```
BOARD_BOOTIMAGE_PARTITION_SIZE := 201326592 # 指定 boot 分区大小为 201326592
BOARD_DTBOIMG_PARTITION_SIZE := 33554432 # 指定 dtbo 分区大小为 33554432
BOARD_FLASH_BLOCK_SIZE := 262144 # 指定刷入块大小为 262144
BOARD_USERDATAIMAGE_PARTITION_SIZE := 114135379968 # 指定 userdata 分区大小为 114135379968
BOARD_VENDOR_BOOTIMAGE_PARTITION_SIZE := 100663296 # 指定 vendor_boot 分区大小为 100663296

BOARD_THYME_DYNAMIC_PARTITIONS_PARTITION_LIST := product system system_ext # 指定动态分区列表
BOARD_SUPER_PARTITION_SIZE := 9126805504 # 指定 super 分区大小为 9126805504
BOARD_SUPER_PARTITION_GROUPS := thyme_dynamic_partitions # 指定 super 分区组为 thyme_dynamic_partitions
BOARD_THYME_DYNAMIC_PARTITIONS_SIZE := 4559208448 # 指定动态分区组 thyme_dynamic_partitions 大小为 4559208448

BOARD_PARTITION_LIST := $(call to-upper, $(BOARD_THYME_DYNAMIC_PARTITIONS_PARTITION_LIST)) # 遍历 BOARD_THYME_DYNAMIC_PARTITIONS_PARTITION_LIST 并赋值给 BOARD_PARTITION_LIST
$(foreach p, $(BOARD_PARTITION_LIST), $(eval BOARD_$(p)IMAGE_FILE_SYSTEM_TYPE := ext4)) # 遍历 BOARD_PARTITION_LIST 并赋值给 p, 然后设置 BOARD_$(p)IMAGE_FILE_SYSTEM_TYPE := ext4
$(foreach p, $(BOARD_PARTITION_LIST), $(eval TARGET_COPY_OUT_$(p) := $(call to-lower, $(p)))) # 遍历 BOARD_PARTITION_LIST 并赋值给 p, 然后设置 TARGET_COPY_OUT_$(p) := $(call to-lower, $(p))

BOARD_USERDATAIMAGE_FILE_SYSTEM_TYPE := f2fs # 指定 userdata 分区文件系统类型为 f2fs
```

# プラットフォーム
```
BOARD_USES_QCOM_HARDWARE := true # 指定使用 Qualcomm 硬件
TARGET_BOARD_PLATFORM := kona # 指定平台为 kona
```

# リカバリー
```
BOARD_INCLUDE_RECOVERY_DTBO := true # 指定包含 recovery DTBO
BOARD_USES_RECOVERY_AS_BOOT := true # 指定 recovery 在 boot 分区中
TARGET_NO_RECOVERY := true # 指定设备没有 recovery 分区
TARGET_RECOVERY_FSTAB := $(DEVICE_PATH)/recovery/recovery.fstab # 指定 recovery fstab 文件
TARGET_RECOVERY_PIXEL_FORMAT := RGBX_8888 # 指定 recovery 像素格式
TARGET_USERIMAGES_USE_F2FS := true # 指定 userdata 使用 f2fs
```

# Verified Boot
```
BOARD_AVB_ENABLE := true # 指定启用 AVB
BOARD_AVB_MAKE_VBMETA_IMAGE_ARGS += --flags 3 # 指定 AVB flags 为 3
BOARD_AVB_MAKE_VBMETA_IMAGE_ARGS += --set_hashtree_disabled_flag # 指定 AVB 为禁用
BOARD_AVB_VBMETA_SYSTEM := system system_ext# 指定使用 AVB 的分区
BOARD_AVB_VBMETA_SYSTEM_ALGORITHM := SHA256_RSA2048 # 指定 AVB 加密算法为 SHA256_RSA2048
BOARD_AVB_VBMETA_SYSTEM_KEY_PATH := external/avb/test/data/testkey_rsa2048.pem # 指定 AVB 密钥
BOARD_AVB_VBMETA_SYSTEM_ROLLBACK_INDEX := $(PLATFORM_SECURITY_PATCH_TIMESTAMP) # 指定 AVB 回滚索引
BOARD_AVB_VBMETA_SYSTEM_ROLLBACK_INDEX_LOCATION := 1 # 指定 AVB 回滚索引位置
```

# VNDK
```
BOARD_VNDK_VERSION := current # 指定 VNDK 版本为 current
```

# device.mk

これはデバイスのコンパイル設定ファイルで、lineage_thyme.mkと同様に、他のコンパイル設定ファイルをインクルードしたり、コンパイルするモジュールを指定したりすることができる。

```
Enable updating of APEXes
$(call inherit-product, $(SRC_TARGET_DIR)/product/updatable_apex.mk) # 启用 apex 更新

Virtual A/B
$(call inherit-product, $(SRC_TARGET_DIR)/product/virtual_ab_ota.mk) # 启用虚拟 A/B

Enable Dalvik
$(call inherit-product, frameworks/native/build/phone-xhdpi-6144-dalvik-heap.mk) # 导入 6G dalvik 配置
```

# API
```
PRODUCT_SHIPPING_API_LEVEL := 30 # 指定预装 Android API 级别为 30, 例如 thyme 预装 Android 11, 故此处为 30
```

# A/B
```
AB_OTA_POSTINSTALL_CONFIG += \
    RUN_POSTINSTALL_system=true \
    POSTINSTALL_PATH_system=system/bin/otapreopt_script \
    FILESYSTEM_TYPE_system=ext4 \
    POSTINSTALL_OPTIONAL_system=true
```

# ブートアニメーション
```
TARGET_SCREEN_HEIGHT := 2400 # 指定屏幕高度为 2400
TARGET_SCREEN_WIDTH := 1080 # 指定屏幕宽度为 1080
```

# Common init scripts
```
PRODUCT_PACKAGES += \
    init.recovery.qcom.rc # 编译额外的自定义 rc 脚本
```

# fastbootd
```
PRODUCT_PACKAGES += \
    android.hardware.fastboot@1.0-impl-mock \ # 编译 fastbootd
    fastbootd
```

# F2FS utilities
```
PRODUCT_PACKAGES += \
    sg_write_buffer \ # 编译 f2fs 工具
    f2fs_io \
    check_f2fs
```

# パーティション
```
PRODUCT_USE_DYNAMIC_PARTITIONS := true # 指定使用动态分区
PRODUCT_BUILD_SUPER_PARTITION := false # 指定不编译 super 分区
```

# Soong 名前空間
```
PRODUCT_SOONG_NAMESPACES += \
    $(LOCAL_PATH) # 指定 soong 命名空间
```

# アップデートエンジン
```
PRODUCT_PACKAGES += \
    otapreopt_script \ # 编译 otapreopt 脚本
    update_engine \ # 编译 update_engine
    update_engine_sideload \ # 编译 update_engine_sideload
    update_verifier \ # 编译 update_verifier

PRODUCT_PACKAGES_DEBUG += \
    update_engine_client \ # 编译 update_engine_client

PRODUCT_HOST_PACKAGES += \
    brillo_update_payload \ # 编译 brillo_update_payload
```

# Vendor boot
```
PRODUCT_COPY_FILES += \
    $(LOCAL_PATH)/rootdir/etc/fstab.qcom:$(TARGET_COPY_OUT_VENDOR_RAMDISK)/first_stage_ramdisk/fstab.qcom # 复制 fstab 到 vendor ramdisk
```

# VNDK
```
PRODUCT_TARGET_VNDK_VERSION := 30 # 指定 VNDK 版本为 30, 该值可以在 vendor/build.prop 中找到
```

# Recoveryに必要な基本ファイルのインポート

これは fstab ファイルで、パーティションのマウントポイントと、読み取り可能かどうかなどのマウントポイントのプロパティを指定します。このファイルがないと、Linuxカーネルは必要なパーティションをマウントできず、AndroidやRecoveryは起動しません！

このファイルは、次のコマンドを使って、ベンダパーティションで見つけることができます。
```
cd dump/vendor	
find | grep fstab.qcom	
```

これをdeviceフォルダのrootdir/etc/ディレクトリにコピーする。

その後、device.mkのPRODUCT_COPY_FILESを使用して、ramdiskまたはvendor ramdiskにコピーすることもできる。あるいは、rootdirディレクトリに新しいAndroid.mkを作成してカスタム・モジュールを定義し、device.mkでコンパイルを指定することもできる。
https://github.com/Lynnrin-Studio/android_device_xiaomi_thyme/commit/cce87ffbd1416eaa8cda26a71aeaebc19890c505#diff-247ee86229a709a6e2eedfc1d3c4a557825aee073e20b0112ab76f4ca8e4bc4eR71-R72

# Recoveryのコンパイル開始

それができたら、リカバリーのコンパイルを始めてテストしましょう。
```
. build/envsetup.sh # 初始化编译环境	
lunch lineage_thyme-userdebug # 初始化设备编译环境	
m bootimage # 编译 boot image, 由于是 A/B 设备, 故此处编译 boot image 而不是 recovery image
```

コンパイル完了後、デバイスにフラッシュをする
```
fastboot flash boot out/target/product/thyme/boot.img # 刷入 boot image	
fastboot reboot recovery # 重启到 recovery
```

コンパイル後、デバイスがリカバリーに移行できれば次のステップに進むことができるが、そうでなければ記事の最後にあるデバッグ・ガイドを参照してデバッグを行ってほしい。

# デバイスツリーを完成させてAndroidのコンパイルを開始する準備をする

このセクションは大部分なので、私のGitHubリポジトリのコミット履歴を参照してください。

proprietary-files.txt, extract-files.sh 和 setup-makefiles.sh

これらの3つのファイルは、生成、変更、抽出のためにvendor tree内のファイルをコピーします。vendor tree内のファイルを生成または変更したい場合は、必ずextract-files.shスクリプトを使用して、生成または変更されたファイルを抽出してください。詳細については、以下のLineageOS公式ドキュメントを参照してください。

1. https://wiki.lineageos.org/extracting_blobs_from_zips
2. https://wiki.lineageos.org/proprietary_blobs

proprietary-files.txt
このファイルは、プロプライエタリ・システムから抽出する必要があるファイルのリストと、そのファイルの対象となる場所に属する。extract-files.shは、このファイル内のファイルを反復処理し、プロプライエタリ・システムからvendor treeに抽出します。

extract-files.sh
このスクリプトは、ストックシステムからproprietary-files.txtからファイルを抽出し、それらをvendor treeに配置した後、setup-makefiles.shスクリプトを呼び出してmakefileを生成する。

setup-makefiles.sh
このスクリプトは、makefileを生成するためにextract-files.shから呼び出される。

FCM 或いは manifest.xml 或いは compatibility_matrix.device.xml
これはHALマニフェストファイルであり、デバイスがサポートするHALとそのバージョンを指定します。 Androidはマニフェスト内のHALに従って対応するHALをロードします。 詳細は公式AOSPドキュメントを参照してください: https://source.android.com/docs/core/architecture/vintf?hl=ja
bootctrl および gpt-utils または mtk_plpath_utils
これらのモジュールは、A/Bデバイスの非破壊アップグレードに必要である。詳細はAOSPの公式ドキュメントを参照されたい： https://source.android.com/docs/core/ota/ab
```
⚠️ 注：QualcommデバイスはMediaTekデバイスとは異なるbootctrlを使用します。
```
クアルコムデバイス用のbootctrlとgpt-utilsはCAFまたはCLOから取得できます。

1. CAF
2. CLO
```
⚠️ 注：Qualcommは2022年5月31日にCAFの更新を停止し、2023年5月31日にCAFの使用を完全に停止することを決定したため、CLOからbootctrlとgpt-utilsを使用することを推奨します。
```
MediaTek デバイスについては、bootctrlとmtk_plpath_utilsを参照してください。

1. bootctrl
2. mtk_plpath_utils

上記の2つのモジュールがMediaTekデバイスで動作しない場合は、ビルド済みのbootctrlとmtk_plpath_utilsの使用を検討してください。

# VoLTE

VoLTEをサポートするためには、いくつかのビルドをコンパイルし、純正システムからいくつかのファイルを抽出する必要がある。

Qualcomm デバイスリファレンス:

- https://github.com/Lynnrin-Studio/android_device_xiaomi_thyme/commit/e6d415fbc9c1ad947ceea4e2860a7a1101e8feec
- https://github.com/Lynnrin-Studio/android_device_xiaomi_thyme/blob/twelve/proprietary-files.txt#L78-L87

MediaTek デバイスリファレンス:

- https://github.com/Lynnrin-Studio/android_device_xiaomi_chopin/blob/lineage-18.1/device.mk#L105-L121
- https://github.com/Lynnrin-Studio/android_device_xiaomi_chopin/blob/lineage-18.1/proprietary-files.txt#L5-L41

また、ROMがMTK IMSサポートでコンパイルされていることを確認してください: https://gerrit.pixelexperience.org/q/topic:mtk-ims

# Overlay

Overlayは、ステータスバーの高さや角丸の大きさなど、いくつかのシステム機能を動的に調整するために非常に重要なものです。

Overlayの設定の一部は、ベンダーのシステムから次の場所で抽出できます。
```
  vendor/overlay
  product/overlay
```

独自のオーバーレイを追加する必要がある場合は、これらのリンクを参照して、利用可能なオーバーレイを確認することができます:
```
  framework/base
  framework/base/packages/SystemUI
  packages/apps/Settings
```

さらにオーバーレイを見つける必要がある場合は、[cs.android.com](https://cs.android.com/)
に行き、対応するアプリモジュールのres/values/ディレクトリを見ることができます。

# SELinux

SELinuxは、一部の悪意のあるアプリがシステムファイルを読み取るのを防ぐセキュリティ・メカニズムだが、このメカニズムは、間違ったSepolicyルールが一部のハードウェアやソフトウェアを不適切に動作させるなど、いくつかの問題にもつながる可能性がある。 しかし、この仕組みは、例えば間違ったSepolicyルールが一部のハードウェアやソフトウェアを不適切に動作させるなど、いくつかの問題にもつながる可能性がある。適応の初期段階では、SELinuxをToleranceに設定し、ハードウェアとソフトウェアの適応作業が基本的に完了するまで待ってから、SELinuxをEnforcingに設定することをお勧めする。

Sepolicyのルールの作成については、以下のリンクを参照してください:

1. https://source.android.com/security/selinux/
2. https://www.cnblogs.com/schips/p/android-selinux_about_avc.html
3. https://lineageos.org/engineering/HowTo-SELinux

# ハードウェア機能の修理

一般的に、 prebuilt vendor treeによってコンパイルされたROMは、ほとんどのハードウェアで動作しますが、一部のモデルでは、指紋認証やLED indicatorなど、ハードウェアが正しく動作するためにいくつかの追加修正が必要になる場合があります。

# 画面下指紋認証
```
⚠️ 注：このセクションでは、光学式画面下指紋認証、グローバルHBMを使用するXiaomi 10Sを例にしています。お使いのデバイスがローカルHBMの場合、実装が若干異なる可能性があります。
```
Xiaomi Mi 10Sでは、画面下指紋認証を使用しているため、vendorにコンパイルされているFingerprint 2.1 HALを直接使用することはできません。
Android 12以上では、UDFPSイベントを処理するためにFingerprint 2.3 HALを手動でコンパイルする必要があります。

UDFPSはGoogleがAndroid 12で新たに実装した画面下指紋で、ソースコードを読むことができる。
- https://cs.android.com/android/platform/superproject/+/master:frameworks/base/packages/SystemUI/src/com/android/systemui/biometrics/

## Android 12以下のシステム
正確な実装は、以下のコミット履歴を参照してください。
- https://github.com/Lynnrin-Studio/android_device_xiaomi_thyme/commits/lineage-18.1/fod
```
⚠️ 注：
Xiaomi 10SはGlobal HBMを使用しているため、カーネル調光またはフレームワーク調光を行う必要がある。
そうしないと、指紋ロック解除を使用する際に全画面の輝度が最大に設定されてしまう。
Xiaomi 10Sはprebuilt kernelを使用しているため、フレームワークを使用することしかできない。
Xiaomi 10Sはprebuilt kernelを使用しているため、調光にはフレームワークしか使用できない。
https://github.com/dataoutputstream/platform_frameworks_base/commit/86a2a0901b8af99019996286decbc6510f77e375
```

## Android 12以上のシステム
実装は以下のコミット履歴にある。
- https://github.com/Lynnrin-Studio/android_device_xiaomi_thyme/commit/a681ba360b31fd7103f1323926c5561fcd13c5a4
さらに、画面下の指紋認証機能を正しく動作させるために、オーバーレイにいくつかのパラメータを設定する必要がある。
- https://github.com/Lynnrin-Studio/android_device_xiaomi_thyme/commit/263342af84e43838bfe37f8b3ad24c6f14659125
```
⚠️ 注：
Xiaomi 10SはGlobal HBMを使用しているため、カーネル調光かフレームワーク調光を行う必要がある。
そうしないと、指紋認証でロック解除するときに全画面の輝度が最大に設定されてしまう。
Xiaomi 10Sはprebuilt kernelを使用しているため、フレームワークでしかできない。
調光の実装については、こちらのコミットを参照してください
https://review.arrowos.net/c/ArrowOS/android_frameworks_base/+/16767
⚠️ 注：
Android 13ではUDFPS部分がKotlinで書き換えられているため、上記の実装はAndroid 13と互換性がありません。
https://github.com/PixelExperience-Devices/kernel_xiaomi_thyme/commits/53248741c3a24008a440e55cb96869d7c26136e9
```

# Bluetooth Audio

## Android 12以下のシステム
通常、純正システムはQCOM BTを使用しているため、このコミットで説明されているように、vendorのQCOM BTをサポートするためにいくつかの変更を行う必要があります。
- https://github.com/Lynnrin-Studio/android_device_xiaomi_thyme/commit/ae9c7040b68d9904c55bf5ce810f5ce8a3212662

## Android 13
GoogleはAndroid 13でBluetoothを含むいくつかのコンポーネントをモジュール化したが、vendorのQCOM BTはこの変更に対応していないため、
"android.hardware.bluetooth.audio"の修正版をコンパイルしてQCOM BTを無効にし、AOSP BTに切り替える必要がある。

具体的な実装については、これらのコミットを参照のこと:

1. https://review.arrowos.net/q/topic:13-gsi
2. https://github.com/Lynnrin-Studio/android_device_xiaomi_nabu/commit/b26c98951d1e8c0397983ed4f96224bd9e836489
3. https://github.com/Lynnrin-Studio/android_device_xiaomi_nabu/commits/arrow-13.0/bluetooth/audio

# Audio

デバイスによってはlibvolumelistenerがメーカーによって変更されている可能性があるので、このコミットでlibvolumelistenerを置き換える必要がある。
- https://github.com/Lynnrin-Studio/android_device_xiaomi_nabu/commit/27e74adfbc30444380e471841957c56b74ffb79b

# デバッグガイド

## デバイスが純正ロゴで止まっている（1つの画面で止まっている）
これにはさまざまな理由がある。

## データパーティションが正しくフォーマットされていない
1. recoveryでデータパーティションをフォーマットしてみる
2. fastboot -wでデータパーティションをフォーマットしてみる
3. 純正リカバリーを使用してデータパーティションをフォーマットしてみる
4. fstabでデータパーティションの暗号化をオフにする

## Sepolicyルールの設定ミス
1. SELinuxの状態をpermissiveに設定してみる
2. logcatまたはpstoreのログを取得し、関連するエラーをチェックし、そのエラーを修正する

## vender imageが正しく設定されていない
1. BoardConfigでvendor imageが正しく設定されていることを確認します

## ramdiskが正しく設定されていない
1. fstabがramdiskに正しくコピーされていることを確認する

## kernelが正しく設定されていないか、build済みkernelが利用できない
1. kernelが正しく設定されているか確認する
2. OSS kernelに切り替える

## BPFローディングの失敗
以下のコミットを使用して、BPFロードの問題を修正してみてください
- Prebuilt kernel
1. https://github.com/AcmeUI/android_frameworks_libs_net/commit/6fcad94ca26fbcf17ae33fca864ab80bf2b1d642
2. https://github.com/AcmeUI/android_system_bpf/commit/50a4dece82954745e40b5d354cfd222642f6fdce
3. https://github.com/AcmeUI/android_frameworks_libs_net/commit/08de55ee6bb3774123ed8bd303855c542093ebb4

- OSS kernel
1. https://github.com/PixelExperience-Devices/kernel_xiaomi_thyme/commit/f1facc1aa372dd2c9eb1336d57e574ace2cbfec7

## logcatまたはpstoreのログを取得する

上記のどの方法でも問題が解決しない場合は、ログを取得して問題を分析することができます。

## logcatログを取得する
デバイスがコンピューターに接続され、adb devicesで認識できる場合、adb logcatを使ってlogcatログを取得することができます。
error、crash、fatalなどのキーワードでlogcatログをチェックすれば、問題を突き止めることができる。

## pstoreログの取得
デバイスがコンピューターに接続されていて、adb devicesで認識できない場合は、pstoreのログを取得する必要があります。
```
⚠️ 注：pstoreはカーネルで有効になっている必要がある。有効になっていないと、pstoreのログを取得できない。
```
1. デバイスをrecoveryに再起動する
2. recoveryのためにadbを有効にする（まだ有効になっていない場合）
3. adb pull /sys/fs/pstoreを使ってpstoreのログを取得する

error、crash、fatalなどのキーワードでpstoreのログをチェックすることで、問題を突き止めることができる。


## デバイスが起動アニメーションで固まる（2つの画面で固まる）

デバイスカードの2番目の画面は、システムが起動し、kernel部分は正常に動作しているが、システムサービスが正常に開始されていないことを意味する。

## logcatログにlinker・エラー

一般的にこのエラーは、ダイナミック・ライブラリーが正しくロードされていないか、不足していることを意味する。

## logcatログでパーミッションが拒否されたエラー

一般的にこのエラーは、SELinuxが正しく設定されていないことを意味する。

## logcat ログに致命的なシグナル 11 (SIGSEGV)、コード 1 (SEGV_MAPERR) エラー

一般的に、このエラーはシステムサービスがクラッシュしたことを意味します。特に、報告されたエラーのログを参照し、クラッシュを修正します！

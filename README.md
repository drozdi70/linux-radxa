**linux-radxa**

Kernel 6.19.6 for testing purpose of rockchip Radxa Rock-2a and Nanopi Zero2 projects (RK3528)

<ins>Steps:</ins>

```
git clone https://github.com/armbian/build
cp rockchip64_common.inc.new build/config/sources/families/include/rockchip64_common.inc
cp rock-2a.conf.new build/config/boards/rock-2a.conf
cd build
./compile.sh build BOARD=rock-2a BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no RELEASE=noble
```

<ins>Config files:</ins>

**rock-2a.conf.new**

```
# Rockchip RK3528 quad core 1-4GB SoC 1xGBe 0-32GB eMMC
BOARD_NAME="ROCK 2A"
BOARD_VENDOR="radxa"
BOARDFAMILY="rk35xx"
BOOTCONFIG="rock-2-rk3528_defconfig"
BOARD_MAINTAINER="CodeChenL"
KERNEL_TARGET="vendor,edge"
KERNEL_TEST_TARGET="vendor,edge"
FULL_DESKTOP="yes"
BOOT_LOGO="desktop"
BOOT_FDT_FILE="rockchip/rk3528-rock-2a.dtb"
BOOT_SCENARIO="spl-blobs"
IMAGE_PARTITION_TABLE="gpt"
enable_extension "radxa-aic8800"
AIC8800_TYPE="usb"

function post_family_config__rock2a_use_mainline_uboot() {
    [[ "${BRANCH}" == "vendor" ]] && return 0
        display_alert "$BOARD" "Mainline U-Boot overrides for $BOARD - $BRANCH" "info"

        # To reuse ATF code in rockchip64_common, let's change the BOOT_SCENARIO and call prepare_boot_configuration() again
        # BOOT_SCENARIO="tpl-blob-atf-mainline"
        # prepare_boot_configuration

        # declare -g BOOTCONFIG="generic-rk3528_defconfig"
        declare -g BOOTDELAY=1
        declare -g BOOTSOURCE="https://github.com/u-boot/u-boot.git"
        declare -g BOOTBRANCH="tag:v2026.04-rc3"
        declare -g BOOTPATCHDIR="v2026.04-rc3"
        declare -g BOOTDIR="u-boot-${BOARD}"
        declare -g UBOOT_TARGET_MAP="BL31=${RKBIN_DIR}/${BL31_BLOB} ROCKCHIP_TPL=${RKBIN_DIR}/${DDR_BLOB};;u-boot-rockchip.bin"
        unset uboot_custom_postprocess write_uboot_platform write_uboot_platform_mtd # disable stuff from rockchip64_common; we're using binman here which does all the work already
        declare -g BOOTSCRIPT="boot-rockchip64-ttyS0.cmd:boot.cmd"
        declare -g SERIALCON="ttyS0"

        # Just use the binman-provided u-boot-rockchip.bin, which is ready-to-go
        function write_uboot_platform() {
                dd "if=$1/u-boot-rockchip.bin" "of=$2" bs=32k seek=1 conv=notrunc status=none
        }

        function write_uboot_platform_mtd() {
                flashcp -v -p "$1/u-boot-rockchip-spi.bin" /dev/mtd0
        }
}
```


**rockchip64_common.inc.new**

```
        edge)
                declare -g KERNEL_MAJOR_MINOR="6.19"
                declare -g LINUXFAMILY=rockchip64
                declare -g LINUXCONFIG='linux-rockchip64-'$BRANCH
                declare -g KERNELSOURCE='https://github.com/drozdi70/linux-radxa.git'
                declare -g KERNELBRANCH='branch:main'
                declare -g KERNELPATCHDIR='rockchip64-edge-6.19'
                declare -g SERIALCON="ttyS0"
                ;;
```

Eventually new patches to apply (from directory called patches):

```
mkdir -p build/userpatches/kernel/archive/rockchip64-6.19/
mkdir -p build/userpatches/kernel/rockchip64-edge-6.19
cp patches/*.patch build/userpatches/kernel/archive/rockchip64-6.19/
cp patches/*.patch build/userpatches/kernel/rockchip64-edge-6.19/
```

**Defconfig**

Please be sur you include all drivers you need in your kernel, for example:

```
...
CONFIG_USB_OTG=y
CONFIG_PHY_ROCKCHIP_PCIE=y
CONFIG_ROCKCHIP_PHY=y
CONFIG_USB_DWC3=y
CONFIG_USB_OHCI_HCD=y
CONFIG_USB_EHCI_HCD=y
...
```

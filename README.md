**linux-radxa**

Kernel 6.19.6 for testing purpose of rockchip Radxa Rock-2a and Nanopi Zero2 projects (RK3528) with Armbian

<ins>Steps:</ins>

```
git clone https://github.com/armbian/build
cp rockchip64_common.inc.new build/config/sources/families/include/rockchip64_common.inc
cp rock-2a.conf.new build/config/boards/rock-2a.conf
cp defconfig build/config/kernel/linux-rockchip64-edge.config
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

**Defconfig (linux-rockchip64-edge.config)**

Please be sure you include all needed drivers you in your kernel, for example:

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

================================================================

**Actual issues:**

1. Power domain

```
[    0.038779] rockchip-pm-domain ff600000.power-management:power-controller: power-domain: failed to get clk at index 0: -517
[    0.038802] rockchip-pm-domain ff600000.power-management:power-controller: failed to handle node power-domain: -517
[    0.442562] PM: genpd: Disabling unused power domains
[   23.012367] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ff140000.usb
[   23.012373] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ff100000.usb
[   23.012379] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to fe500000.usb
[   23.012393] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ffad0000.tsadc
[   23.012402] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ffbe0000.ethernet
[   23.012415] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ffdf0000.usb2phy
[   23.012325] platform fe500000.usb: deferred probe pending: platform: wait for supplier /soc/usb2phy@ffdf0000/otg-port
[   23.012346] platform fe500000.dwc3: deferred probe pending: platform: wait for supplier /soc/usb2phy@ffdf0000/otg-port
[   23.012351] platform ff100000.usb: deferred probe pending: platform: wait for supplier /soc/usb2phy@ffdf0000/host-port
```

2. Clock clk_usbphy_480m missing? or issue with phy-rockchip-usb.c/phy-rockchip-inno-usb2.c?

```
root@rock-2a:~# cat /sys/kernel/debug/clk/clk_summary |grep -i usb
    clk_ref_usb3otg                  1       1        0        24000000    0          0     50000      Y      usbdrd                          no_connection_id
    clk_suspend_usb3otg              1       1        0        24000000    0          0     50000      Y      usbdrd                          no_connection_id
    clk_ref_usbphy                   0       0        0        24000000    0          0     50000      N      deviceless                      no_connection_id
                aclk_usb3otg         1       1        0        198000000   0          0     50000      Y                  usbdrd                          no_connection_id
                hclk_usbhost_arb     0       0        0        148500000   0          0     50000      N                  deviceless                      no_connection_id
                hclk_usbhost         0       1        0        148500000   0          0     50000      N                  power-domain@7                  no_connection_id
                pclk_usbphy          0       1        0        99600000    0          0     50000      N                  power-domain@7                  no_connection_id
```

Kernel 6.19.6 for testing purpose of Radxa Rock-2a (RK3528) with Armbian

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

Please be sure you include all needed drivers in your kernel, for example:

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

**Radxa Rock-2A actual issues:**

1. Power domain

```
root@rock-2a:~# dmesg |grep -i power
[    0.031158] thermal_sys: Registered thermal governor 'power_allocator'
[    0.070307] rockchip-pm-domain ff600000.power-management:power-controller: power-domain: failed to get clk at index 0: -517
[    0.070328] rockchip-pm-domain ff600000.power-management:power-controller: failed to handle node power-domain: -517
[    0.975548] PM: genpd: Disabling unused power domains
[   12.522908] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to fe500000.usb
[   12.522915] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to fe4f0000.pcie
[   12.522922] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ff700000.gpu
[   12.522931] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ff7c0000.video-codec
[   12.522948] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ffad0000.tsadc
[   12.522965] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ffdf0000.usb2phy

domain                          status          children        performance
    /device                         runtime status                  managed by
------------------------------------------------------------------------------
vpu                             on                              0
    ffdc0000.phy                    unsupported                 0           SW
    ff7c0800.iommu                  suspended                   0           SW
    ffaf0000.gpio                   unsupported                 0           SW
    ffb10000.gpio                   unsupported                 0           SW
    ffbf0000.mmc                    suspended                   0           SW
    ffae0000.adc                    unsupported                 0           SW
    ffbe0000.ethernet               active                      0           SW
    ff7c0000.video-codec            suspended                   0           SW
vo                              on                              0
    ffb00000.gpio                   unsupported                 0           SW
    ffc30000.mmc                    suspended                   0           SW
venc                            on                              0
    ffa58000.i2c                    unsupported                 0           SW
    ffb20000.gpio                   unsupported                 0           SW
gpu                             off-0                           0
    ff700000.gpu                    suspended                   0           SW

```

2. Clock clk_usbphy_480m missing? or issue with phy-rockchip-usb.c/phy-rockchip-inno-usb2.c?

```
root@rock-2a:~# dmesg |grep -i usb
[    0.109508] usbcore: registered new interface driver usbfs
[    0.109542] usbcore: registered new interface driver hub
[    0.109582] usbcore: registered new device driver usb
[    0.385484] usbcore: registered new interface driver cdc_acm
[    0.385502] cdc_acm: USB Abstract Control Model driver for USB modems and ISDN adapters
[    0.385788] usbcore: registered new interface driver uas
[    0.385848] usbcore: registered new interface driver usb-storage
[    0.443550] usbcore: registered new interface driver usbhid
[    0.443567] usbhid: USB HID core driver
[   12.522855] platform fe500000.usb: deferred probe pending: platform: wait for supplier /soc/usb2phy@ffdf0000/otg-port
[   12.522879] platform ff140000.usb: deferred probe pending: platform: wait for supplier /soc/usb2phy@ffdf0000/host-port
[   12.522886] platform fe500000.dwc3: deferred probe pending: platform: wait for supplier /soc/usb2phy@ffdf0000/otg-port
[   12.522908] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to fe500000.usb
[   12.522965] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to ffdf0000.usb2phy



    clk_ref_usb3otg                  1       2        0        24000000    0          0     50000      Y      power-domain@8                  no_connection_id
                                                                                                              usbdrd                          no_connection_id
    clk_suspend_usb3otg              1       2        0        24000000    0          0     50000      Y      power-domain@8                  no_connection_id
                                                                                                              usbdrd                          no_connection_id
    clk_ref_usbphy                   0       1        0        24000000    0          0     50000      N      power-domain@7                  no_connection_id
                aclk_usb3otg         1       2        0        198000000   0          0     50000      Y                  power-domain@8                  no_connection_id
                                                                                                                          usbdrd                          no_connection_id
                hclk_usbhost_arb     0       1        0        148500000   0          0     50000      N                  power-domain@7                  no_connection_id
                hclk_usbhost         0       1        0        148500000   0          0     50000      N                  power-domain@7                  no_connection_id
                pclk_usbphy          0       1        0        99600000    0          0     50000      N                  power-domain@7                  no_connection_id
```

3. PCIE issues:

```
root@rock-2a:~# dmesg |grep -i pci
[    0.047876] /soc/pcie@fe4f0000: Fixed dependency cycle(s) with /soc/pcie@fe4f0000/legacy-interrupt-controller
[    0.128113] PCI: CLS 0 bytes, default 64
[    0.599952] dw-pcie fe4f0000.pcie: host bridge /soc/pcie@fe4f0000 ranges:
[    0.599999] dw-pcie fe4f0000.pcie:       IO 0x00fc100000..0x00fc1fffff -> 0x00fc100000
[    0.600019] dw-pcie fe4f0000.pcie:      MEM 0x00fc200000..0x00fdffffff -> 0x00fc200000
[    0.600030] dw-pcie fe4f0000.pcie:      MEM 0x0100000000..0x013fffffff -> 0x0100000000
[    0.600144] dw-pcie fe4f0000.pcie: invalid resource
[    0.600153] dw-pcie fe4f0000.pcie: Failed to initialize host
[    0.600158] dw-pcie fe4f0000.pcie: probe with driver dw-pcie failed with error -22
[   12.522915] rockchip-pm-domain ff600000.power-management:power-controller: sync_state() pending due to fe4f0000.pcie

    clk_pcie_aux                     0       1        0        24000000    0          0     50000      N      power-domain@8                  no_connection_id
          clk_ref_pcie_100m_phy      0       0        0        24000000    0          0     50000      Y            deviceless                      no_connection_id
          clk_ref_pcie_inner_phy     0       1        0        24000000    0          0     50000      Y            phy@ffdc0000                    no_connection_id
                hclk_pcie_dbi        0       1        0        198000000   0          0     50000      N                  power-domain@8                  no_connection_id
                hclk_pcie_slv        0       1        0        198000000   0          0     50000      N                  power-domain@8                  no_connection_id
                aclk_pcie            0       1        0        198000000   0          0     50000      N                  power-domain@8                  no_connection_id
                pclk_pcie_phy        0       2        0        99600000    0          0     50000      N                  phy@ffdc0000                    no_connection_id
                pclk_pcie            0       1        0        99600000    0          0     50000      N                  power-domain@8                  no_connection_id
                pclk_cru_pcie        1       2        0        99600000    0          0     50000      Y                  power-domain@8                  no_connection_id
```


****************************************************************************************

Kernel 6.19.6 for testing purpose of FriendlyELEC Nanopi Zero2 (RK3528) with Armbian

<ins>Steps:</ins>

```
git clone https://github.com/armbian/build
cp rockchip64_common.inc.new build/config/sources/families/include/rockchip64_common.inc
cp nanopi-zero2.csc.new build/config/boards/nanopi-zero2.csc
cp defconfig build/config/kernel/linux-rockchip64-edge.config
cd build
./compile.sh build BOARD=nanopi-zero2 BRANCH=edge BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no RELEASE=noble
```

<ins>Config files:</ins>

**nanopi-zero2.csc.new**

```
# Rockchip RK3528 quad core 1/2GB RAM SoC GBe eMMC USB2 USB-C PCIe 2.1
BOARD_NAME="NanoPi Zero2"
BOARD_VENDOR="friendlyelec"
BOARDFAMILY="rk35xx"
BOOTCONFIG="hinlink_rk3528_defconfig"
BOARD_MAINTAINER=""
KERNEL_TARGET="vendor,edge"
KERNEL_TEST_TARGET="vendor,edge"
FULL_DESKTOP="no"
HAS_VIDEO_OUTPUT="no"
BOOT_FDT_FILE="rockchip/rk3528-nanopi-zero2.dtb"
BOOT_SCENARIO="spl-blobs"
IMAGE_PARTITION_TABLE="gpt"
BOOTFS_TYPE="ext4"
BOOTSIZE="512"

function post_family_config__nanopi_zero2_use_mainline_uboot() {
        [[ "${BRANCH}" == "vendor" ]] && return 0
                display_alert "$BOARD" "Mainline U-Boot overrides for $BOARD - $BRANCH" "info"

                # To reuse ATF code in rockchip64_common, let's change the BOOT_SCENARIO and call prepare_boot_configuration() again
                # BOOT_SCENARIO="tpl-blob-atf-mainline"
                # prepare_boot_configuration

                declare -g BOOTCONFIG="generic-rk3528_defconfig"
                declare -g BOOTDELAY=1
                declare -g BOOTSOURCE="https://github.com/u-boot/u-boot.git"
                declare -g BOOTBRANCH="tag:v2026.04-rc3"
                declare -g BOOTPATCHDIR="v2026.04-rc3"
                declare -g BOOTDIR="u-boot-${BOARD}"
                declare -g BOOT_FDT_FILE="rockchip/rk3528-nanopi-zero2.dtb"
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

Please be sure you include all needed drivers in your kernel, for example:

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

**NanoPi Zero2 actual issues:**

Mostly as above for Radxa Rock-2A

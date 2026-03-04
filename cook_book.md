# MT7988 cook book

## Get base BSP

* Use BPI-R4-bsp as base BSP
  https://github.com/BPI-SINOVOIP/BPI-R4-bsp-6.1

## Build u-boot

  * start from the make target
    * SD:   mt7988a_bananapi_bpi-r4-sdmmc_config
    * EMMC: mt7988a_bananapi_bpi-r4-emmc_config

  * CONFIG_DEFAULT_ENV_FILE defines default u-boot env
    ----------------------------------------------------------------------------------------------------------------
    configs/mt7988a_bananapi_bpi-r4-emmc_defconfig
    17:CONFIG_DEFAULT_ENV_FILE="bananapi_bpi-r4_emmc_env"

    configs/mt7988a_bananapi_bpi-r4-sdmmc_defconfig
    17:CONFIG_DEFAULT_ENV_FILE="bananapi_bpi-r4_sdmmc_env"
    ----------------------------------------------------------------------------------------------------------------
  * boot with settings in uEnv.txt from boot partition
    ----------------------------------------------------------------------------------------------------------------
    bananapi_bpi-r4_emmc_env
    22:bootmenu_0d=Run default boot command.=run boot_bpi_default
    34:boot_bpi_default=fatload mmc 0:5 0x43000000 bananapi/bpi-r4/linux-6.1/uEnv.txt ; env import -t  0x43000000 ${filesize} ; run uenvcmd

    bananapi_bpi-r4_sdmmc_env
    20:bootmenu_0d=Run default boot command.=run boot_bpi_default
    31:boot_bpi_default=fatload mmc 0:5 0x43000000 bananapi/bpi-r4/linux-6.1/uEnv.txt ; env import -t  0x43000000 ${filesize} ; run uenvcmd
    ----------------------------------------------------------------------------------------------------------------
  * uEnv.txt provides extra u-boot env for given image to boot with Kernel and FDT
    ----------------------------------------------------------------------------------------------------------------
    bpi=bananapi
    board=bpi-r4
    chip=mt7988a
    service=linux-6.1
    device=mmc
    partition=0:5
    kernel=uImage
    fdt=mt7988a-bananapi-bpi-r4-emmc-snand.dtb
    kaddr=0x46000000
    dtaddr=0x47000000
    root=/dev/mmcblk0p6
    rootopt=rootfstype=ext4 rw rootwait
    console=ttyS0,115200n1 earlyprintk
    bootopts=loglevel=8 initcall_debug=0 androidboot.hardware=mt7988a swiotlb=512 cgroup_enable=memory cgroup_memory=1
    abootargs=setenv bootargs board=${board} console=${console} root=${root} ${rootopt} service=${service} ${bootopts}
    ahello=echo Banana Pi ${board} chip: $chip Service: $service
    aboot=bootm $kaddr - $dtaddr
    aload_fdt=fatload $device $partition $dtaddr ${bpi}/${board}/${service}/dtb/${fdt}
    aload_kernel=fatload $device $partition $kaddr ${bpi}/${board}/${service}/${kernel}
    uenvcmd=run ahello abootargs aload_fdt aload_kernel aboot
    ----------------------------------------------------------------------------------------------------------------

## Build kernel with a different version

### Replace kernel in BSP

* Drop kernel 6.6.104 from Orange OpenWRT build env into BSP as linux-mt/
  linux-mt -> linux-6.6.104

### Patch 6.6.104 kernel for BPI-R4

#### Prepare board boot files

* Check for DTB and kernel image used in BPI-R4 boot

  * uboot ENV: critical config file to customize u-boot to boot the kernel with DTB

    /boot/bananapi/bpi-r4/linux-6.1/uEnv.txt

  * kernel image: build uImage from Image with bpi_script

    arch/arm64/boot/Makefile
    --------------------------------------------------------------------------
    $(obj)/Image: vmlinux FORCE
      $(call if_changed,objcopy)
      $(obj)/mkimage -E -B 0x1000 -p 0x1000 -f $(obj)/bpi_script $(obj)/uImage
    --------------------------------------------------------------------------

    /boot/bananapi/bpi-r4/linux-6.1/uImage

  * DTS:

    # use DTS from new kernel as base : mt7988a-bananapi-bpi-r4.dts
    # and merge in SD(mmc) block from DTS of BSP : mt7988a-bananapi-bpi-r4-sdmmc-snand.dts

    # copy mt7988a-bananapi-bpi-r4.dts to mt7988a-bananapi-bpi-r4-sdmmc-snand.dts (name used in BPI build process)
    cp mt7988a-bananapi-bpi-r4.dts mt7988a-bananapi-bpi-r4-sdmmc-snand.dts
    # merge following blocks from BSP DTS file into it
    vim -O2 mt7988a-bananapi-bpi-r4-sdmmc-snand.dts mt7988a-bananapi-bpi-r4-sdmmc-snand.dts.bsp
    * all lines of #include <dt-bindings/...>
    * &spi0
    * &spi0_nand
    * &uart0
    * &mmc0

    # enable DTB to build in arch/arm64/boot/dts/mediatek/Makefile
    dtb-$(CONFIG_ARCH_MEDIATEK) += mt7988a-bananapi-bpi-r4-sdmmc-snand.dtb

#### Kernel config 
  * Use kernel config from new kernel and add configs to solve build errors
  
  cp linux-mt-6.6.104/.config linux-mt-6.6.104/arch/arm64/configs/mt7988a_bananapi_bpi_r4_defconfig


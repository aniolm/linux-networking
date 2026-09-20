# Guide: Adding a New Target/Hardware in OpenWrt

Adding a new architecture, SoC family, or specific board (target/device) to OpenWrt requires modifying and creating a set of files in the source tree (primarily in `target/linux/` and `target/linux/<arch>/image/`).

This guide walks you through the configuration files and scripts required to successfully add a new device.

## 1. Defining a New Target / Subtarget

If the SoC belongs to an existing family (e.g., MediaTek, Qualcomm Atheros, Rockchip), you modify the existing directory in `target/linux/<target>/`. If you are adding an entirely new architecture, create a new target directory.

### Key Files:

* **`target/linux/<target>/Makefile`**

  Defines basic metadata for the target SoC family (name, CPU architecture, required packages, kernel version).

  ```makefile
  include $(TOPDIR)/rules.mk
  
  ARCH:=arm
  BOARD:=myboard
  BOARDNAME:=My Custom Board Family
  FEATURES:=squashfs ext4 nand
  CPU_TYPE:=cortex-a7
  
  KERNEL_PATCHVER:=6.6
  
  include $(INCLUDE_DIR)/target.mk
  ```

* **`target/linux/<target>/config-<kernel_version>`**

  The main Linux kernel configuration file (`.config`) for the target family. Contains drivers and kernel options shared across all boards on this SoC.

## 2. Hardware Configuration via Device Tree (DTS)

Every modern device in OpenWrt requires a Device Tree file describing controllers, RAM, flash memory, network interfaces, and GPIO pins.

### Key Files:

* **`target/linux/<target>/files/arch/<arch>/boot/dts/<vendor>_<board-name>.dts`**

  Your source Device Tree file. Defines flash storage layout, partition tables, LEDs, and buttons (reset/WPS).

  * **Buttons & LEDs:** Defined under the `gpio-keys` and `gpio-leds` nodes.

  * **Partitions:** Defined under `fixed-partitions` specifying offsets and sizes for `u-boot`, `kernel`, `rootfs`, or combined `firmware`.

## 3. Image & Device Profile Definitions (`image/`)

This directory contains rules for constructing final firmware images (`.bin`, `.img`) for flashing.

### Key Files:

* **`target/linux/<target>/image/Makefile`**

  Defines global image creation rules for the target or subtarget.

* **`target/linux/<target>/image/<subtarget>.mk`** (or directly inside `image/Makefile`)

  Contains device-specific configurations using the `Device/` macro.

  ```makefile
  define Device/vendor_my-board
    DEVICE_VENDOR := Vendor
    DEVICE_MODEL := My Board Name
    DEVICE_DTS := vendor_my-board
    SUPPORTED_DEVICES := vendor,my-board
    IMAGE_SIZE := 16384k
    DEVICE_PACKAGES := kmod-mt7603e kmod-usb2
    IMAGES += sysupgrade.bin
    IMAGE/sysupgrade.bin := append-kernel | pad-to $$$$(BLOCKSIZE) | append-rootfs | pad-rootfs | check-size
  endef
  TARGET_DEVICES += vendor_my-board
  ```

## 4. User-space Scripts & Board Detection (`etc/board.d/`)

To enable OpenWrt to identify the board at boot and automatically configure network interfaces and LEDs, modify scripts in `target/linux/<target>/base-files/etc/board.d/`.

### Key Files:

* **`etc/board.d/01_board_detect`**

  Detects board model based on the compatible string from Device Tree (`/proc/device-tree/compatible`).

  ```sh
  board_config_update
  
  case "$(board_name)" in
  vendor,my-board)
      ucidef_set_board_id "vendor_my-board"
      ;;
  esac
  
  board_config_flush
  ```

* **`etc/board.d/02_network`**

  Defines default network interface setup (WAN, LAN, bridges, VLANs, switch/DSA ports).

  ```sh
  case "$(board_name)" in
  vendor,my-board)
      ucidef_set_interfaces_lan_wan "eth0" "eth1"
      ;;
  esac
  ```

* **`etc/board.d/03_gpio_switches`** or **`01_leds`**

  Maps physical LEDs to system states (e.g., status/power LED, network activity on WAN).

  ```sh
  case "$(board_name)" in
  vendor,my-board)
      ucidef_set_led_netdev "wan" "WAN" "green:wan" "eth1"
      ;;
  esac
  ```

* **`lib/upgrade/platform.sh`**

  Handles the **Sysupgrade** procedure (flashing updates while preserving config). Validates the image format and writes it to the appropriate flash partition.

## 5. Step-by-Step Summary

1. **Hardware Abstraction (DTS):** Create a `.dts` file in `target/linux/<target>/files/arch/<arch>/boot/dts/` defining CPU, RAM, flash partitions, and GPIOs.

2. **Build System (Image Makefile):** Define the hardware profile in `target/linux/<target>/image/Makefile` using `Device/your-board`.

3. **Runtime Logic (board.d):** Update `01_board_detect` and `02_network` so OpenWrt automatically sets up system ID and network defaults during first boot.

4. **Compilation:** Run `make menuconfig`, select your **Target System**, **Subtarget**, and enable your board under **Target Devices**.

### Post-Boot Verification

To confirm board identification works after booting your compiled firmware, run:

```sh
board_name
```

It should return the exact board string configured in `01_board_detect` (e.g., `vendor_my-board`).

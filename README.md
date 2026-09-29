# 🛠️ Reviving a Dead Hi-Link HLK-7628N: Complete Unbricking (breed + openwrt)

[![OpenWrt Version](https://img.shields.io/badge/OpenWrt-25.12.2-blue?logo=openwrt&logoColor=white)](https://openwrt.org)
[![Kernel](https://img.shields.io/badge/Linux_Kernel-6.12.74-brightgreen?logo=linux&logoColor=white)](https://kernel.org)
[![SoC](https://img.shields.io/badge/SoC-MediaTek_MT7628AN-orange)](https://www.mediatek.com)
[![Flash](https://img.shields.io/badge/SPI_Flash-Winbond_W25Q256_(32MB)-red)](#)
[![Status](https://img.shields.io/badge/Status-Fully_Revived-success)](#)

A comprehensive, engineering-grade walkthrough on reviving a hard-bricked **Hi-Link HLK-7628N** embedded module (MediaTek MT7628AN SoC, 128MB DDR2 RAM, 32MB Winbond W25Q256 SPI Flash) from zero to a fully functional modern OpenWrt deployment.

---

## 📑 Table of Contents
- [Hardware Overview](#-hardware-overview)
- [SPI Flash Memory Layout](#-spi-flash-memory-layout)
- [Prerequisites & Bill of Materials](#-prerequisites--bill-of-materials)
- [Step 1: Hardware Connections (CH341A & Pinout)](#step-1-hardware-connections-ch341a--pinout)
- [Step 2: Rescuing the Factory Wi-Fi Calibration (Crucial!)](#step-2-rescuing-the-factory-wi-fi-calibration-crucial)
- [Step 3: Flashing Breed Bootloader & Solving In-Circuit Erase Glitches](#step-3-flashing-breed-bootloader--solving-in-circuit-erase-glitches)
- [Step 4: Serial Console & Booting into Breed](#step-4-serial-console--booting-into-breed)
- [Step 5: Solving the LZMA Overlap Issue & Booting OpenWrt](#step-5-solving-the-lzma-overlap-issue--booting-openwrt)
- [Step 6: Permanent Installation & Verification](#step-6-permanent-installation--verification)
- [Troubleshooting & Lessons Learned](#-troubleshooting--lessons-learned)

---

## 🔬 Hardware Overview

The **HLK-7628N** is an industrial Wi-Fi module powered by the **MediaTek MT7628AN** (MIPS 24KEc @ 580MHz) featuring 128MB RAM and a 32MB SPI Flash (Winbond W25Q256FV).

```text
       +---------------------------------------------------+
       |                  HLK-7628N MODULE                 |
       |                                                   |
       |   +-------------------+   +-------------------+   |
       |   |  MediaTek MT7628  |   |  128MB DDR2 RAM   |   |
       |   |      (MIPS)       |   +-------------------+   |
       |   +-------------------+   +-------------------+   |
       |                           | Winbond W25Q256FV |   |
       |                           |    (32MB Flash)   |   |
       |                           +-------------------+   |
       +---------------------------------------------------+
```

---

## 🗺️ SPI Flash Memory Layout

Understanding the exact memory map prevents destructive overwrites:

```text
0x00000000 ┌─────────────────────────────────────────┐ 0 KB
           │ U-Boot / Breed Bootloader (192 KB)      │
0x00030000 ├─────────────────────────────────────────┤ 192 KB
           │ U-Boot Environment (64 KB)              │
0x00040000 ├─────────────────────────────────────────┤ 256 KB
           │ Factory / EEPROM / Wi-Fi Calib (64 KB)  │  <-- DO NOT OVERWRITE!
0x00050000 ├─────────────────────────────────────────┤ 320 KB
           │ Firmware: OpenWrt Kernel + RootFS       │
           │ (Remaining ~31.6 MB)                    │
0x02000000 └─────────────────────────────────────────┘ 32 MB (End of Flash)
```

---

## 🧰 Prerequisites & Bill of Materials

* **Hardware:**
  * Hi-Link HLK-7628N Module
  * CH341A USB SPI Flash Programmer (with SOIC-8 clip or ZIF socket)
  * USB-to-TTL Serial Adapter (CP2102 / CH340 / FTDI)
  * Ethernet Cable
* **Software Tools (Linux):**
  * `flashrom`, `binwalk`, `python3`, `tio` / `minicom`, `openssh-client`
* **Binaries:**
  * Bootloader: [breed-mt7628-hiwifi-hc5661a.bin](https://breed.hackpascal.net/breed-mt7628-hiwifi-hc5661a.bin)
  * Firmware: [openwrt-25.12.2-ramips-mt76x8-hilink_hlk-7628n-squashfs-sysupgrade.bin](https://downloads.openwrt.org/releases/25.12.2/targets/ramips/mt76x8/openwrt-25.12.2-ramips-mt76x8-hilink_hlk-7628n-squashfs-sysupgrade.bin)

---

## Step 1: Hardware Connections (CH341A & Pinout)

Connect the HLK-7628 spi pins to your CH341A programmer.

![Wiring Diagram](/img/00.png)

Test chip detection on Linux:
```bash
sudo flashrom -p ch341a_spi
```
Expected output:
```text
Found Winbond flash chip "W25Q256FV" (32768 kB, SPI) on ch341a_spi.
```

---

## Step 2: Rescuing the Factory Wi-Fi Calibration (Crucial!)

> [!CAUTION]
> **Never flash a chip without dumping its original content first!**
> The `factory` partition contains unique RF calibration tables, transmit power curves, and MAC addresses. Without this data, the Wi-Fi signal range drops to less than 1 meter.

1. **Dump the full 32MB chip:**
   ```bash
   sudo flashrom -p ch341a_spi -c W25Q256FV -r dump -V
   ```
2. **Inspect the dump with `binwalk`:**
   ```bash
   binwalk dump
   ```
   *Notice the stock kernel at `0x50000` (decimal 327680), proving the original layout is intact.*

3. **Carve out the 64KB `factory.bin` partition:**
   ```bash
   dd if=dump of=factory.bin bs=1k skip=256 count=64
   ```

4. **Verify the calibration data:**
   ```bash
   hexdump -C factory.bin | head -n 4
   ```
   Ensure byte 0 starts with `7628` (MediaTek magic signature) along with original MAC addresses:
   ```text
   00000000  28 76 00 02 00 0c 43 e1  76 30 00 00 00 00 00 00  |(v....C.v0......|
   ```

---

## Step 3: Flashing Breed Bootloader & Solving In-Circuit Erase Glitches

we use **Breed** ("The Immortal Bootloader"), which features a built-in HTTP Web Recovery Console.

### 1. Construct the Padded 32MB Flash Image
`flashrom` strictly requires the payload to match the physical chip size (33,554,432 bytes). Padding with `0xFF` ensures empty sectors are skipped during write:

```bash
# 1. Create a 32MB file filled with 0xFF
python3 -c "open('full_flash.bin', 'wb').write(b'\xff' * 33554432)"

# 2. Inject Breed at offset 0x0
dd if=breed-mt7628-hiwifi-hc5661a.bin of=full_flash.bin conv=notrunc

# 3. Inject the preserved factory.bin at offset 0x40000 (256 KB)
dd if=factory.bin of=full_flash.bin bs=1k seek=256 conv=notrunc
```

### 2. Overcoming the `ERASE FAILED at 0x00002000` In-Circuit Glitch
In-circuit flashing often fails during erase because:
1. The MT7628 SoC draws power from the programmer rail, dropping VCC below the write-inhibit threshold.
2. The SoC tries to boot simultaneously, causing SPI bus contention.

**Fix:** Pull the module's **RST** pin to **GND** to keep the CPU halted.

### 3. Flash the Image
```bash
sudo flashrom -p ch341a_spi -c W25Q256FV -w full_flash.bin
```
Output:
```text
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
```

---

## Step 4: Serial Console & Booting into Breed

Wire your USB-to-TTL converter:
* **HLK-7628N TX0** ➔ **Adapter RX**
* **HLK-7628N RX0** ➔ **Adapter TX**
* **GND** ➔ **GND**

Open the serial terminal at **57600 baud**:
```bash
sudo tio /dev/ttyUSB0 -b 57600
```

Power on the module. The Breed banner will greet you:

```text
Boot and Recovery Environment for Embedded Devices
Copyright (C) 2021 HackPascal <hackpascal@gmail.com>
Version 1.1 (r1337)

DRAM: 128MB
Platform: MediaTek MT7628AN/MT7688AN ver 1, eco 2
Flash: Winbond W25Q256 (32MB) on mt7628-spi.0
rt5350-eth: Using MAC address 00:0c:43:e1:76:29
eth0: MediaTek MT7628 built-in 5-port 10/100M switch

Network started on eth0, inet addr 192.168.1.1, netmask 255.255.255.0
```

![Breed Web Recovery Console — System Information](/img/01.png)
*The Breed Web Recovery Console confirms the platform: MediaTek MT7628AN/MT7688AN ver 1 eco 2, 128MB DDR2, and a Winbond W25Q256 (32MB) flash @ 47MHz.*

Once Breed is reachable at `http://192.168.1.1`, its built-in **Firmware update** page can flash a bootloader, firmware, or EEPROM independently:

![Breed Web Recovery Console — Firmware update](/img/02.png)
*The Firmware update tab. Flash Layout is set to `Public version (0x50000)`, which keeps the Wi-Fi calibration partition intact — exactly what we want.*

---

## Step 5: Solving the LZMA Overlap Issue & Booting OpenWrt

Breed’s built-in `wget` command automatically writes incoming files to `0x80000000`. However, OpenWrt's kernel load address is **also** `0x80000000`. Attempting to boot directly from `0x80000000` causes the LZMA decompressor to overwrite its own input stream, resulting in `Decoding error = 1`.

### The Solution: High-Memory Relocation

1. **Serve the OpenWrt image from your PC:**
   ```bash
   cd ~/Downloads
   python3 -m http.server 8000
   ```
2. **Download into RAM via Breed:**
   ```text
   breed> wget http://192.168.1.10:8000/openwrt-25.12.2-ramips-mt76x8-hilink_hlk-7628n-squashfs-sysupgrade.bin
   ```
   *(File size: `0x5c0111` saved at `0x80000000`)*

3. **Relocate payload to High RAM (`0x82000000`) using `mem copy <dst> <src> <size>`:**
   ```text
   breed> mem copy 0x82000000 0x80000000 0x5c0111
   ```

4. **Verify header at `0x82000000`:**
   ```text
   breed> mem dump 0x82000000 16
   82000000: 56190527 bc9e65e2 1141c469 52302000   '..V.e..i.A.. 0R
   ```

5. **Boot kernel from memory:**
   ```text
   breed> boot mem 0x82000000
   ```
   *The kernel decompresses cleanly from `0x82000000` into `0x80000000`!*

```text
Starting kernel at 0x80000000...
[    0.000000] Linux version 6.12.74
[    0.000000] SoC Type: MediaTek MT7628AN ver:1 eco:2
[    0.000000] MIPS: machine is HILINK HLK-7628N

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 25.12.2, Dave's Guitar
 -----------------------------------------------------
root@OpenWrt:~#
```

---

## Step 6: Permanent Installation & Verification

Now that OpenWrt is running live in RAM, flash it permanently to the SPI chip using `sysupgrade`:

1. **Download image from your PC:**
   ```bash
       wget http://192.168.1.10:8000/openwrt-25.12.2-ramips-mt76x8-hilink_hlk-7628n-squashfs-sysupgrade.bin -P /tmp/
   ```
2. **Execute native flash write:**
   ```bash
   sysupgrade -v -n /tmp/fw.bin
   ```

### Post-Flash Health Check

#### 1. Confirm Network & Factory MAC Addresses
```bash
root@OpenWrt:~# ip link
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 00:0c:43:e1:76:29 brd ff:ff:ff:ff:ff:ff
6: eth0.2@eth0: ...
    link/ether 00:0c:43:e1:76:2a brd ff:ff:ff:ff:ff:ff
```

#### 2. Confirm Wireless Driver
```bash
root@OpenWrt:~# uci show wireless
wireless.radio0=wifi-device
wireless.radio0.type='mac80211'
wireless.radio0.path='platform/10300000.wmac'
```
Enable Wi-Fi:
```bash
uci set wireless.default_radio0.disabled='0'
uci commit wireless
wifi
```

#### 3. Verify MTD Partitions
```bash
root@OpenWrt:~# cat /proc/mtd
dev:    size   erasesize  name
mtd0: 00030000 00010000 "u-boot"
mtd1: 00010000 00010000 "u-boot-env"
mtd2: 00010000 00010000 "factory"
mtd3: 01fb0000 00010000 "firmware"
mtd4: 00203092 00010000 "kernel"
mtd5: 01dacf6e 00010000 "rootfs"
mtd6: 019f0000 00010000 "rootfs_data"
```

#### 4. Confirm the LuCI Web Interface
Reach the router at `http://192.168.1.1/cgi-bin/luci/`:

![OpenWrt LuCI login](/img/06.png)
*Fresh install has no root password yet — log in with username `root` and an empty password, then set one immediately.*

![OpenWrt LuCI status](/img/07.png)
*LuCI Status page reporting the fully revived build: OpenWrt 25.12.2 (r32802-505120278), Kernel 6.12.74, model HILINK HLK-7628N, target ramips/mt76x8, and ~120MB RAM available.*

---

## 💡 Troubleshooting

| Issue / Error | Root Cause | Working Resolution |
| :--- | :--- | :--- |
| `No EEPROM/flash device found` | Poor SOIC-8 clip contact or SoC powering on simultaneously. | Clean pins with alcohol; tie CPU `RST` pin to `GND`. |
| `Image size doesn't match expected size` | `flashrom` requires file size to equal physical capacity. | Create 32MB file padded with `0xFF` using Python before injecting binaries. |
| `ERASE FAILED at 0x00002000` | In-circuit brownout/voltage drop under high erase current. | Halt SoC via `RST` to GND, or desolder chip directly to ZIF socket. |
| `Decoding error = 1` during `boot mem` | LZMA decompressor output address collides with input buffer (`0x80000000`). | Copy compressed image to `0x82000000` before running `boot mem 0x82000000`. |
| `Operation not permitted` with `wget` | OpenWrt default firewall (fw4) blocking outbound client requests. | Use `scp` from host PC to `/tmp/fw.bin` directly. |

---

## 📜 License
Documentation and recovery research provided under the [MIT License](LICENSE).

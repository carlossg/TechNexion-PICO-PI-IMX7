# WiFi Setup — TechNexion PICO-IMX7D

## Hardware

- Board: TechNexion PICO-IMX7D with PI baseboard
- WiFi chip: **AMPAK AP6335** (Broadcom BCM4339) on SDIO bus
  - SDIO ID: `02D0:4335`
- Kernel: 5.15.71 (TechNexion custom build, from `pico-imx7_pico-pi_ubuntu-22.04_qca9377_lcd-800x480_20240426.zip`)

## Root Cause

Two issues must be fixed:

1. **`CONFIG_BRCMFMAC=n`** — TechNexion's custom 5.15.71 kernel has the Broadcom WiFi driver compiled out. Must cross-compile `brcmfmac.ko` from source.

2. **Missing `mmc-pwrseq-simple`** — The QCA-variant DTB (`imx7d-pico-pi-qca.dtb`) doesn't configure the 38.4MHz reference clock (CLKO2) that BCM4339 requires. Without it, the driver loads but the chip hangs with `HT Avail timeout (clkctl 0x50)`. Requires a custom DTB.

---

## Step 1 — Flash the Ubuntu 22.04 image

Get `pico-imx7_pico-pi_ubuntu-22.04_qca9377_lcd-800x480_20240426.zip` from TechNexion.

### Put board in SDP recovery mode

Short the two SDP pins on the PICO module (JP3), then connect USB-C to your Mac.
Verify detection:

```bash
uuu -lsusb
# Should show: SDP: MX7D
```

### Extract and flash

```bash
unzip pico-imx7_pico-pi_ubuntu-22.04_qca9377_lcd-800x480_20240426.zip -d flash \
  "*/imx7/pico-imx7/imx7-SPL" \
  "*/imx7/pico-imx7/imx7-u-boot.img" \
  "*/multiboard/emmc_imx7_img.auto" \
  "*.wic.bz2"

cd flash
sudo uuu -b emmc_imx7_img.auto imx7-SPL imx7-u-boot.img \
  pico-imx7_pico-pi_ubuntu-22.04_qca9377_lcd-800x480_20240426.wic.bz2
```

Remove the SDP jumper and power-cycle. SSH in as `ubuntu` / `ubuntu`.

---

## Step 2 — Cross-compile brcmfmac.ko

Do this on a Mac/Linux host with Docker. The kernel vermagic must match exactly — build from TechNexion's own source.

```bash
mkdir -p brcmfmac-build
cd brcmfmac-build

# Pull kernel config from device
ssh technexion.local "zcat /proc/config.gz" > config-5.15.71

# Enable brcmfmac
sed \
  's/# CONFIG_BRCMFMAC is not set/CONFIG_BRCMFMAC=m\nCONFIG_BRCMFMAC_SDIO=y\nCONFIG_BRCMFMAC_PROTO_BCDC=y\nCONFIG_BRCMFMAC_PROTO_MSGBUF=y/' \
  config-5.15.71 > config-5.15.71-brcmfmac

# Shallow-clone TechNexion 5.15.71 kernel source
git clone --depth=1 --branch tn-imx_5.15.71_2.2.0-stable \
  https://github.com/TechNexion/linux-tn-imx.git \
  linux-tn-imx-5.15

cp config-5.15.71-brcmfmac linux-tn-imx-5.15/.config
```

Build using Docker (bind-mount hits Mac file handle limits; use a Docker volume):

```bash
docker volume create uboot-build   # reuse name if already exists

docker run --rm \
  --ulimit nofile=65536:65536 \
  -v $(pwd)/linux-tn-imx-5.15:/build \
  ubuntu:20.04 bash -c "
    apt-get update -qq &&
    apt-get install -y -qq gcc-arm-linux-gnueabi make bc flex bison libssl-dev libelf-dev python3 &&
    cd /build &&
    make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- olddefconfig &&
    make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- modules_prepare &&
    make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- \
      M=drivers/net/wireless/broadcom/brcm80211 modules
  "
```

Verify:

```bash
ls linux-tn-imx-5.15/drivers/net/wireless/broadcom/brcm80211/brcmfmac/brcmfmac.ko
ls linux-tn-imx-5.15/drivers/net/wireless/broadcom/brcm80211/brcmutil/brcmutil.ko
```

---

## Step 3 — Download firmware files

```bash
# BCM4339 firmware binary (from linux-firmware via Cypress)
curl -fsSL \
  "https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/cypress/cyfmac4339-sdio.bin" \
  -o cyfmac4339-sdio.bin

# BCM4339 nvram calibration file for TechNexion PICO boards
curl -fsSL \
  "https://raw.githubusercontent.com/buildroot/buildroot/master/board/technexion/imx6ulpico/rootfs_overlay/lib/firmware/brcm/brcmfmac4339-sdio.txt" \
  -o brcmfmac4339-sdio.txt
```

---

## Step 4 — Build custom DTB with mmc-pwrseq-simple

The QCA-variant DTB lacks the `mmc-pwrseq-simple` node that provides BCM4339's 38.4MHz reference clock via CLKO2. Create a custom DTS:

```bash
cat > linux-tn-imx-5.15/arch/arm/boot/dts/imx7d-pico-pi-brcm.dts <<'EOF'
/dts-v1/;

#include "imx7d.dtsi"
#include "imx7d-pico-qca.dtsi"
#include "baseboard_pico_pi.dtsi"

/ {
	model = "TechNexion PICO-IMX7D with Broadcom WiFi and PI baseboard";
	compatible = "fsl,pico-imx7d", "fsl,imx7d";

	usdhc2_pwrseq: usdhc2_pwrseq {
		compatible = "mmc-pwrseq-simple";
		clocks = <&clks IMX7D_CLKO2_ROOT_DIV>;
		clock-names = "ext_clock";
	};
};

&clks {
	assigned-clocks = <&clks IMX7D_CLKO2_ROOT_SRC>,
			  <&clks IMX7D_CLKO2_ROOT_DIV>;
	assigned-clock-parents = <&clks IMX7D_CKIL>;
	assigned-clock-rates = <0>, <32768>;
};

&usdhc2 {
	mmc-pwrseq = <&usdhc2_pwrseq>;
};
EOF
```

Compile:

```bash
docker run --rm \
  --ulimit nofile=65536:65536 \
  -v $(pwd)/linux-tn-imx-5.15:/build \
  ubuntu:20.04 bash -c "
    apt-get update -qq &&
    apt-get install -y -qq gcc-arm-linux-gnueabi make bc flex bison libssl-dev device-tree-compiler &&
    cd /build &&
    make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- imx7d-pico-pi-brcm.dtb
  "
```

> **Why `imx7d-pico-qca.dtsi` and not `imx7d-pico.dtsi`?**
> The TechNexion-specific macros (`PICO_PI_GPIO_DEFS()`, `PICO_I2CA`, etc.) that
> `baseboard_pico_pi.dtsi` needs are defined in `imx7d-pico-qca.dtsi` but missing from
> the upstream `imx7d-pico.dtsi`. The mmc-pwrseq node is added back manually above.

---

## Step 5 — Deploy to the board

```bash
BASE=$(pwd)
KER=$BASE/linux-tn-imx-5.15

# Install kernel modules
ssh technexion.local "sudo mkdir -p \
  /lib/modules/5.15.71/kernel/drivers/net/wireless/broadcom/brcm80211/brcmfmac \
  /lib/modules/5.15.71/kernel/drivers/net/wireless/broadcom/brcm80211/brcmutil \
  /lib/firmware/brcm"

scp $KER/drivers/net/wireless/broadcom/brcm80211/brcmfmac/brcmfmac.ko technexion.local:/tmp/
scp $KER/drivers/net/wireless/broadcom/brcm80211/brcmutil/brcmutil.ko  technexion.local:/tmp/
scp $BASE/cyfmac4339-sdio.bin   technexion.local:/tmp/
scp $BASE/brcmfmac4339-sdio.txt technexion.local:/tmp/
scp $KER/arch/arm/boot/dts/imx7d-pico-pi-brcm.dtb technexion.local:/tmp/

ssh technexion.local "
  sudo cp /tmp/brcmfmac.ko \
    /lib/modules/5.15.71/kernel/drivers/net/wireless/broadcom/brcm80211/brcmfmac/
  sudo cp /tmp/brcmutil.ko \
    /lib/modules/5.15.71/kernel/drivers/net/wireless/broadcom/brcm80211/brcmutil/
  sudo cp /tmp/cyfmac4339-sdio.bin  /lib/firmware/brcm/brcmfmac4339-sdio.bin
  sudo cp /tmp/brcmfmac4339-sdio.txt /lib/firmware/brcm/
  sudo depmod -a
"
```

---

## Step 6 — Update boot configuration

U-boot selects the DTB via this logic:
- `wifi_module=qca` → loads `imx7d-pico-pi-qca.dtb`
- anything else → loads `imx7d-pico-pi.dtb`

Copy the custom DTB as `imx7d-pico-pi.dtb` and set `wifi_module=brcm`:

```bash
ssh technexion.local "
  sudo mount /dev/mmcblk2p1 /mnt
  sudo cp /tmp/imx7d-pico-pi-brcm.dtb /mnt/imx7d-pico-pi.dtb
  sudo tee /mnt/uEnv.txt <<'EOF'
baseboard=pi
displayinfo=video=mxcfb0:dev=lcd,800x480@60,if=RGB24,bpp=32
wifi_module=brcm
display_autodetect=off
bootcmd_mmc=run loadimage;run mmcboot;
uenvcmd=run bootcmd_mmc;
EOF
  sudo reboot
"
```

---

## Step 7 — Verify and connect

After reboot:

```bash
ssh technexion.local

# Confirm driver loaded and interface is up
ip link show wlan0
iw dev

# Check dmesg — should see firmware version, no HT Avail timeout
dmesg | grep brcmfmac

# Scan and connect
sudo nmcli dev wifi list
sudo nmcli dev wifi connect "SSID" password "password"
```

Expected dmesg on success:

```
sdhci-esdhc-imx 30b50000.mmc: allocated mmc-pwrseq
brcmfmac: brcmf_fw_alloc_request: using brcm/brcmfmac4339-sdio for chip BCM4339/2
brcmfmac: brcmf_c_preinit_dcmds: Firmware: BCM4339/2 wl0: ... version 6.37.39.113
```

---

## Boot setup reference

| Item | Value |
|---|---|
| Bootloader | U-Boot 2021.04 |
| Boot partition | `/dev/mmcblk2p1` (vfat, mounted at `/mnt`) |
| Boot config | `/mnt/uEnv.txt` |
| Kernel image | `/mnt/zImage` |
| DTB (QCA variant) | `/mnt/imx7d-pico-pi-qca.dtb` |
| DTB (Broadcom/this fix) | `/mnt/imx7d-pico-pi.dtb` |
| Kernel source branch | `tn-imx_5.15.71_2.2.0-stable` |
| Kernel vermagic | `5.15.71 SMP preempt mod_unload modversions ARMv7 p2v8` |

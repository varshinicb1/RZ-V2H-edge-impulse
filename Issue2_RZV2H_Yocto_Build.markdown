# Issue 2: Building and Flashing Yocto Image for Renesas RZ/V2H EVK

I encountered issues setting up the Yocto build environment for the Renesas RZ/V2H EVK to run Edge Impulse projects, including enabling development features and SSH. Below is the step-by-step solution I used to execute the build script, flash the image, and boot the board, based on the Edge Impulse documentation (March 2025) and Renesas AI SDK v5.20.

## Prerequisites
- Ubuntu 20.04 system with ~100GB free space and internet access.
- Files downloaded to ~/edge_impulse: RTK0EF0180F05000SJ_linux-src.zip, nodejs_patches_for_EdgeImpulse_20240805.tar.gz, meta-ei.zip (from Edge Impulse CDN).
- Windows PC for flashing bootloaders (Flash Writer requires Windows).
- MicroSD card (16GB+ recommended) for rootfs.
- Serial cable (MicroUSB to Serial, included with EVK).
- Tera Term (download from https://teratermproject.github.io/) and FTDI VCP driver (from ftdichip.com/drivers/vcp-drivers).
- RZ/V2H EVK with power adapter and DIP switch access.

## Solution

### Step 1: Prepare Working Directory and Files
I started in my working directory ~/edge_impulse, where all required files were downloaded. I executed the setup script in groups to catch errors early, running all commands as my user (no sudo).

**Group 1: Set up tracing, create archive, move files**
```bash
set -x
DIR=`pwd`
mkdir ./archive
mv RTK* ./archive
mv nodejs_patches*gz ./archive
mv meta-ei.zip ./archive
```

**Group 2: Extract nodejs patches**
```bash
cd ./archive
tar xzpf ./nodejs_patches_for_EdgeImpulse_20240805.tar.gz
cd $DIR
```

**Group 3: Extract main archives**
```bash
unzip ./archive/RTK0EF0180F05000SJ_linux-src.zip
tar zxf rzv2h_ai-sdk_yocto_recipe_v5.00.tar.gz
unzip ./archive/meta-ei.zip
cd $DIR
```

**Group 4: Initialize build environment**
```bash
TEMPLATECONF=$DIR/meta-renesas/meta-rzv2h/docs/template/conf/
export MACHINE=rzv2h-evk-ver1
source poky/oe-init-build-env
```

**Group 5: Add Yocto layers**
```bash
cd $DIR/build
bitbake-layers add-layer ../meta-rz-features/meta-rz-graphics
bitbake-layers add-layer ../meta-rz-features/meta-rz-drpai
bitbake-layers add-layer ../meta-rz-features/meta-rz-opencva
bitbake-layers add-layer ../meta-rz-features/meta-rz-codecs
bitbake-layers add-layer ../meta-openembedded/meta-filesystems
bitbake-layers add-layer ../meta-openembedded/meta-networking
bitbake-layers add-layer ../meta-virtualization
bitbake-layers add-layer ../meta-ei
```

**Group 6: Apply tesseract patch**
```bash
patch -p1 < ../0001-tesseract.patch
```

**Group 7: Update nodejs recipe**
```bash
cd ${DIR}/meta-openembedded/meta-oe/recipes-devtools/
tar -zxvf ${DIR}/archive/nodejs_patches_for_EdgeImpulse/nodejs_18.17.1.tar.gz
mv nodejs nodejs_12.22.12
ln -s nodejs_18.17.1 nodejs
cd ${DIR}
```

**Group 8: Update icu recipe**
```bash
cd ${DIR}/poky/meta/recipes-support/
tar -zxvf ${DIR}/archive/nodejs_patches_for_EdgeImpulse/icu_70.1.tar.gz
mv icu icu_66.1
ln -s icu_70.1 icu
cd $DIR/build
```

**Group 9: Append base packages to local.conf**
```bash
echo ""                                     >> ./conf/local.conf
echo "IMAGE_INSTALL_append = \" \\"         >> ./conf/local.conf
echo "    nodejs \\"                        >> ./conf/local.conf
echo "    nodejs-npm \\"                    >> ./conf/local.conf
echo "    \""                               >> ./conf/local.conf
echo ""                                     >> ./conf/local.conf
echo ""                                     >> ./conf/local.conf
echo "IMAGE_INSTALL_append = \" \\"         >> ./conf/local.conf
echo "    nvme-cli \\"                      >> ./conf/local.conf
echo "    sudo \\"                          >> ./conf/local.conf
echo "    curl \\"                          >> ./conf/local.conf
echo "    zlib \\"                          >> ./conf/local.conf
echo "    drpaitvm \\"                      >> ./conf/local.conf
echo "    binutils \\"                      >> ./conf/local.conf
echo "    \""                               >> ./conf/local.conf
echo ""                                     >> ./conf/local.conf
```

**Group 10: Append dev and SSH features**
```bash
echo "WHITELIST_GPL-3.0 += \" cpp gcc gcc-dev mpfr g++ cpp make make-dev binutils libbfd \""             >> ./conf/local.conf
echo "IMAGE_INSTALL_append = \" zlib gcc g++ make cpp packagegroup-core-buildessential \""               >> ./conf/local.conf
echo "IMAGE_INSTALL_append = \" python3 python3-pip python3-core python3-modules \""                     >> ./conf/local.conf
echo "IMAGE_INSTALL_append = \" gstreamer1.0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good \""     >> ./conf/local.conf
echo "IMAGE_INSTALL_append = \" gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly \""                   >> ./conf/local.conf
echo "EXTRA_IMAGE_FEATURES ?= \" debug-tweaks dev-pkgs tools-debug tools-sdk \""                         >> ./conf/local.conf
echo "DISTRO_FEATURES ?= \" usbgadget usbhost wifi opengl \""                                            >> ./conf/local.conf
echo "IMAGE_ROOTFS_EXTRA_SPACE_append_qemuall = \" + 3000000\""                                          >> ./conf/local.conf
```

**Group 11: Upgrade glibc**
```bash
sed -i 's/^CIP_MODE = "Buster"/CIP_MODE = "Bullseye"/g' ./conf/local.conf
```

The EXTRA_IMAGE_FEATURES line includes debug-tweaks (enables SSH with blank root password—secure later with passwd), dev-pkgs, tools-debug, and tools-sdk for development.

### Step 2: Build Yocto Image
The Edge Impulse doc recommends core-image-weston for graphics support. I ran these commands to build the image (takes 4-12+ hours).

**Group 1: Re-initialize build environment**
```bash
set -x
DIR=`pwd`
export TEMPLATECONF=$DIR/meta-renesas/meta-rzv2h/docs/template/conf/
export MACHINE=rzv2h-evk-ver1
source poky/oe-init-build-env
```

**Group 2: Build core-image-weston**
```bash
time bitbake core-image-weston
```

If errors occurred, I checked logs in tmp/work/*/log.*. The output files appeared in build/tmp/deploy/images/rzv2h-evk-ver1.

**Group 3: List output files**
```bash
DIR=`pwd`
ls -al $DIR/build/tmp/deploy/images/rzv2h-evk-ver1
```

Key files: core-image-weston-rzv2h-evk-ver1.wic.bz2 (rootfs image), Flash_Writer_SCIF_RZV2H_DEV_INTERNAL_MEMORY.mot, bl2_bp_spi-rzv2h-evk-ver1.srec, fip-rzv2h-evk-ver1.srec (bootloaders).

### Step 3: Flash Image to RZ/V2H EVK
I followed Renesas AI SDK v5.20 flashing instructions, using a Windows PC for bootloaders and Ubuntu for the rootfs.

1. **Prepare Windows PC**
   - Installed Tera Term and FTDI driver.
   - Copied Flash_Writer*.mot, bl2_bp_spi*.srec, fip*.srec from build/tmp/deploy/images/rzv2h-evk-ver1 to PC (or used pre-built bootloaders from SDK board_setup/xSPI/bootloader if available).

2. **Flash bootloaders to xSPI**
   - Connected PC to EVK CN2 (DEBUG USB) with serial cable.
   - Set DSW1 to Boot mode 3 (SCIF download, per EVK manual).
   - Powered on: 12V to CN13, SW3 ON, SW2 ON.
   - In Tera Term: Serial > COM port > Setup > Terminal: Receive Auto, Transmit CR; Serial: 115200/8/N/1/None, delay 0.
   - Saw "SCI Download mode".
   - File > Send file > Flash_Writer*.mot (text mode). Got ">" prompt.
   - Typed XLS2 > Enter program top addr: 8101e00 > QSPI save addr: 00000 > Sent bl2_bp_spi*.srec (text), confirmed 'y' to clear.
   - Repeated XLS2: program top: 00000 > QSPI save: 60000 > Sent fip*.srec (text), 'y' to clear.
   - Powered off.

3. **Write rootfs to microSD**
   - On Ubuntu: `bunzip2 core-image-weston-rzv2h-evk-ver1.wic.bz2`
   - Used dd: `sudo dd if=core-image-weston-rzv2h-evk-ver1.wic of=/dev/sdX bs=4M conv=fsync status=progress` (replaced sdX with SD device, checked with lsblk).
   - Alternatively, used balenaEtcher on Windows for GUI flashing.

4. **Boot the board**
   - Inserted SD into EVK SD2 slot.
   - Set DSW1 to Boot mode 2 (xSPI boot).
   - Reconnected serial, powered on (SW3 ON, SW2 ON).
   - In Tera Term, pressed Enter during boot to enter U-Boot.
   - Ran: `env default -a; saveenv; boot`
   - Board booted to login: root (no password).
   - Shutdown: `poweroff`

### Step 4: Post-Boot Setup
I accessed the board via serial (screen /dev/ttyUSB0 115200 on Ubuntu) or SSH (root@<board-ip>, no password). To install the Edge Impulse CLI:
```bash
npm install -g edge-impulse-linux
edge-impulse-linux
```
Followed login prompts to connect to my Edge Impulse project.

### Step 5: Deploy Model
In the Edge Impulse dashboard, I selected Renesas RZ/V2H as the target, chose DRP-AI TVM optimization, and deployed the model. On the board, I ran inferences:
```bash
edge-impulse-linux-runner --model-file <model-file> --device-id <device-id>
```
Added --camera for camera input or --hw-accel for DRP-AI acceleration if needed. Verified output in the console.

## Notes
- If boot fails, check serial logs or DSW1 settings.
- For eMMC instead of SD, use Renesas guide’s eMMC flashing steps.
- Blank root password (from debug-tweaks) is insecure; set one with `passwd` on the board.
- Build errors may stem from missing files or network issues; verify paths and connectivity.
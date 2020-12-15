

## [Precondition] Set up a dev environment

### Build server

My laptop isn't the best machine for building.
Will set up a build server on Digital ocean.

Pros:
* faster build speed
* faster package download

Cons:
* need to download the artifacts before flashing
    * this isn't an issue if the image built is small and compressed image

#### Setup

In the DO interface spin up a droplet with an extra volume for potential yocto cache.

```
ssh root@206.81.23.110

# Yocto isn't happy running as root
# Create a user

adduser user
usermod -aG sudo user
# Copy authorised keys to user from root
# chown the .ssh dir so the user can access it
# exit server

# Locally create an ssh config ~/.ssh/config
Host build_server
    User user
    HostName 206.81.23.110
    ForwardAgent yes

# This allows us to push to restricted repos
# If forwarding agent not working [ref for debugging](https://docs.github.com/en/free-pro-team@latest/developers/overview/using-ssh-agent-forwarding)

ssh build_server

# Prepare yocto cache directories
echo "export DRIVE_PATH=/mnt/volume_fra1_01" >> ~/.bashrc
echo "export YOCTO_CACHE=\$DRIVE_PATH/yocto_cache" >> ~/.bashrc
source ~/.bashrc
sudo mkdir $YOCTO_CACHE
sudo chmod 777 $YOCTO_CACHE

# Start tmux so if ssh dies bitbake can continue cooking
tmux

# There is where we start the development
```


###  [Precondition] Get a board which boots

There were two choices here:
* https://github.com/jumpnow/meta-wandboard
    * Depends on a lot of layers, more complicated setup
* https://github.com/Freescale/fsl-community-bsp-platform
    * Less layers
    * Seems to be used with some other boards with Zeus


#### Set up a minimal board build which can be quickl

```
mkdir mender-nxp
cd mender-nxp

curl https://storage.googleapis.com/git-repo-downloads/repo > repo
chmod 755 repo
export PATH=$(pwd):${PATH}

repo init -u https://github.com/Freescale/fsl-community-bsp-platform -b dunfell
repo sync

export DISTRO=poky
export MACHINE=wandboard

. ./setup-environment build

cat > conf/local.conf << 'EOF'
MACHINE ??= 'wandboard'
DISTRO ?= 'poky'
PACKAGE_CLASSES ?= 'package_rpm'
EXTRA_IMAGE_FEATURES ?= "debug-tweaks"
USER_CLASSES ?= "buildstats image-mklibs image-prelink"
PATCHRESOLVE = "noop"
BB_DISKMON_DIRS ??= "\
    STOPTASKS,${TMPDIR},1G,100K \
    STOPTASKS,${DL_DIR},1G,100K \
    STOPTASKS,${SSTATE_DIR},1G,100K \
    STOPTASKS,/tmp,100M,100K \
    ABORT,${TMPDIR},100M,1K \
    ABORT,${DL_DIR},100M,1K \
    ABORT,${SSTATE_DIR},100M,1K \
    ABORT,/tmp,10M,1K"
PACKAGECONFIG_append_pn-qemu-system-native = " sdl"
CONF_VERSION = "1"

ACCEPT_FSL_EULA = "1"
DL_DIR="/mnt/volume_fra1_01/yocto_cache/downloads"
SSTATE_DIR="/mnt/volume_fra1_01/yocto_cache/sstate"
EOF


bitbake core-image-minimal
```

Local flashing:

```
scp build_server:/home/user/mender-nxp/build/tmp/deploy/images/wandboard/core-image-minimal-wandboard.wic.gz image.wic.gz
gunzip -c image.wic.gz > image                                       
                                                                     
# Plug in the sdcard reader                               
umount /dev/sdb*                              
sudo dd if=image of=/dev/sdb bs=4M                              
sync                                                              
```


## Mender attempts

Mender community has a method of supporting multiple boards through one repo.
The system is each board family provides a repo manifest with the required layers.
In addition it has required custom recipes and .append files which the `setup-environment` appends on top of board agnostic `meta-mender-community/templates/local.conf.append`.

I've skipped that and took a route to manually create a layer for board.

#### Local flashing

New flashing steps with the sdimg.

```
scp build_server:/home/user/mender-nxp/build/tmp/deploy/images/wandboard/core-image-minimal-wandboard.sdimg.bz2 image.bz2
bzip2 -d -c image.bz2 > image

# Put the sd card
umount /dev/sdb*
sudo dd if=image of=/dev/sdb bs=4M
sync
```

### Auto patching

The starting point for this is the setup for the board without mender mentioned above.

I'm hoping to get mender auto patch do most of the work for my forked u-boot.


#### Steps


```
# Do the build above

cd ~/mender-nxp/sources
git clone -b wandboard git@github.com:TheMeaningfulEngineer/meta-mender-community.git
git clone -b dunfell https://github.com/mendersoftware/meta-mender.git

cd ../build


cat > conf/bblayers.conf << 'EOF'
LCONF_VERSION = "7"

BBPATH = "${TOPDIR}"
BSPDIR := "${@os.path.abspath(os.path.dirname(d.getVar('FILE', True)) + '/../..')}"

BBFILES ?= ""
BBLAYERS = " \
  ${BSPDIR}/sources/poky/meta \
  ${BSPDIR}/sources/poky/meta-poky \
  \
  ${BSPDIR}/sources/meta-openembedded/meta-oe \
  ${BSPDIR}/sources/meta-openembedded/meta-multimedia \
  ${BSPDIR}/sources/meta-openembedded/meta-python \
  \
  ${BSPDIR}/sources/meta-freescale \
  ${BSPDIR}/sources/meta-freescale-3rdparty \
  ${BSPDIR}/sources/meta-freescale-distro \
  ${BSPDIR}/sources/meta-mender/meta-mender-core \
  ${BSPDIR}/sources/meta-mender/meta-mender-demo \
"
EOF

# Make a layer to work with to make appends to the forked u-boot recipe
bitbake-layers create-layer meta_mender_wandboard
bitbake-layers add-layer meta_mender_wandboard
mkdir -p meta_mender_wandboard/recipes-bsp/u-boot/

cat > meta_mender_wandboard/recipes-bsp/u-boot/u-boot-fslc_%.bbappend << EOF
require recipes-bsp/u-boot/u-boot-mender.inc
EOF
 
cat > conf/local.conf << 'EOF'
MACHINE ??= 'wandboard'
DISTRO ?= 'poky'
PACKAGE_CLASSES ?= 'package_rpm'
EXTRA_IMAGE_FEATURES ?= "debug-tweaks"
USER_CLASSES ?= "buildstats image-mklibs image-prelink"
PATCHRESOLVE = "noop"
BB_DISKMON_DIRS ??= "\
    STOPTASKS,${TMPDIR},1G,100K \
    STOPTASKS,${DL_DIR},1G,100K \
    STOPTASKS,${SSTATE_DIR},1G,100K \
    STOPTASKS,/tmp,100M,100K \
    ABORT,${TMPDIR},100M,1K \
    ABORT,${DL_DIR},100M,1K \
    ABORT,${SSTATE_DIR},100M,1K \
    ABORT,/tmp,10M,1K"
PACKAGECONFIG_append_pn-qemu-system-native = " sdl"
CONF_VERSION = "1"

ACCEPT_FSL_EULA = "1"
DL_DIR="/mnt/volume_fra1_01/yocto_cache/downloads"
SSTATE_DIR="/mnt/volume_fra1_01/yocto_cache/sstate"

# Disble mender systemd so I can use the sysv images to be quicker
MENDER_FEATURES_DISABLE_append = " mender-systemd  mender-grub mender-image-uefi"
# mender-install isn't recognized as a MENDER_FEATURES_ENABLE
MENDER_FEATURES_ENABLE_append = " mender-image mender-uboot mender-image-sd"


# Got this from uarting on the device 
MENDER_STORAGE_DEVICE = "/dev/mmcblk2"
MENDER_STORAGE_TOTAL_SIZE_MB = "256"

MENDER_UBOOT_AUTO_CONFIGURE = "1"

MENDER_ARTIFACT_NAME="wandboard_mender"
EOF

bitbake core-image-minimal
```

This fails to build with the error:

```
CONFIG_ENV_SIZE=0x2000
but Mender expects:
CONFIG_ENV_SIZE=0x20000
Please fix U-Boot's configuration file.
```

I need to create a patch in my header file and add it to the recipe.

```
# it will live here
mkdir -p meta_mender_wandboard/recipes-bsp/u-boot/files

# bitbake -c devshell u-boot
# in the new terminal
# create a patch with git
# end result 

cat > meta_mender_wandboard/recipes-bsp/u-boot/0001-Add-env-size-in-header.patch << 'EOF'
From eaab945eb93edbc4b35ff8f345740746c4b95fb0 Mon Sep 17 00:00:00 2001
From: Your Name <you@example.com>
Date: Tue, 15 Dec 2020 19:47:47 +0000
Subject: [PATCH] Add env size in header

---
 include/configs/wandboard.h | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

diff --git a/include/configs/wandboard.h b/include/configs/wandboard.h
index a65d23bbe8..6c15456589 100644
--- a/include/configs/wandboard.h
+++ b/include/configs/wandboard.h
@@ -124,7 +124,7 @@
        (CONFIG_SYS_INIT_RAM_ADDR + CONFIG_SYS_INIT_SP_OFFSET)

 /* Environment organization */
-
+#define CONFIG_ENV_SIZE 0x20000
 #define CONFIG_SYS_MMC_ENV_DEV         0

 #endif                        /* __CONFIG_H * */
--
2.25.1
EOF
```

```
cat > meta_mender_wandboard/recipes-bsp/u-boot/u-boot-fslc_%.bbappend << EOF
require recipes-bsp/u-boot/u-boot-mender.inc
BOOTENV_SIZE = "0x20000"
SRC_URI_append = "file://001-Add-env-size-in-header.patch"
EOF

bitbake core-image-minimal
# flash
```

Builds but doesn't create a bootable board.


Adding to local.conf also

```
MENDER_IMAGE_BOOTLOADER_BOOTSECTOR_OFFSET = "2"
MENDER_IMAGE_BOOTLOADER_FILE = "u-boot.img"
```

Adding to local.conf also

```
MENDER_IMAGE_BOOTLOADER_BOOTSECTOR_OFFSET = "2"
MENDER_IMAGE_BOOTLOADER_FILE = "SPL"
```

Makes the board boot but loops in the SPL.
I need to make SPL and u-boot.img a single binary and than pass that file to MENDER_IMAGE_BOOTLOADER_FILE = "SPL".

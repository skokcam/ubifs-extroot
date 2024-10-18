# Howto use ubifs on usb disk as extroot on OpenWrt 23.05 installed on TD-W9980

I like additional functionality OpenWrt brings so I have decided to install the latest stable (23.05 as of now) OpenWrt on my TD-W9980. Unfortunately additional functionality requires additional configuration space and after installing OpenWrt my Router is left with a 512KiB rootfs-data partition, which over half (~ 260 KiB) was already in use. Fortunately there is a solution: external root. For extroot, OpenWrt documentation [[1]](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration) states that I have to install "[block-mount](https://openwrt.org/packages/pkgdata/block-mount), [kmod-fs-ext4](https://openwrt.org/packages/pkgdata/kmod-fs-ext4), [e2fsprogs](https://openwrt.org/packages/pkgdata/e2fsprogs), [parted](https://openwrt.org/packages/pkgdata/parted)" packages and I actually needed an extra package, which is "[kmod-usb-storage](https://openwrt.org/packages/pkgdata/kmod-usb-storage)", which are impossible to fit on leftover space on rootfs-data (~ 250Kib). 

Actually there is a solution for this in [OpenWrt extroot documentation](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration#custom_image)) Custom Image creation: get rid of some functionality in rootfs (squashfs) image and install what you need. But I didn't want to create a custom image and still wanted to use the extroot.
I cant install ext4 (too big), reiserfs, jfs neither did fit. So I am left with filesystems already installed in OpenWrt stock image which are jffs2 and ubifs. I tried jffs2 first, but block-mount doesn't like jffs2 as extroot so all I got was an error message that says jffs2 is not supported as extroot, so ubifs it is.  

Too bad both jffs2 and ubifs partitions need to be on mtd to be mounted. The solution: dimunitive "[kmod-block2mtd](https://openwrt.org/packages/pkgdata/kmod-block2mtd)" package which allows disk partitions to be used as mtd partition by linux.

How it works:
- Prepare a 512 MiB partition on usb flash disk (I resized the first partition on a usb flash disk and created a 512MiB sized second partition -> sda2)
  - no fdisk installed on OpenWrt yet, so do this step on another computer
- Configure OpenWrt box to connect to internet (we need OpenWrt repos)
- SSH into OpenWrt (W9980) and install required packages (block-mount, block2mtd, kmod-usb-storage)
  - opkg update
  - opkg install block-mount block2mtd kmod-usb-storage
- Rename /sbin/block to /sbin/block.bin then install [this](block.sh) script at /sbin/block, then make this script executable
  - mv /sbin/block /sbin/block.bin
  - wget https://raw.githubusercontent.com/skokcam/ubifs-extroot/main/block.sh -O /sbin/block
  - chmod +x /sbin/block
- Emulate prepared usb flash partition (/dev/sda2 in my case) as mtd [[2](https://wiki.emacinc.com/wiki/Mounting_JFFS2_Images_on_a_Linux_PC)]  
  - modprobe block2mtd
  - echo "/dev/sda2,64KiB" > /sys/module/block2mtd/parameters/block2mtd
- Format the new emulated mtd partition (mtd7 in my case) as ubifs and mount it at /mnt [[3](https://bootlin.com/blog/creating-flashing-ubi-ubifs-images/)]
  - ubiformat /dev/mtd7
  - ubiattach -p /dev/mtd7
  - ubimkvol /dev/ubi0 -N my_extroot -s 510MiB
  - mount -t ubifs ubi0_0 /mnt
- Copy contents of overlay to the newly created ubifs volume mounted at /mnt
  - cp -a /overlay /mnt
- Set new ubifs partition (/dev/ubi0_0 in my case) as extroot [[1](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)]
  - eval $(block info | grep /dev/ubi0_0 | grep -o -e 'UUID="\S*"')
  - eval $(block info | grep -o -e 'MOUNT="\S*/overlay"')
  - uci -q delete fstab.extroot
  - uci set fstab.extroot="mount"
  - uci set fstab.extroot.uuid="${UUID}"
  - uci set fstab.extroot.target="${MOUNT}"
  - uci commit fstab'
- Set original overlay device to be mounted at /rwm  [[1](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration)]
  - DEVICE="$(block info | sed -n -e '/MOUNT="\S*\/overlay"/s/:\s.*$//p')"
  - uci -q delete fstab.rwm
  - uci set fstab.rwm="mount"
  - uci set fstab.rwm.device="${DEVICE}"
  - uci set fstab.rwm.target="/rwm"
  - uci commit fstab
- Rebooot the router and hopefully everything works after restart
  - reboot
 

OpenWrt doesn't allow these steps to be completed as is, but thankfully there is a way for [mounting luks formatted partitions as extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration#luks_encrypted_extroot) in documentation: rename then replace "[block-mount](https://openwrt.org/packages/pkgdata/block-mount)" package's block utility with a script. Armed with this information, a few (ok, dozens) tries later I had a [working script](block.sh).

Notice this is not a catchall script like the original luks-mount script, this script expects ubi container at */dev/sda2* and new mtd partition as */dev/mtd7*. Of course it is possible to write a cathall script:

TODO:
- enumerate block partition (preferably parsing output of block) and find ubi containers
- mount found ubi containers as mtd (block2mtd)
- mount ubifs partitions in emulated mtd partitions

Sources:
- 1 https://openwrt.org/docs/guide-user/additional-software/extroot_configuration
- 2 https://wiki.emacinc.com/wiki/Mounting_JFFS2_Images_on_a_Linux_PC
- 3 https://bootlin.com/blog/creating-flashing-ubi-ubifs-images/

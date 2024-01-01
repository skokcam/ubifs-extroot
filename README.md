# Howto use ubifs on usb disk as extroot on OpenWrt 23.05 installed on TD-W9980

I like additional functionality OpenWrt brings so I have decided to install the latest stable (23.05 as of now) OpenWrt on my TD-W9980. Unfortunately additional functionality requires additional configuration space and after installing OpenWrt my Router is left with a 512KiB rootfs-data partition, which over half (~ 260 KiB) was already in use. Fortunately there is a solution: external root. For extroot, OpenWrt documentation [[1]](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration) states that I have to install "[block-mount](https://openwrt.org/packages/pkgdata/block-mount), [kmod-fs-ext4](https://openwrt.org/packages/pkgdata/kmod-fs-ext4), [e2fsprogs](https://openwrt.org/packages/pkgdata/e2fsprogs), [parted](https://openwrt.org/packages/pkgdata/parted)" packages and I actually needed an extra package, which is "[kmod-usb-storage](https://openwrt.org/packages/pkgdata/kmod-usb-storage)", which are impossible to fit on leftover space on rootfs-data (~ 250Kib). 

Actually there is a solution for this in [OpenWrt extroot documentation](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration#custom_image)) Custom Image creation: get rid of some functionality in rootfs (squashfs) image and install what you need. But I didn't want to create a custom image and still wanted to use the extroot.
I cant install ext4 (too big), reiserfs, jfs neither did fit. So I am left with filesystems already installed in OpenWrt stock image which are jffs2 and ubifs. I tried jffs2 first, but block-mount doesn't like jffs2 as extroot so all I got was an error message that says jffs2 is not supported as extroot, so ubifs it is.  

Too bad both jffs2 and ubifs partitions need to be on mtd to be mounted. The solution: dimunitive "[kmod-block2mtd](https://openwrt.org/packages/pkgdata/kmod-block2mtd)" package which allows disk partitions to be used as mtd partition by linux.

So here is my game plan:
- Prepare a 512 Mib partition on usb flash disk
- Mount prepared partition as mtd [[2](https://wiki.emacinc.com/wiki/Mounting_JFFS2_Images_on_a_Linux_PC)]
- Format the new mtd partition (mtd7) as ubifs[[3](https://bootlin.com/blog/creating-flashing-ubi-ubifs-images/)]
- Use new ubifs partition as extroot

OpenWrt doesn't allow these steps to be completed as is, but thankfully there is a way for [mounting luks formatted partitions as extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration#luks_encrypted_extroot) in documentation: rename then replace "[block-mount](https://openwrt.org/packages/pkgdata/block-mount)" package's block utility with a script. Armed with this information, a few (ok, dozens) tries later I had a [working script](block).

Notice this is not a catchall script like the original luks-mount script, this script expects ubi container at */dev/sda2* and new mtd partition as */dev/mtd7*. Of course it is possible to write a cathall script:

TODO:
- enumerate block partition (preferably parsing output of block) and find ubi containers
- mount found ubi containers as mtd (block2mtd)
- mount ubifs partitions in emulated mtd partitions

Sources:
- 1 https://openwrt.org/docs/guide-user/additional-software/extroot_configuration
- 2 https://wiki.emacinc.com/wiki/Mounting_JFFS2_Images_on_a_Linux_PC
- 3 https://bootlin.com/blog/creating-flashing-ubi-ubifs-images/

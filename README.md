# Howto use ubifs on usb disk as extroot on OpenWrt 23.05 installed on TD-W9980

I like additional functionality OpenWrt brings so I have decided to install the latest stable (23.05 as of now) OpenWrt on my TD-W9980. Unfortunately additional functionality requires additional configuration space and after installing OpenWrt my Router is left with a 512KiB rootfs-data partition, which over half (~ 260 KiB) was already in use. Fortunately there is a solution: external root[1]. For extroot OpenWrt documentation[1] states that I have to install "block-mount kmod-fs-ext4 e2fsprogs parted" packages and I actually needed an extra package, which is "kmod-usb-storage", which are impossible to fit on leftover space on rootfs-data (~ 250Kib). Actually there is a solution for this in OpenWrt documentation[1] Custom Image creation: get rid of some functionality in rootfs (squashfs) image and install what you need. But I didn't want to create a custom image and still wanted to use the extroot.
I cant install ext4 (too big), reiserfs, jfs neither did fit. So I am left with filesystems already installed in OpenWrt stock image which are jffs2 and ubifs. I tried jffs2 first, but block-mount doesn't like jffs2 as extroot so all I got was an error message that says jffs2 is not supported as extroot. So ubifs it is.  
Too bad both jffs2 and ubifs partitions need to be on mtd to be mounted. The solution: dimunitive "kmod-block2mtd" package which allows disk partitions to be used as mtd partition by linux.

So here is my game plan:
- Prepare a 512 Mib partition on usb flash disk
- Mount prepared partition as mtd [2]
- Format the new mtd partition (mtd7) as ubifs[3]
- Use new ubifs partition as extroot

OpenWrt doesn't allow these steps to be completed as is but thankfully there is a way for mounting luks formatted partitions as extroot in documentation [1]: replace "block-mount" package's block utility with a script. A few (ok, dozens) tries later I had a working script.

Notice this is not a catchall script like the original luks-mount script, this script expects ubi container at /dev/sda2 and new mtd partition as /dev/mtd7. Of course it is possible to write a cathcall script:

TODO:
- enumerate block partition (preferably parsing output of block) and find ubi containers
- mount found ubi containers as mtd (block2mtd)
- mount ubifs partitions in emulated mtd partitions

Sources:
- 1 https://openwrt.org/docs/guide-user/additional-software/extroot_configuration
- 2 https://wiki.emacinc.com/wiki/Mounting_JFFS2_Images_on_a_Linux_PC
- 3 https://bootlin.com/blog/creating-flashing-ubi-ubifs-images/

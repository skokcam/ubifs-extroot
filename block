!/bin/sh

# Set to 1 to enable debug logs
export DEBUG=

SDIR=${0%/*}
BLOCK="${SDIR}/block.bin"
LD_LIBRARY_PATH=${LD_LIBRARY_PATH:-.}
LD_LIBRARY_PATH="${SDIR}/../usr/lib:${LD_LIBRARY_PATH}"
PATH=$PATH:${SDIR}:${SDIR}/../usr/sbin:${SDIR}/../usr/bin

block() {
  ( exec -a ${0} ${BLOCK} "$@" )
}

if [ "$1" = "extroot" ]; then
  # Load modules needed for mtd emulation
  KVER=$(uname -r)
  insmod ${SDIR}/../lib/modules/${KVER}/block2mtd.ko

  for i in $(seq 1 20); do
    [ -e /dev/sda2 ] && continue
    sleep 1
    # Hotplug runs too late, create device nodes for /dev/sd*, if there are any
    for SYSDEVPATH in /sys/class/block/sd*; do                                 
      [ ! -f "$SYSDEVPATH"/dev ] && continue                                   
      [ -e "/dev/${SYSDEVPATH##*/}" ] && continue                              
      MAJMIN=$(cat "$SYSDEVPATH"/dev | tr ':' ' ')                             
      mknod /dev/${SYSDEVPATH##*/} b $MAJMIN                                   
    done  
  done
  [ -e /dev/sda2 ] && echo block: sda2 found > /dev/kmsg

  # Try emulating sda2 as mtd7
  echo "/dev/sda2,64KiB" > /sys/module/block2mtd/parameters/block2mtd
  # Try attach
  /usr/sbin/ubiattach -m 7 > /dev/kmsg 2>&1
  # Hotplug runs too late, create device nodes for /dev/ubi*, if there are any
  for SYSDEVPATH in /sys/class/ubi/ubi*; do                                 
    [ ! -f "$SYSDEVPATH"/dev ] && continue                                   
    [ -e "/dev/${SYSDEVPATH##*/}" ] && continue                              
    MAJMIN=$(cat "$SYSDEVPATH"/dev | tr ':' ' ')                             
    mknod /dev/${SYSDEVPATH##*/} c $MAJMIN                                   
  done 
  sleep 5
  echo "Block list" > /dev/kmsg
  block info > /dev/kmsg 2>&1
fi

block "$@"


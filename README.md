# PMFS2: Persistent-memory file-system in user-space via ZUFS

## License: GPL-2

## Overview
PMFS2 is a file system for persistent memory in user-space via ZUFS.
The file system is optimized to be lightweight and efficient in providing
access to persistent memory that is directly accessible via CPU load/store
instructions. It manages the persistent memory directly and avoids the block
driver layer and page cache layer and thus provides synchronous reads and
writes to persistent area. It supports the standard POSIX style file system
APIs so that the applications need not be modified to use this file system.

Current implementation is a proof-of-concept and not meant to be used in
production environment. Most of all, it demonstrates the full potential of the
ZUFS framework for developing high performance PMEM-based file-systems in
user-space. See [performance](#Performance) section below.

The work on user-space PMFS2 is derived from now discontinued work of in-kernel
PMFS file-system. Namely, using a radix-tree for file's mapping.
See [pmfs on github](https://github.com/linux-pmfs/pmfs) for more details.

## How to

### Setup (fedora)
- OS: Fedora32 (VM)
- Memory size: 10G
- Storage size: 20G
- CPUs: 8


```
  $ uname -srpm
  Linux 5.7.8-200.fc32.x86_64 x86_64 x86_64
```

### Preparations (as root):
Install extra packages (some are optional):
```
  $ dnf install -y kernel-devel kernel-headers
  $ dnf groupinstall -y "Development Tools"
  $ dnf install -y hmaccalc zlib-devel binutils-devel elfutils-libelf-devel
  $ dnf install -y libunwind libunwind-devel libuuid libuuid-devel systemd-devel
  $ dnf install -y fio
  ...
```
Disable SE linux (optional):
```
  $ setenforce 0
```


### Emulate pmem device (4G)
```
  $ grubby --update-kernel $(grubby --default-kernel) --args="memmap=4G\!6G"
  $ reboot
  ...
  $ lsblk -n /dev/pmem0
  pmem0 259:0    0   4G  0 disk
```

### Build ZUFS & PMFS2 (as normal user):
Create ZUFS source code development environment:

```
  $ mkdir -p zufs
  $ cd zufs
  $ git clone --branch upstream-5.4 https://github.com/NetApp/zufs-zuf ./zuf
  $ git clone --branch upstream https://github.com/NetApp/zufs-zus ./zus
  $ git clone --branch upstream https://github.com/sagimnl/pmfs2 ./zus/fs/pmfs2
```

Build zuf kernel module:

```
  $ cp ./zus/fs/pmfs2/linux5-zuf-module.mk ./zuf/fs/zuf/module.mk
  $ cd zuf
  $ make -f fs/zuf/module.mk KDIR=/usr/src/kernels/$(uname -r)
  $ ls fs/zuf/*.ko
  $ cd ..
```

Build zus & pmfs2:

```
  $ cd zus
  $ cp fs/pmfs2/zus-pmfs2-config .config
  $ make
  ...
  $ cd ..
```

### Running ZUFS with PMFS2 (as root):
Start **zusd** service:

```
  $ export ZUF_ROOT=/sys/fs/zuf
  $ export ZUFS_HOME=$(pwd)
  $ export LD_LIBRARY_PATH=${ZUFS_HOME}/zus:${ZUFS_HOME}/zus/fs/pmfs2/
  $ export ZUFS_LIBFS_LIST=pmfs2

  $ insmod ${ZUFS_HOME}/zuf/fs/zuf/zuf.ko
  $ mount -v -t zuf nodev ${ZUF_ROOT}
  $ ${ZUFS_HOME}/zus/fs/pmfs2/mkfs/mkfs.pmfs2 /dev/pmem0
  $ ${ZUFS_HOME}/zus/zusd &
```

Mount and test **pmfs2**:

```
  $ mkdir -p /mnt/pmfs2
  $ mount -t pmfs2 /dev/pmem0 /mnt/pmfs2
  $ mount | grep pmfs2
  /dev/pmem0 on /mnt/pmfs2 type pmfs2 (rw,relatime)

  $ echo "hello, world" > /mnt/pmfs2/hello
  $ ls /mnt/pmfs2/
  hello
  $ cat /mnt/pmfs2/hello
  hello, world
  $ rm -f /mnt/pmfs2/hello
```

## Performace
PMFS2 demonstates that user-space file-systems can achive high-bandwidth and
low-latency I/O. This is done via **ZUFS** framework. On an 8-cores VM you
should expect staggering performance:

```
  $ fio --numjobs=1 --bs=4K --directory=/mnt/pmfs2 --name=pmfs2 \
        --size=64M --fallocate=none -rw=readwrite --rwmixread=70 \
	--ioengine=psync --sync=1 --direct=1 --time_based       \
	--runtime=20 --thinktime=0 --norandommap --group_reporting
  ...
  READ: bw=5174MiB/s (5425MB/s), 5174MiB/s-5174MiB/s
  WRITE: bw=2217MiB/s (2324MB/s), 2217MiB/s-2217MiB/s
  ...

```

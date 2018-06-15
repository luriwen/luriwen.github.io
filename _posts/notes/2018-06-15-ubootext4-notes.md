---
layout: post
title: uboot下无法识别ext4文件系统目录中的文件
date: 2018-06-15 11:36:00 +0800
category: 学习记录
---

出现这个问题是做了一个u盘安装系统，格式为ext4。在uboot下加载相关文件时，发现相应的目录为空。但是通过光盘安装好的系统，同样是ext4格式，在uboot下是能识别到目录中的文件的。于是便研究了出现这种问题的原因。

### 一、u盘安装盘制作

1. 查看u盘盘符

   `fdisk -l`

   查看，u盘是哪个设备。这里为/dev/sdc

   ```shell
   Disk /dev/sdc：29.1 GiB，31229607936 字节，60995328 个扇区
   单元：扇区 / 1 * 512 = 512 字节
   扇区大小(逻辑/物理)：512 字节 / 512 字节
   I/O 大小(最小/最佳)：512 字节 / 512 字节
   磁盘标签类型：dos
   磁盘标识符：0xa3693efd
   
   设备       启动  起点     末尾     扇区  大小 Id 类型
   /dev/sdc1        2048 60995327 60993280 29.1G 83 Linux
   ```

2. 卸载u盘

   `umount /dev/sdc*`

3. 清空与新建u盘分区

   `fdisk /dev/sdc`

4. 格式化u盘

   格式化为**ext4**格式，可用tune2fs查看ext4文件系统参数。

   `mkfs.ext4 /dev/sdc1`

5. 挂载ISO

   `mount -o loop xxx.iso /cdrom`

6. 挂载u盘

   `mount /dev/sdc1 /mnt`

7. 将ISO中的内容同步到u盘中

   `rsync -a /cdrom/ /mnt/`

8. 卸载设备

   `umount /mnt`

   `umount /cdrom`

### 二、ext4ls显示目录为空

使用tune2fs查看u盘ext4文件系统参数，具体各参数用途可自行查阅资料。

```shell
# tune2fs -l /dev/sdc1

tune2fs 1.44.1 (24-Mar-2018)
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          cb53f633-9b59-4888-824c-e4fd9aa56a01
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize metadata_csum
Filesystem flags:         signed_directory_hash 
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              1908736
Block count:              7624160
Reserved block count:     381208
Free blocks:              7460305
Free inodes:              1908725
First block:              0
Block size:               4096
Fragment size:            4096
Group descriptor size:    64
Reserved GDT blocks:      1024
Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         8192
Inode blocks per group:   512
Flex block group size:    16
Filesystem created:       Fri Jun 15 12:01:07 2018
Last mount time:          Fri Jun 15 14:11:03 2018
Last write time:          Fri Jun 15 14:11:03 2018
Mount count:              1
Maximum mount count:      -1
Last checked:             Fri Jun 15 12:01:07 2018
Check interval:           0 (<none>)
Lifetime writes:          132 MB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:	              256
Required extra isize:     32
Desired extra isize:      32
Journal inode:            8
Default directory hash:   half_md4
Directory Hash Seed:      f052d6b4-4a2e-47c0-bd41-d862252bceaf
Journal backup:           inode blocks
Checksum type:            crc32c
Checksum:                 0x94d72b26
```

在uboot下通过ext4ls查看u盘中的文件。

```shell
# ext4ls usb 0:1
<DIR>       4096 .
<DIR>       4096 ..
<DIR>      16384 lost+found
<SYM>          1 ubuntu
<DIR>          0 .disk
<DIR>          0 EFI
<DIR>          0 boot
<DIR>          0 casper
<DIR>       4096 dists
<DIR>          0 install
<DIR>       4096 isolinux
<DIR>          0 pics
<DIR>       4096 pool
<DIR>      12288 preseed
             231 README.diskdefines
           24055 md5sum.txt
           
# ext4ls usb 0:1 casper
```

由上可看到，u盘根目录中的文件可显示出来，而目录中的文件无法识别，显示为空。

使用的uboot版本为：U-Boot 2013.10

经分析，此问题时因为这版uboot不支持64位ext4文件系统。通过上面`tune2fs -l /dev/sdc1`显示的u盘ext4文件系统参数**Filesystem features**中有**64bit特性**，表示该u盘为64位ext4文件系统。去掉该**64bit**特性，改为32位文件系统，即可在uboot下识别到目录中的文件。

### 三、修改ext4文件系统

将64位ext4文件系统改为32位比较常用的方法列举如下：

1. 使用mkfs.ext4格式化时指定不使用64bit特性。使用**-O**选项可去除或者增加相应特性，在特性**64bit**前加 **^** 表示去除该特性；

   `mkfs.ext4 -O ^64bit /dev/sdc1`

2. mkfs.ext4实际会调用mke2fs，运行该命令格式化文件系统时默认使用的配置文件为**/etc/mke2fs.conf**，内容如下：

   ```c
   [defaults]
           base_features = sparse_super,large_file,filetype,resize_inode,dir_index,ext_attr
           default_mntopts = acl,user_xattr
           enable_periodic_fsck = 0
           blocksize = 4096
           inode_size = 256
           inode_ratio = 16384
   
   [fs_types]
           ext3 = {
                   features = has_journal
           }
           ext4 = {
                   features = has_journal,extent,huge_file,flex_bg,metadata_csum,64bit,dir_nlink,extra_isize
                   inode_size = 256
           }
           small = {
                   blocksize = 1024
                   inode_size = 128
                   inode_ratio = 4096
           }
           floppy = {
                   blocksize = 1024
                   inode_size = 128
                   inode_ratio = 8192
           }
           big = {
                   inode_ratio = 32768
           }
           huge = {
                   inode_ratio = 65536
           }
           news = {
                   inode_ratio = 4096
           }
           largefile = {
                   inode_ratio = 1048576
                   blocksize = -1
           }
           largefile4 = {
                   inode_ratio = 4194304
                   blocksize = -1
           }
           hurd = {
                blocksize = 4096
                inode_size = 128
           }
   ```

   修改该配置文件，将ext4文件类型的features中的**64bit**去除，再使用mkfs.ext4格式化时即可生成32位的ext4文件系统。

3. 使用tune2fs命令将64位ext4文件系统修改位32位，不需要再对u盘重新格式化，其中的内容也得到保留。有时还需运行e2fsck磁盘修复命令，对磁盘进行修复。

   ```shell
   # tune2fs -O ^64bit /dev/sdc1
   tune2fs 1.44.1 (24-Mar-2018)
   
   Please run e2fsck -f on the filesystem.
   
   在运行 e2fsck 后，请运行“resize2fs -s /dev/sdc1”来禁用 64 位模式。
   
   # e2fsck -f /dev/sdc1
   e2fsck 1.44.1 (24-Mar-2018)
   第 1 步：检查inode、块和大小
   第 2 步：检查目录结构
   第 3 步：检查目录连接性
   第 4 步：检查引用计数
   第 5 步：检查组概要信息
   /dev/sdc1：694/1908736 文件（0.6% 为非连续的）， 632702/7624160 块
   
   # resize2fs -s /dev/sdc1
   resize2fs 1.44.1 (24-Mar-2018)
   将文件系统转换为 32 位。
   /dev/sdc1 上的文件系统现在为 7624160 个块（每块 4k）。
   ```

使用tune2fs查看ext4文件系统参数：

```shell
# tune2fs -l /dev/sdc1

tune2fs 1.44.1 (24-Mar-2018)
Filesystem volume name:   <none>
Last mounted on:          <not available>
Filesystem UUID:          cb53f633-9b59-4888-824c-e4fd9aa56a01
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent flex_bg sparse_super large_file huge_file dir_nlink extra_isize metadata_csum
Filesystem flags:         signed_directory_hash 
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              1908736
Block count:              7624160
Reserved block count:     381208
Free blocks:              6991480
Free inodes:              1908042
First block:              0
Block size:               4096
Fragment size:            4096
Reserved GDT blocks:      1024
Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         8192
Inode blocks per group:   512
Flex block group size:    16
Filesystem created:       Fri Jun 15 12:01:07 2018
Last mount time:          Fri Jun 15 15:34:56 2018
Last write time:          Fri Jun 15 15:37:09 2018
Mount count:              0
Maximum mount count:      -1
Last checked:             Fri Jun 15 15:36:34 2018
Check interval:           0 (<none>)
Lifetime writes:          137 MB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:	              256
Required extra isize:     32
Desired extra isize:      32
Journal inode:            8
Default directory hash:   half_md4
Directory Hash Seed:      f052d6b4-4a2e-47c0-bd41-d862252bceaf
Journal backup:           inode blocks
Checksum type:            crc32c
Checksum:                 0xd1d353b2
```

由上可看到，在**Filesystem features**中没有了**64bit**特性。

在uboot中使用ext4ls查看u盘中的文件，能正常显示目录下的文件。

```shell
# ext4ls usb 0:1
<DIR>       4096 .
<DIR>       4096 ..
<DIR>      16384 lost+found
<SYM>          1 ubuntu
<DIR>       4096 .disk
<DIR>       4096 EFI
<DIR>       4096 boot
<DIR>       4096 casper
<DIR>       4096 dists
<DIR>       4096 install
<DIR>      12288 isolinux
<DIR>       4096 pics
<DIR>       4096 pool
<DIR>       4096 preseed
             231 README.diskdefines
           24055 md5sum.txt
           
# ext4ls usb 0:1 casper
<DIR>       4096 .
<DIR>       4096 ..
            4201 filesystem.manifest-remove
           54520 filesystem.manifest
        36915149 initrd.lz
            2905 filesystem.manifest-minimal-remove
              11 filesystem.size
      1831378944 filesystem.squashfs
             916 filesystem.squashfs.gpg
         8249080 vmlinuz
```

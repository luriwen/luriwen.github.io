---
layout: post
title: xenomai在ubuntu上的移植及运行
date: 2018-06-15 16:36:00 +0800
category: 学习记录
---

### 一、ubuntu上安装xenomai

1. 下载linux内核

   使用的内核版本为4.9.51。

   `https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/`

2. 下载ipipe

   下载对应内核版本的ipipe。

   `http://xenomai.org/downloads/ipipe/v4.x/x86/`

3. 下载xenomai

   `http://xenomai.org/downloads/xenomai/stable/latest/`

4. 打ipipe和xenomai补丁

   在xenomai目录下：

   `./scripts/prepare-kernel.sh --arch=x86_64 --adeos=../ipipe-core-4.9.51-x86-5.patch --linux=../linux-4.9.51`

   ```shell
   --arch 系统架构；
   --adeos ipipe补丁路径；
   --linux 内核源码路径
   ```

5. 配置内核

   ```shell
   CONFIG_APM [=n]
   CONFIG_ACPI_PROCESSOR [=n]
   CONFIG_CPU_FREQ [=n]
   CONFIG_CPU_IDLE [=n]
   CONFIG_INPUT_PCSPKR [=n]
   ```

6. 编译内核

   **编译内核时，可能会出现一些报错**。

   - __COBALT_CALL32_SYSNR报错

     ```c
     arch/x86/xenomai/include/asm/xenomai/syscall.h: In function ‘__xn_get_syscall_nr’:
     arch/x86/xenomai/include/asm/xenomai/syscall.h:46:31: error: implicit declaration of function ‘__COBALT_CALL32_SYSNR’; did you mean ‘__COBALT_COMPAT_BIT’? [-Werror=implicit-function-declaration]
     #define __xn_syscall(regs)    __COBALT_CALL32_SYSNR(__xn_reg_sys(regs) \
     ```

     修改arch/x86/xenomai/Kconfig文件，去除以下两行：

     ```c
     config XENO_ARCH_SYS3264
     	def_bool IA32_EMULATION
     ```

   - debug重新申明报错

     ```c
     ./arch/x86/include/asm/traps.h:13:17: error: ‘debug’ redeclared as different kind of symbol
     asmlinkage void debug(void);
     drivers/xenomai/net/drivers/eepro100.c:55:12: note: previous definition of ‘debug’ was here
     static int debug = -1; /* The debug level */
     ```

     drivers/xenomai/net/drivers/目录下的网络相关.c文件编译时，可能报debug重新申明错误，将其改名字即可。

     

   **编译命令**

   `sudo make-kpkg --initrd kernel-image kernel-headers`

7. 安装内核

   ```shell
   sudo dpkg -i linux-image-4.9.51-ipipe_4.9.51-ipipe-10.00.Custom_amd64.deb
   sudo dpkg -i linux-headers-4.9.51-ipipe_4.9.51-ipipe-10.00.Custom_amd64.deb
   ```

8. 启动xenomai内核

   从grub选择xenomai内核启动。

   ```c
   dmesg | grep Xenomai
   # [    1.417024] [Xenomai] scheduling class idle registered.
   # [    1.417025] [Xenomai] scheduling class rt registered.
   # [    1.417045] [Xenomai] disabling automatic C1E state promotion on Intel processor
   # [    1.417055] [Xenomai] SMI-enabled chipset found, but SMI workaround disabled
   # [    1.417088] I-pipe: head domain Xenomai registered.
   # [    1.417704] [Xenomai] allowing access to group 1234
   # [    1.417726] [Xenomai] Cobalt v3.0.5 (Sisyphus's Boulder) [DEBUG]
   ```

9. RTnet

  RTnet的实现参考Linux网络子系统的层次化结构 。子系统主要包括数据包管理、UDP/IP 协议栈 、RT驱动 、Socket API接口 、RTcap、RTcfg、RTmac等模块。RTnet通过把Linux内核的UDP/IP协议栈移植到Xenomai系统，且以静态地址解析协议取代动态地址解析协议，以确定的方式实现UDP/IP、 ICMP和ARP协议栈，为实时用户进程和内核模块提供 POSIX 兼容的 Socket API接口。

  ![rtnet_protocol]({{site.baseurl}}/assets/img/notes/20180615-rtnet.img/rtnet_protocol.png)

  

  为了避免以太网上的冲突和堵塞，RTnet还提供一个附加的媒体接入控制层RTmac。通过VNIC虚拟网卡技术为TCP/IP等非实时性任务提供服务。rtnet网络模型如下图所示。

  ![rtnet]({{site.baseurl}}/assets/img/notes/20180615-rtnet.img/rtnet.png)

10. rtnet.conf配置

  rtnet.conf配置文件默认位于`/usr/xenomai/etc/rtnet.conf`。一些主要改动如下：

  - RT_DRIVER="rt_r8169" 

    使用的实时网络驱动。

  - REBIND_RT_NICS="0000:03:00.0"

    PCI网络设备的NIC地址。可通过使用`lshw -C network`以及查看`bus info: pci@`字段来获得相关的NIC地址。

  - IPADDR="192.168.140.22"

    主设备的IP地址。

  - NETMASK=”255.255.0.0”

    掩码。

  - TDMA_SLAVES="192.168.140.23 192.168.140.24"

    从设备的IP地址，用空格隔开。

    

11. 系统实时网络测试

   在运行rtnet之前，需要卸载相应网络设备的非实时驱动。然后加载实时网络接口设备的相关驱动，即运行rtnet命令。

   `rmmod r8169 rt_r8169 rtnet`

   实时网络接口设备相关的驱动加载之后，实时网络开始工作，在相同的测试条件下进行实时网络实时性测试。网络传输数据为72字节，发送10个数据包。

   - 在经过 Linux 内核实时性改造,但不具有实时网络的系统中测试，结果如下图所示。其中最小的往返延时(min)为 0.041 ms，平均往返延时(avg)为 0.046ms，最大往返延时(max)为0.052ms。

   ![ping1]({{site.baseurl}}/assets/img/notes/20180615-rtnet.img/ping1.png)

   

   - 在经过 Linux 内核实时性改造,且具有实时网络的系统中，使用虚拟网卡VNIC进行测试。结果如下图所示。其中最小的往返延时(min)为 0.672 ms，平均往返延时(avg)为 0.881ms，最大往返延时(max)为1.102ms。

   ![ping]({{site.baseurl}}/assets/img/notes/20180615-rtnet.img/ping.png)

   

   - 在经过 Linux 内核实时性改造,且具有实时网络的系统中测试,结果如图所示。其中最差的往返延时(worst case rtt)为 1065.4μs。


   ![rtping]({{site.baseurl}}/assets/img/notes/20180615-rtnet.img/rtping.png)

   

   

12. 出现的问题

   #####运行rtnet报错

   ```shell
   cd /usr/xenomai/sbin
   sudo ./rtnet --cf ../etc/rtnet.conf start
   ```

   ```c
   sh: echo: I/O error
   ioctl: No such device
   ioctl: No such device
   ioctl: No such device
   ioctl: No such device
   ioctl (add): No such device
   ioctl (add): No such device
   ioctl (add): No such device
   ioctl (add): No such device
   vnic0: 获取接口标志时出错: 没有那个设备
   SIOCSIFADDR: 没有那个设备
   vnic0: 获取接口标志时出错: 没有那个设备
   SIOCSIFNETMASK: 没有那个设备
   Waiting for all slaves...ioctl: No such device
   ioctl: No such device
   ```

   机器使用的网卡型号为RTL8168，xenomai对应的驱动没有将8168网卡型号加进去。

   `vim drivers/xenomai/net/drivers/r8169.c`

   修改`rtl8169_pci_tbl`，加入8168网卡设备id：

   ```c
   static struct pci_device_id rtl8169_pci_tbl[] = {
           { PCI_DEVICE(PCI_VENDOR_ID_REALTEK,     0x8136), 0, 0, 2 },
           { PCI_DEVICE(PCI_VENDOR_ID_REALTEK,     0x8167), 0, 0, 1 },
           { PCI_DEVICE(PCI_VENDOR_ID_REALTEK,     0x8168), 0, 0, 2 },
           { PCI_DEVICE(PCI_VENDOR_ID_REALTEK,     0x8169), 0, 0, 1 },
           { PCI_DEVICE(PCI_VENDOR_ID_DLINK,       0x4300), 0, 0, 1 },     /* <kk> D-Link DGE-528T */
           {0,},
   };
   ```

   

   #####DMA报错

   运行`rtnet start`后，一直刷SW-IOMMU空间不足错误。

   ```c
   DMA: Out of SW-IOMMU space for 60 bytes at device 0000:03:00.0
   DMA: Out of SW-IOMMU space for 60 bytes at device 0000:03:00.0
   DMA: Out of SW-IOMMU space for 60 bytes at device 0000:03:00.0
   ```

   在内存大于4GB的系统中，能正常工作的芯片只有 **RealTek 8139**，**Intel PRO/1000 PCI-E (e1000e, NOT e1000)**，**Intel 82575 (igb)**。其他驱动的变通方案，在内核启动时在启动参数中将内存指定为4G，即`mem=4096M`。经验证，在内存大小为4G的系统中，对于**RTL8111/8168/8411**芯片也需要在内核启动时指定内存大小为4G。

   

   另外一个问题，看了几天后没解决。在运行rtnet start后一直刷下面的信息。

   ```shell
   TDMA: Failed to transmit sync frame!
   TDMA: Failed to transmit sync frame!
   TDMA: Failed to transmit sync frame!
   ```

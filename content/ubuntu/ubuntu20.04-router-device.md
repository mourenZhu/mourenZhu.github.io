---
title: '将Ubuntu20.04变为一个路由器'
date: 2025-06-06T10:16:50+08:00
draft: false
tags: ["linux", "ubuntu", "router"]
author: ["zhumouren"]
---

# 前言
本文记录的是作者手头上的设备搭建情况，不同的芯片架构、系统版本搭建情况可能有所不同。以下是设备情况介绍
- 芯片架构: aarch64
- Linux内核版本: 4.19.232
- 发行版: Ubuntu20.04

# 设备路由设置
## 修改内核参数
```shell
vi /etc/sysctl.conf
```
```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.all.accept_ra = 2
```

## 修改物理网口名称
在ubuntu中用`ifconfig`查看网口信息有可能有`enP1p17s0`这种网口，想要修改为规律的`ethx`可以按下面的方式做
```shell
udevadm info -a -p /sys/class/net/eth0 # 可以通过这个命令查看网口属性以编写下面的配置
```
---
```shell
vi /etc/udev/rules.d/70-persistent-net.rules
```
```
SUBSYSTEM=="net", ACTION=="add", SUBSYSTEMS=="platform", KERNELS=="fe2a0000.ethernet", NAME="eth0"
SUBSYSTEM=="net", ACTION=="add", SUBSYSTEMS=="platform", KERNELS=="fe010000.ethernet", NAME="eth1"
SUBSYSTEM=="net", ACTION=="add", SUBSYSTEMS=="pci", KERNELS=="0001:11:00.0", NAME="eth2"
SUBSYSTEM=="net", ACTION=="add", SUBSYSTEMS=="pci", KERNELS=="0002:21:00.0", NAME="eth3"
SUBSYSTEM=="net", ACTION=="add", SUBSYSTEMS=="pci", KERNELS=="0000:01:00.0", NAME="eth4"
```
这里可以关注SUBSYSTEMS有platform、pci两种值但实际还有更多，看具体以太网口的属性。KERNELS代表设备硬件位置，可以通过`udevadm`命令来查看。

## 设置lan口、wan口
以下设置为eth0为wan0口，其他为lan口。

```shell
vi /etc/network/interfaces.d/wan
```

```
auto wan0
iface wan0 inet dhcp
bridge_ports eth0
metric 30
```
---
```shell
vi /etc/network/interfaces.d/lan
```
```
auto lan
iface lan inet static
bridge_ports eth1 eth2 eth3 eth4
address 192.168.1.1
netmask 255.255.255.0
```

## 监听wan口拔插
在wan口插入网线时需要自动启动dhcp clinet获取ip地址并设置route，在wan口拔出网线时需要清空wan口的ip并更新route。默认情况下ubuntu并不会做这些事情

```shell
vi /etc/netplug/netplugd.conf
```
```
wan*
```

## 设置DHCP服务器
在这里我们要使用dnsmasq来作设备的dns服务和dhcp服务，所以要关闭ubuntu自带的systemd-resolved（dns解析）服务，并设置dnsmasq配置文件。
```shell
# 先下载dnsmasq
sudo apt install dnsmasq
```
---
```shell
# 关闭 systemd-resolved dns解析
vi /etc/systemd/resolved.conf
```
```
[Resolve]
DNSStubListener=no
```
---
```shell
# 配置dnsmasq dhcp server
vi /etc/dnsmasq.d/dhcp.conf
```
```
dhcp-range=set:lan,192.168.1.2,192.168.1.254,86400
dhcp-option=tag:lan,1,255.255.255.0
dhcp-option=tag:lan,3,192.168.1.1
dhcp-option=tag:lan,6,223.5.5.5,8.8.8.8
```

## DHCP其他问题

```shell
vi /etc/dhcp/dhclient.conf
```
```
timeout 20;
retry 60;
```
在作者手上的设备如果不做以上修改会有一个问题，在设备启动时如果wan口不插入网线会导致开机等待5分钟，这是由于`/etc/network/interfaces.d/wan`文件中设置的`auto wan0`以及`iface wan0 inet dhcp`会在开机时开机wan0口并尝试通过dhcp client获取IP地址，但是由于wan0口没有插网线如果dhcpclient默认超时时间过长的话，会导致开机时间也会变的非常长，也会间接导致如果dhcp client超时时间大于`/lib/systemd/system/networking.service`文件中的配置`TimeoutStartSec=5min`5分钟会让后面`/etc/network/interfaces.d/*`的脚本无法启动。所以要将`timeout`调小  
还需将`retry`显示设置，让dhcpcclient没有获取ip地址时主动每隔一段时间重试。

# overlayroot
这里说的`overlayroot`是一个软件包，这个软件包可以很方便的让系统开启OverlayFS。  
```shell
sudo apt install overlayroot

vi /etc/overlayroot.conf
```
```
overlayroot="tmpfs" # 或者/dev/mmcblk0p3等实际分区
```
如果你的设备是x86架构，那恭喜你按上述方式做就能立马生效，但是如果是arm架构uboot启动，那就有点麻烦了。

需要安装`sudo apt install initramfs-tools`，并用`update-initramfs -c -k $(uname -r)`生成`initrd.img-4.19.232`文件（不同内核后缀不一样）(默认生成在`/boot/`目录下)。然后将`initrd.img`复制到能让uboot读到的分区，设置uboot参数告知内核在指定位置读取`initramfs`到`RAM`中。

大致的启动流程是`uboot`会将内核和`initramfs映像`加载到内存中并启动内核。内核会检查`initramfs`是否存在，如果找到，则将其挂载为`/`并运行`/init`。在`/init`中会使用`overlayfs`的方式第二次挂载`rootfs`。

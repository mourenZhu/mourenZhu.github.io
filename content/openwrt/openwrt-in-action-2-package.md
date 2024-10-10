---
title: 'OpenWrt开发入门实战(二)创建软件包'
date: 2024-10-10T16:10:50+08:00
draft: false
tags: ["openwrt"]
author: ["zhumouren"]
---

# 1. 创建一个简单的helloworld程序
软件开发中的一种良好做法是将各个部分分开，并且只提供足够的连接点，以便软件组合的不同部分可以协同运行。这种方法称为关注点分离。

为了保持这种方法，我们将应用程序的源代码放在一个独立的目录中，与 OpenWrt 构建系统分开。 引用至OpenWrt官方文档-- https://openwrt.org/docs/guide-developer/helloworld/chapter2

根据上一章的内容，进入openwrt编译环境容器，执行以下命令
``` shell
cd /home/buildbot 
mkdir helloopenwrt 
cd helloopenwrt
```
创建并进入helloopenwrt目录后创建一个helloopenwrt.c
```shell
touch helloopenwrt.c
```
选择用vi或者其他编辑器将以下内容复制进helloopenwrt.c
```c
#include <stdio.h>
 
int main(void)
{
    printf("\nHello, OpenWrt!\n\n");
    return 0;
}
```

编译helloopenwrt
```shell
gcc -o helloopenwrt helloopenwrt.c -Wall
```

可以执行helloopenwrt看控制台是否由打印Hello, OpenWrt!
```shell
./helloopenwrt
```

# 2. 创建软件包
OpenWrt 构建系统主要围绕软件包的概念。它们是系统的支柱。无论什么软件，几乎总有一个软件包。这适用于系统中的几乎所有东西，无论是独立于目标的工具、交叉编译工具链、目标固件的 Linux 内核、与内核捆绑的附加模块，还是将安装到目标固件的根文件系统上的各种应用程序。

由于这种面向包的性质，对"helloopenwrt"应用程序采用相同的方法也是合乎逻辑的。

对于不属于构建系统本身组成部分的软件包，主要的交付系统是软件包源。它只是一个软件包存储库，其中包含最终固件中可能包含的软件包。存储库可以位于本地目录中，也可以位于网络共享中，也可以位于 GitHub 等版本控制系统中。创建和维护软件包源使我们能够通过将与软件包相关的文件与示例应用程序的源代码分开来保持关注点分离。

出于本文的目的，我们在本地目录中创建一个新的包存储库。此存储库的名称为"mypackages"，它包含一个名为"examples"的类别。在此类别中，只有一个条目，即我们的"helloopenwrt"应用程序。 引用至OpenWrt官方文档-- https://openwrt.org/docs/guide-developer/helloworld/chapter3

## 2.1 创建本地包源

```shell
cd /home/buildbot
mkdir -p mypackages/examples/helloopenwrt
```

## 2.2 创建包清单文件package manifest file
OpenWrt 构建系统中的每个软件包都由软件包清单文件描述。包清单文件负责描述软件包及其功能，并且必须至少提供有关从何处获取源代码、如何构建源代码以及最终可安装软件包中应包含哪些文件的说明。软件包清单还可能包含可选配置脚本的选项，指定软件包之间的依赖关系等。

为了使我们的应用程序的源代码成为一个包，并成为我们之前创建的包存储库的一部分，我们需要为其创建一个包清单文件：
```shell
cd mypackages/examples/helloopenwrt
touch Makefile
```

选择vi或者其他文本编辑器将以下内容写入Makefile，请注意，文件中有较短和较长的空白缩进。较短的是简单空格字符，而较长的是硬制表符（Tab），本章相关的几个文件是由OpenWrt自己的GUN Make编译的，而`GUN Make的缩进不接受空格`。

```dockerfile
include $(TOPDIR)/rules.mk

# Name, version and release number
# The name and version of your package are used to define the variable to point to the build directory of your package: $(PKG_BUILD_DIR)
PKG_NAME:=helloopenwrt
PKG_VERSION:=1.0
PKG_RELEASE:=1

# Source settings (i.e. where to find the source codes)
# This is a custom variable, used below
SOURCE_DIR:=/home/buildbot/helloopenwrt

include $(INCLUDE_DIR)/package.mk

# Package definition; instructs on how and where our package will appear in the overall configuration menu ('make menuconfig')
define Package/helloopenwrt
  SECTION:=examples
  CATEGORY:=Examples
  TITLE:=Hello, OpenWrt!
endef

# Package description; a more verbose description on what our package does
define Package/helloopenwrt/description
  A simple "Hello, OpenWrt!" -application.
endef

# Package preparation instructions; create the build directory and copy the source code. 
# The last command is necessary to ensure our preparation instructions remain compatible with the patching system.
define Build/Prepare
		mkdir -p $(PKG_BUILD_DIR)
		cp $(SOURCE_DIR)/* $(PKG_BUILD_DIR)
		$(Build/Patch)
endef

# Package build instructions; invoke the target-specific compiler to first compile the source file, and then to link the file into the final executable
define Build/Compile
		$(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/helloopenwrt.o -c $(PKG_BUILD_DIR)/helloopenwrt.c
		$(TARGET_CC) $(TARGET_LDFLAGS) -o $(PKG_BUILD_DIR)/$1 $(PKG_BUILD_DIR)/helloopenwrt.o
endef

# Package install instructions; create a directory inside the package to hold our executable, and then copy the executable we built previously into the folder
define Package/helloopenwrt/install
		$(INSTALL_DIR) $(1)/usr/bin
		$(INSTALL_BIN) $(PKG_BUILD_DIR)/helloopenwrt $(1)/usr/bin
endef

# This command is always the last, it uses the definitions and variables we give above in order to get the job done
$(eval $(call BuildPackage,helloopenwrt))
```

# 3. 将软件包加入OpenWrt构建系统

## 3.1 包源配置
OpenWrt 构建系统使用一个名为`feeds.conf`的特定文件，该文件将在固件配置阶段提供的包源。为了使包含应用程序的包可见，必须将新的包源包含到此文件中。

默认情况下，OpenWrt源代码目录中不存在此文件，因此需要创建它：
```shell
cd /home/buildbot/openwrt
touch feeds.conf
```
将以下内容写进`feeds.conf`
```
src-link mypackages /home/buildbot/mypackages
```

## 3.2 更新包源
```shell
cd /home/buildbot/openwrt
./scripts/feeds update mypackages
./scripts/feeds install -a -p mypackages
```
如果一切正常，你将会看到以下内容，说明我们已经将helloopenwrt包加入构建系统了。
```
Installing package 'helloopenwrt' from mypackages
```

# 4. 构建、部署、测试helloopenwrt

## 4.1 构建软件包
构建软件包之前我们需要将我们的软件包添加到目标固件的配置当中，然后再构建软件包。  

1. 在openwrt目录下执行`make menuconfig`进入图形化配置
2. 进入图形化配置后，选择并进入`Examples`子菜单 -> 选中此菜单下的`helloopenwrt`条目 -> 单击`Y`键将此包添加到配置中
3. 最后按`Esc`键退出并保存  

执行完以上步骤后运行以下命令，即可编译软件包
```shell
make package/helloopenwrt/compile
```
如果一切正常我们就会在目录`/home/buildbot/openwrt/bin/packages/x86_64/mypackages`下找到一个新文件`helloopenwrt_1.0-1_x86_64.ipk`

## 4.2 部署软件包
1. 将上述文件`helloopenwrt_1.0-1_x86_64.ipk`下载到第一章创建的`openwrt-example/openwrt-23.05-study-demo`目录下
2. 使用ssh或者docker cp的方式将软件包传输至上一章创建的docker openwrt容器中。  
	2.1 ssh方式自行查找  
	2.2 docker cp方式，首先进入`openwrt-example/openwrt-23.05-study-demo`目录
	```shell
	docker ps #找到docker openwrt容器ID 以下是输出

	CONTAINER ID   IMAGE                                      COMMAND               CREATED        STATUS       PORTS                                          NAMES
	d86c80f3e38d   openwrt-example-openwrt-23.05-study-demo   "/sbin/init"          26 hours ago   Up 2 hours   0.0.0.0:11122->22/tcp, 0.0.0.0:11180->80/tcp   openwrt-example-openwrt-23.05-study-demo-1
	4427fa5e92b1   docker-linux-env-ubuntu-compile-openwrt    "/usr/sbin/sshd -D"   27 hours ago   Up 2 hours   0.0.0.0:2211->22/tcp                           docker-linux-env-ubuntu-compile-openwrt-1

	docker cp ./helloopenwrt_1.0-1_x86_64.ipk d86c80f3e38d:/tmp/ # docker cp 本地文件路径 ID全称:容器路径
	```

## 4.3 测试软件包
先进入上一章创建的docker openwrt容器内部，建议使用docker exec进入容器。
```shell
docker exec -it d86c80f3e38d sh # 容器ID自行修改为自己的实际值
```

### 4.3.1 安装软件包
使用opkg安装软件包
```shell
opkg install /tmp/helloopenwrt_1.0-1_x86_64.ipk
```
不出意外会看到以下内容，说明软件包已成功安装
```
Installing helloopenwrt (1.0-1) to root...
Configuring helloopenwrt.
```

### 4.3.2 测试软件包
安装完成后可执行helloopenwrt，直接输入`helloopenwrt`就好了，无需`./helloopenwrt`
```shell
helloopenwrt
```
不出意外的话可以看到屏幕打印`Hello, OpenWrt!`

### 4.3.3 删除软件包
同样使用opkg删除软件包
```shell
opkg remove helloopenwrt
```
不出意外会看到以下内容，说明软件包已成功删除
```
Removing package helloopenwrt from root...
```

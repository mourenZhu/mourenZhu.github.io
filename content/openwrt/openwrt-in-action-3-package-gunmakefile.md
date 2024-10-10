---
title: 'OpenWrt开发入门实战(三)使用GUNmake'
date: 2024-10-10T16:40:50+08:00
draft: false
tags: ["openwrt"]
author: ["zhumouren"]
---

# 1. 创建Makefile
在上一章中我们是在包清单文件中定义如何编译`helloopenwrt`，以下是上一章中的包清单文件
```makefile
# Package build instructions; invoke the target-specific compiler to first compile the source file, and then to link the file into the final executable
define Build/Compile
		$(TARGET_CC) $(TARGET_CFLAGS) -o $(PKG_BUILD_DIR)/helloopenwrt.o -c $(PKG_BUILD_DIR)/helloopenwrt.c
		$(TARGET_CC) $(TARGET_LDFLAGS) -o $(PKG_BUILD_DIR)/$1 $(PKG_BUILD_DIR)/helloopenwrt.o
endef
```
由于这是一个示例项目，只有一个源文件，编译命令比较简单。根据关注点分离原则，一个程序如何编译应该又要在源码中给出。
```shell
cd /home/buildbot/helloopenwrt
touch Makefile
```
将以下内容复制进入`Makefile`
```makefile
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
		cp $(SOURCE_DIR)/* $(PKG_BUILD_DIR) -r
		$(Build/Patch)
endef

# Package build instructions; invoke the target-specific compiler to first compile the source file, and then to link the file into the final executable
define Build/Compile
		$(MAKE) -C $(PKG_BUILD_DIR) \
               CC="$(TARGET_CC)" \
           CFLAGS="$(TARGET_CFLAGS)" \
          LDFLAGS="$(TARGET_LDFLAGS)"
endef

# Package install instructions; create a directory inside the package to hold our executable, and then copy the executable we built previously into the folder
define Package/helloopenwrt/install
		$(INSTALL_DIR) $(1)/usr/bin
		$(INSTALL_BIN) $(PKG_BUILD_DIR)/helloopenwrt $(1)/usr/bin
endef

# This command is always the last, it uses the definitions and variables we give above in order to get the job done
$(eval $(call BuildPackage,helloopenwrt))

```
`Makefile`文件写好之后，我们可以输入以下命令测试一下是否有效
```shell
cd /home/buildbot/helloopenwrt
make clean
make
```
如果一切正常，当前目录下会有一个文件`helloopenwrt`，可以执行`./helloopenwrt`

# 2. 修改包清单文件
我们编写完`Makefile`后，在包清单文件里面还是原有的编译方式，这时我们需要修改包清单文件`/home/buildbot/mypackages/examples/helloopenwrt/Makefile`。只需要修改`Build/Compile`部分，修改内容如下
```makefile
# Package build instructions; invoke the GNU make tool to build our package
define Build/Compile
		$(MAKE) -C $(PKG_BUILD_DIR) \
               CC="$(TARGET_CC)" \
           CFLAGS="$(TARGET_CFLAGS)" \
          LDFLAGS="$(TARGET_LDFLAGS)"
endef
```

# 3. 重新构建包
修改包清单文件后，我们就可以再次构建包了。
```shell
cd /home/buildbot/openwrt
make package/helloopenwrt/{clean,compile}
```

如果构建包时出现异常，可能需要更新feeds
```shell
cd /home/buildbot/openwrt
./scripts/feeds update mypackages
./scripts/feeds install -a -p mypackages
```


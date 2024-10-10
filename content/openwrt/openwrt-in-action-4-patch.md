---
title: 'OpenWrt开发入门实战(四)添加补丁'
date: 2024-10-10T16:44:50+08:00
draft: false
tags: ["openwrt"]
author: ["zhumouren"]
---

# 1. 关于补丁
在OpenWrt中，补丁的功能使用一个名为`quilt`的工具完成的。这个工具由OpenWrt下载编译了，就在`/home/buildbot/openwrt/staging_dir/host/bin/quilt`。
如果我们要使用`quilt`首先要配置quilt的配置文件，然后配置环境变量。`注意，下面的EDITOR="nano"时指quilt默认的编辑环境，想用vi的可以替换成EDITOR="vi"`
```shell
cat << EOF > ~/.quiltrc
QUILT_DIFF_ARGS="--no-timestamps --no-index -p ab --color=auto"
QUILT_REFRESH_ARGS="--no-timestamps --no-index -p ab"
QUILT_SERIES_ARGS="--color=auto"
QUILT_PATCH_OPTS="--unified"
QUILT_DIFF_OPTS="-p"
EDITOR="nano"
EOF

export PATH=/home/buildbot/openwrt/staging_dir/host/bin:$PATH
```
配置完成后，输入以下命令就可以看到quilt的版本。
```shell
quilt --version
```


# 2. 使用补丁添加新文件

## 2.1 准备补丁代码
在创建补丁之前，需要准备补丁源码环境
```shell
cd /home/buildbot/openwrt
make package/helloopenwrt/{clean,prepare} QUILT=1
```
请注意，当执行`make`指定参数`QUILT=1`时，`build_dir`下的软件包不用于构建最终包，并且该参数会在构建目录中创建额外的文件夹和文件。

最后到补丁源码目录，确保提交所有现有的补丁:
```shell
cd /home/buildbot/openwrt/build_dir/target-x86_64_musl/helloopenwrt-1.0
quilt push -a
```
现在执行`quilt push -a`时不会执行任何操作，因为这个示例包中还没有任何补丁。但时在处理由多人编写的软件包时，最好执行这条命令，因为不知道其他人是否提交了补丁。

## 2.2 创建一个补丁
现在开始创建第一个补丁
```shell
quilt new 100-add_functions_files.patch
```
补丁的命令最好遵循OpenWrt的构建约定，名称通常以数字开始，后跟其功能的简短描述。数字一般有特殊含义
```
0xx - 上游补丁
1xx - 代码等待上游合并
2xx - 内核构建、配置、头补丁
3xx - 特定架构的补丁
4xx - mtd相关的补丁（子系统和驱动）
5xx - 文件系统相关的补丁
6xx - 通用网络补丁
7xx - 网络层/物理层驱动补丁
8xx - 其他驱动
9xx - 未分类的其他补丁
```

我们要在`helloopenwrt`中增加两个文件，首先用`quilt`追踪这两个文件
```shell
quilt add functions.c
quilt add functions.h
```
创建C源码并用`quilt`编辑
```shell
touch functions.c
quilt edit functions.c
```
```c
int mul(int a, int b)
{
        return a * b;
}
```
创建头文件
```shell
touch functions.h
quilt edit functions.h
```
```h
int mul(int, int);
```

编辑完成之后我们可以使用以下命令查看更改的内容
```shell
quilt diff
```

最后别忘了执行以下命令提交更改 
```shell
quilt refresh
```

## 2.3 将补丁提交到软件包中
在上述步骤中，我们已经在`build_dir`下的`helloopenwrt`代码中添加了补丁，但并没有将这些补丁添加到实际的软件包中，所以要执行以下命令，将补丁提交到软件包中。
```shell
cd /home/buildbot/openwrt
make package/helloopenwrt/update
```

这时候我们可以检测本地包源中的软件包的内容，和实际`helloopenwrt`的源码内容
```shell
ls -la /home/buildbot/mypackages/examples/helloopenwrt/
ls -la /home/buildbot/helloopenwrt/
```
我们可以发现本地包源中`/home/buildbot/mypackages/examples/helloopenwrt/`多了一个`patches`目录，而`/home/buildbot/helloopenwrt/`并无任何改变。  

现在我们需要修改包清单文件`/home/buildbot/mypackages/examples/helloopenwrt/Makefile`中的`Build/Prepare`部分，因为在之前`cp`命令没有带参数`-r`无法把补丁文件复制到`openwrt`构建代码中。以下时修改部分的代码
```makefile
define Build/Prepare
		mkdir -p $(PKG_BUILD_DIR)
		cp $(SOURCE_DIR)/* $(PKG_BUILD_DIR) -r
		$(Build/Patch)
endef
```

最后我们执行以下命令，已确保我们的补丁已生效
```shell
cd /home/buildbot/openwrt
make package/helloopenwrt/{clean,prepare}
ls -lls build_dir/target-x86_64_musl/helloopenwrt-1.0/
```
不出意外可以看到`build_dir/target-x86_64_musl/helloopenwrt-1.0/`下的文件都时重新生成的，而且functions.h、functions.c文件都存在，我们的补丁已生效。


# 3. 使用补丁修改原有文件

## 3.1 创建第二个补丁
在上一节中，我们创建了一个补丁，增加了两个文件，但文件中的`mul`方法并没有被调用，所以我们增加一个新的补丁，在`helloopenwrt.c`中调用`mul`方法。

```shell
cd /home/buildbot/openwrt
make package/helloopenwrt/{clean,prepare} QUILT=1
cd /home/buildbot/openwrt/build_dir/target-x86_64_musl/helloopenwrt-1.0
quilt push -a
quilt new 101-use_function.patch
```

## 3.2 编译原文件
现在使用`quilt`编辑`helloopenwrt.c`
```shell
quilt edit helloopenwrt.c
```
helloopenwrt.c
```c
#include <stdio.h>
#include "functions.h"
 
int main(void)
{
        int result = mul(2, 4);
        printf("\nHello, OpenWrt!\nThe mul is '%d'\n", result);
        return 0;
}
```
保存更改之后，我们可以使用以下命令查看更改内容和生成最终的补丁
```shell
quilt diff
quilt refresh
```
最后返回OpenWrt源码目录，使用新补丁更新软件包
```shell
cd /home/buildbot/openwrt
make package/helloopenwrt/update
```

# 4. 测试加入补丁的新包
在上一节我们更新了软件包，所以我们先要重新编译软件包
```shell
cd /home/buildbot/openwrt
make package/helloopenwrt/{clean,compile}
```
然后根据第二章的内容，将新的软件包放入openwrt容器中，进行测试。
如果不出意外，安装完新的`helloopenwrt`包之后，执行`helloopenwrt`，会出现以下结果
```
Hello, OpenWrt!
The mul is '8'
```

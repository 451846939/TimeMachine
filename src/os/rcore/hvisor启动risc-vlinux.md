# 在hvisor上运行risc-v的linux

我们主要参考官方文档并记录一些坑点

> https://hvisor.syswonder.org/chap02/QemuRISC-V.html

首先按照指导手册安装我们的[安装qemu](https://hvisor.syswonder.org/chap02/QemuRISC-V.html#安装qemu)

## 安装QEMU 9.0.2：

```
wget https://download.qemu.org/qemu-9.0.2.tar.xz
# 解压
tar xvJf qemu-9.0.2.tar.xz
cd qemu-9.0.2
# 配置Riscv支持
./configure --target-list=riscv64-softmmu,riscv64-linux-user 
make -j$(nproc)
#加入环境变量
export PATH=$PATH:/path/to/qemu-9.0.2/build
#测试是否安装成功
qemu-system-riscv64 --version
```



## 安装交叉编译器
riscv的交叉编译器需从riscv-gnu-toolchain获取并编译。

```
# 安装必要工具
sudo apt-get install autoconf automake autotools-dev curl python3 python3-pip libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev ninja-build git cmake libglib2.0-dev libslirp-dev

git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
git rm qemu 
git submodule update --init --recursive
#上述操作会占用超过5GB以上磁盘空间
# 如果git报网络错误，可以执行：
git config --global http.postbuffer 524288000
```

之后开始编译工具链：

```
cd riscv-gnu-toolchain
mkdir build
cd build
../configure --prefix=/opt/riscv64
sudo make linux -j $(nproc)
# 编译完成后，将工具链加入环境变量
echo 'export PATH=/opt/riscv64/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

这样就得到了 riscv64-unknown-linux-gnu工具链。



在这里基本都不会有太大的问题，下面几部都需要注意一些问题



## 编译Linux

```
git clone https://github.com/torvalds/linux -b v6.2 --depth=1
cd linux
git checkout v6.2
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- modules -j$(nproc)
# 开始编译
make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- Image -j$(nproc)
```

编译完成后Image在`arch/riscv/boot/`中,也可以直接` objcopy -O binary vmlinux Image`获得



## 制作ubuntu根文件系统

```
wget http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.2-base-riscv64.tar.gz
mkdir rootfs
dd if=/dev/zero of=riscv_rootfs.img bs=1M count=1024 oflag=direct
mkfs.ext4 riscv_rootfs.img
sudo mount -t ext4 riscv_rootfs.img rootfs/
sudo tar -xzf ubuntu-base-20.04.2-base-riscv64.tar.gz -C rootfs/

sudo cp /path-to-qemu/build/qemu-system-riscv64 rootfs/usr/bin/
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
sudo mount -o bind /dev rootfs/dev
sudo mount -o bind /dev/pts rootfs/dev/pts
sudo chroot rootfs 
# 进入chroot后，安装必要的软件包：
apt-get update
apt-get install git sudo vim bash-completion \
    kmod net-tools iputils-ping resolvconf ntpdate
exit

sudo umount rootfs/proc
sudo umount rootfs/sys
sudo umount rootfs/dev/pts
sudo umount rootfs/dev
sudo umount rootfs
```

这里的`sudo chroot rootfs`需要注意，可能会遇到下面的情况

```text
sudo chroot rootfs
chroot: failed to run command ‘/bin/bash’: No such file or directory
```

这里先检查一下下面这个是否存在

```
ls rootfs/lib/ld-linux-riscv64-lp64d.so.1
```

存在的话就进行下面步骤，不存在的话需要重新解压一下`ubuntu-base-20.04.2-base-riscv64.tar.gz`

```
sudo apt install qemu-user qemu-user-static binfmt-support
sudo update-binfmts --enable qemu-riscv64
```

主要是注册 qemu-riscv64 到 binfmt_misc（支持 chroot）

## 运行hvisor

我们先看下官方手册文档的描述

> 将做好的根文件系统、Linux内核镜像放在hvisor目录下的指定位置，在hvisor根目录下执行`make run ARCH=riscv64` 
>
> 默认情况下使用 PLIC，执行`make run ARCH=riscv64 IRQ=aia`开启AIA规范

在官方文档里写的是放在hvisor目录下的指定位置，这个指定位置其实蛮奇怪的，具体需要查看代码，首先我们看项目根目录的MakeFile 可以看到默认配置有`BOARD ?= qemu-gicv3`这个目录需要指定或者改掉因为指定目录是按照下面规则进行拼接的

```
platform/$(ARCH)/$(BOARD)/image/
```

那具体内核镜像和rootfs放在哪里呢，这时候我们需要看`platform/riscv64/platform.mk`有如下内容

```makefile
FSIMG1 := $(image_dir)/virtdisk/rootfs1.ext4
FSIMG2 := $(image_dir)/virtdisk/rootfs-busybox.qcow2
# HVISOR ENTRY
HVISOR_ENTRY_PA := 0x80200000
zone0_kernel := $(image_dir)/kernel/Image
zone0_dtb    := $(image_dir)/dts/zone0.dtb
# zone1_kernel := $(image_dir)/kernel/Image
# zone1_dtb    := $(image_dir)/devicetree/linux.dtb

QEMU_ARGS := -machine virt
QEMU_ARGS += -bios default
QEMU_ARGS += -cpu rv64
QEMU_ARGS += -smp 4
QEMU_ARGS += -m 2G
QEMU_ARGS += -nographic

QEMU_ARGS += -kernel $(hvisor_bin)
QEMU_ARGS += -device loader,file="$(zone0_kernel)",addr=0x90000000,force-raw=on
QEMU_ARGS += -device loader,file="$(zone0_dtb)",addr=0x8f000000,force-raw=on
# QEMU_ARGS += -device loader,file="$(zone1_kernel)",addr=0x84000000,force-raw=on
# QEMU_ARGS += -device loader,file="$(zone1_dtb)",addr=0x83000000,force-raw=on

QEMU_ARGS += -drive if=none,file=$(FSIMG1),id=X10008000,format=raw
QEMU_ARGS += -device virtio-blk-device,drive=X10008000,bus=virtio-mmio-bus.7
QEMU_ARGS += -device virtio-serial-device,bus=virtio-mmio-bus.6 -chardev pty,id=X10007000 -device virtconsole,chardev=X10007000 -S
QEMU_ARGS += -drive if=none,file=$(FSIMG2),id=X10006000,format=qcow2
QEMU_ARGS += -device virtio-blk-device,drive=X10006000,bus=virtio-mmio-bus.5
# -------------------------------------------------------------------
```

可以看到有`FSIMG1`这就是我们放rootfs的地方默认是`platform/$(ARCH)/$(BOARD)/image/virtdisk/rootfs1.ext4`



这里要注意有`FSIMG1`和`FSIMG2`，对应下面`QEMU_ARGS`增加的参数，所以可以取消掉一个，同时要看到`QEMU_ARGS += -device virtio-serial-device,bus=virtio-mmio-bus.6 -chardev pty,id=X10007000 -device virtconsole,chardev=X10007000 -S`这里有一个`-S`这个可以去掉



内核存放的位置就是`zone0_kernel`默认为`platform/$(ARCH)/$(BOARD)/image/kernel/Image`



这里我们还看到`zone0_dtb`所以不要忘了去`platform/$(ARCH)/$(BOARD)/image/dts`目录下执行一下make



当一切都做好以后运行

```text
make run ARCH=riscv64 BOARD=qemu-plic
或者注意前面的目录准备
make run ARCH=riscv64 BOARD=qemu-aia IRQ=aia
```










# rust-for-linux总结

# 1. rust-for-linux的分支作用

> https://rust-for-linux.com/branches

这个一定一定要看每个分支的作用还有功能支持，还有rust的分支为什么还留着

举个例子https://github.com/Rust-for-Linux/linux/blob/rust/Documentation/rust/arch-support.rst 不同分支支持的就不一样！！

还有在linux文档是有llvm的版本需求的可以看具体的文档举个例子🌰： https://github.com/Rust-for-Linux/linux/blob/rust-next/Documentation/process/changes.rst 当然有些llvm高版本也可以兼容代码

## rust-next

rust-next是包含下一个Linux内核合并窗口期间要提交的新Rust功能的分支。

对此分支的更改通过邮件列表发送的补丁进行提交。

它是linux-next的一部分。

## rust-fixes

rust-fixes是包含当前Linux内核周期的Rust修复程序的分支。

对此分支的更改通过邮件列表发送的补丁进行提交。

它是linux-next的一部分。

## rust-dev

rust-dev是用于集成目的的实验性分支。它是一个补丁队列，其中包含“看起来足够好”的补丁。

其预期用例包括：

尽早查找合并/应用冲突。 为开发提供一个共同基础，该基础需要尚未包含在主线或rust-next中的功能，即提前访问功能。这可能包括来自其他子系统的与Rust相关的更改。 通过向更多开发人员轻松提供它们，为补丁提供额外的测试。 请注意，此分支可能会经常更新/变基，并且在将来可能会被删除。

## rust

rust是Rust支持合并到内核之前两年进行开发的原始分支。

它包含项目作为原型/展示中工作的大多数抽象。其中一些最终将上游合并，其他一些可能会根据上游的反馈进行重新设计，而一些可能会在不需要时被丢弃。

该分支现在基本上已冻结，通常只会将意图最小化与主线的差异的更改合并到其中。当差异足够小时，将对该分支进行归档/删除。在那之前，该分支对于查看尚未上游的内容以及对于某些下游项目基于其进行工作是有用的。



# 2.实验环境

最开始遇到的问题就是qemu如何启动编译好的内核以及initramfs如何制作，做好了以后如何更新



可以使用debian的镜像或者busybox做，如何更新呢，只需要把ko文件通过挂载或者scp传输就可以了



再使用 

```shell
insmod rust_print.ko
rmmod rust_print.ko
```

就可以了





编写e1000的话，一定一定要用指定的实验环境！！！！不然就是痛苦的环境问题，rust语法等等有些特性还在重写，因为本身目前rust-for-linux驱动侧的代码并不算很成熟！！！没有合并到主分支的驱动代码是很可能修改的！合并到主分支的稳定性会好很多

还有，**一定一定**要注意你的分支！！！**e1000** 就要用**e1000**分支！

https://github.com/fujita/linux/tree/rust-e1000



安装e1000环境的时候

如果出现了bindgen 0.56.0下载不下来

可以去 https://github.com/rust-lang/rust-bindgen.git 下载然后切换分支使用cargo install 以及可以把bindgen-cli的cli去掉如下：

```shell
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen
```





在迁移rust驱动代码的时候遇到的编译的坑

这个是`Permit impl Trait in type aliases`

https://github.com/rust-lang/rust/issues/63063

这个是`impl const` and `~const` 

https://github.com/rust-lang/rust/issues/110395

https://github.com/rust-lang/rust/issues/67792



关于`rust-analyzer` 代码提示启动，这里可是大坑了



为我之前编译都是加了O=build，然后使用

```shell
make LLVM=1 O=build rust-analyzer
```

这时候问题就来了，bindgen生成的代码没有提示，可以编译通过，就是没有提示，写代码相当痛苦

根本原因是生成的`rust-project.json` OBJTREE的问题多了一个build！

```json
"display_name": "bindings",
"edition": "2021",
"env": {
  //重点在这里/mnt/rust-for-linux/fujita/linux/build 的build
  "OBJTREE": "/mnt/rust-for-linux/fujita/linux/build",
  "RUST_MODFILE": "This is only for rust-analyzer"
},
"is_proc_macro": false,
"is_workspace_member": true,
"root_module": "/mnt/rust-for-linux/fujita/linux/rust/bindings/lib.rs"
```

要把build去掉，让它扫到源码，可以观察编译后的目录，编译以后生成的代码和源码没有在一个目录，虽然有个链接源码目录，但是好像扫不到，编译不加O=build就有提示了，必须要和源码在一个目录才有提示。。。





ps： bindgen生成的代码在哪里张啥样一定要去看一看，还有如何使用rust-for-linux是如何使用bindgen的



# 3.e1000驱动

linux里本身有e1000驱动的实现，可以看一下c实现，也挺有意思的，可以对比一下两者差距

关于e1000在MIT 6.S081中有相关的介绍

https://pdos.csail.mit.edu/6.S081/2020/labs/net.html

https://pdos.csail.mit.edu/6.S081/2020/readings/8254x_GBe_SDM.pdf

想要真的理解e1000的话还是要看具体的pdf文档，重点理解ring

因为我们是有模版所以写起来会简单一些，但是对网络分层还有网络协议这块还是需要非常了解的





在这里可以初体验了一下rust-for-linux写驱动的快乐



在这里我们也会接触到一个有意思的东西[Out-of-tree modules](https://rust-for-linux.com/out-of-tree-modules#out-of-tree-modules) 这个就很重要虽然用的人少，但是在听了模块化操作系统以后，可以利用rust的cargo来解耦和操作系统无关的功能，然后只需要找需要的包来进行组装自己的操作系统即可



那这里我们也有一些坑比如上面的`rust-analyzer` 提示问题



解决方法如下：

目录之外如果我们需要开启rust-analyzer提示需要在代码的src平级目录使用

```shell
make LLVM=1 -C /mnt/rust-for-linux/linux/build M=$PWD rust-analyzer
```

/mnt/rust-for-linux/linux 替换成你自己的linux目录



在vscode `.vscode/settings.json`中添加如下

```json
    "rust-analyzer.linkedProjects": [
        "${workspaceFolder}/e1000-driver/rust-project.json",
        "${workspaceFolder}/linux/rust-project.json"
    ],
```



# 4. 终极实验virtio

这个就是真的由于时间关系还有驱动不完善，各自有各自的实现，兼容这些问题导致浪费了大量的时间

还有要理解一下bus、device、device_driver之间的关系

如果实现virtio_blk或者别的，一定要去看一下c实现，还有linux里这3个源码文件的结构体定义，看了以后才能理解一些nvme驱动的封装，还有内存的分配在实验的rust e1000里分配内存使用了dma分配，内核里大部分是kmalloc



这里发现还有很多知识的欠缺和断链比如rust的内存分配是如何分配的，在rust-for-linux中都有答案，所以想要用rust写好驱动这些基础模块实现和linux中的差别以及如何封装的一定要了解



在写的过程中最需要操作的就是如何把c代码转换为rust的抽象，c的实现其实个人觉得已经很优秀，转换为rust就需要把c面向过程语言是如何实现类似trait功能的理解清楚，之后再根据自己的理解去抽象成rust代码







时间关系，最后一个实验实现的东西并不多。。。环境问题和理解比较花时间。。后续也会继续抽时间玩这个东西





总体来说，学到了很多，也玩到了rust-for-linux，并且体验到了驱动的编写，通过e1000也了解了网卡那块的处理，包括通过virtio的学习也理解了自己在使用虚拟机读取本机文件的时候是怎么操作的


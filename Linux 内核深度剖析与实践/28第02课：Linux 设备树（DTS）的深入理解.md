# 2/8第02课：Linux 设备树（DTS）的深入理解

### 设备树的出现

上一节说过设备树的出现是为了解决内核中大量的板级文件代码，通过 DTS 可以像应用程序里的 XML 语言一样很方便的对硬件信息进行配置。关于设备树的出现其实在 2005 年时候就已经在 PowerPC Linux 里出现了，由于 DTS 的方便性，慢慢地被广泛应用到 ARM、MIPS、X86 等架构上。为了理解设备树的出现的好处，先来看下在使用设备树之前是采用什么方式的。

关于硬件的描述信息之前一般放在一个个类似 arch/xxx/mach-xxx/board-xxx.c 的文件中，如：

```
static struct resource gitchat_resource[] = {
    {
        .start = 0x20100000 ,
        .end = 0x20100000 +1,
        .flags = IORESOURCE_MEM 
        …
        .start = IRQ_PF IRQ_PF 15 ,
        .end = IRQ_PF IRQ_PF 15 ,
        .flags = IORESOURCE_IRQ | IORESOURCE_IRQ_HIGHEDGE
    }
};

static struct platform_device gitchat_device = {
    .name name ="gitchat",
    .id = 0,
    .num_resources num_resources = ARRAY_SIZE(gitchat_resource),
    .resource = gitchat_resource,
};

static struct platform_device *ip0x_devices[] __initdata ={
    &gitchat_device,
};

static int __init ip0x_init(void)
{
    platform_add_devices(ip0x_devices, ARRAY_SIZE(ip0x_devices)); 
}
```

一个很少的地址获取，我们就要写大量的类似代码，当年 Linus 看到内核里有大量的类似代码，很是生气并且在 Linux 邮件列表里发了份邮件，才有了现在的设备树概念，至于设备树的出现到底带来了哪些好处，先看一下设备树的文件：

```
eth:eth@ 4,c00000 {
    compatible ="csdn, gitchat";
    reg =<
        4 0x00c00000 0x2
        4 0x00c00002 0x2
    >;
    interrupt-parent =<&gpio 2>;
    interrupts=<14 IRQ_TYPE_LEVEL_LOW>;
    …
};
```

从代码中可看到对于 GITCHAT 这个网卡驱动、一些寄存器、中断号和上一层 gpio 节点都很清晰的被描述。比上一图的代码优化了很多，也容易维护了很多。这样就形成了设备在脚本，驱动在 c 文件里的关系图：

![image](http://images.gitbook.cn/eb4a8cd0-c487-11e7-a68e-3d5e4f9f8dae)

从图中可以看出 A、B、C 三个板子里都含有 GITCHAT 设备树文件，这样对于 GITCHAT 驱动写一份就可以在 A、B、C 三个板子里共用。从上幅图里不难看出，其实设备树的出现在软件模型上相对于之前并没有太大的改变，设备树的出现主要在设备维护上有了更上一层楼的提高，此外在内核编译上使内核更精简，镜像更小。

### 设备树的文件结构和剖析

设备树和设备树之间到底是什么关系，有着哪些依赖和联系，先看下设备树之间的关系图：

![image](http://images.gitbook.cn/eec42330-c487-11e7-b434-554ad14cb5ef)

除了设备树（DTS）外，还存有 dtsi 文件，就像代码里的头文件一样，是不同设备树共有的设备文件，这不难理解，但是值得注意的是如果 dts 和 dtsi 里都对某个属性进行定义的话，**底层覆盖上层的属性定义**。这样的好处是什么呢？假如你要做一块电路板，电路板里有很多模块是已经存在的，这样就可以直接像包含头文件一样把共性的 dtsi 文件包含进来，大大减少工作量，后期也可以对类似模块再次利用。

设备树文件的格式是 dts，包含的头文件格式是 dtsi，dts 文件是一种程序员可以看懂的格式，但是 Uboot 和 Linux 只能识别二进制文件，不能直接识别。所以就需要把 dts 文件编译成 dtb 文件。把 dts 编译成 dtb 文件的工具是 dtc，位于内核目录下 scripts/dtc，也可以手动安装：sudo apt-get install device-tree-compiler 工具。具体 dts 是如何转换成机器码并在内存里供 kernel 识别的，请看下图：

![image](http://images.gitbook.cn/f29b1bd0-c487-11e7-b434-554ad14cb5ef)

### 设备树的应用

有了理论，在具体的工程里如何做设备树呢？这里介绍三大法宝：**文档、脚本、代码**。文档是对各种 node 的描述，位于内核 documentation/devicetree/bingdings/arm/ 下，脚本就是设备树 dts，代码就是你要写的设备代码，一般位于 arch/arm/ 下，以后在写设备的时候可以用这种方法，绝对的事半功倍。很多上层应用开发者没有做过内核开发的经验，对内核一直觉得很神秘，其实可以换一种思路来看内核，相信上层应用开发者最熟悉的就是各种 API，工作中可以说就是和 API 打交道，对于内核也可以想象是各种 API，只不过是内核态的 API。这里设备文件就是根据各种内核态的 API 来调用设备树里的板级信息：

- `struct device_node *of_find_node_by_phandle(phandle handle);`
- `struct device_node *of_get_parent(const struct device_node_ *node);`
- `of_get_child_count()`
- `of_property_read_u32_array()`
- `of_property_read_u64()`
- `of_property_read_string()`
- `of_property_read_string_array()`
- `of_property_read_bool()`

具体的用法这里不做进一步的解释，大家可以查询资料或者看官网解释。

这里对设备树做个总结，设备树可以总结为三大作用：一是平台标识，所谓平台标识就是板级识别，让内核知道当前使用的是哪个开发板，这里识别的方式是根据 root 节点下的 compatible 字段来匹配。二是运行时配置，就是在内核启动的时候 ramdisk 的配置，比如 bootargs 的配置，ramdisk 的起始和结束地址。三是设备信息集合，这也是最重要的信息，集合了各种设备控制器，接下来的实践课会对这一作用重点应用。这里有张图对大家理解设备树作用有一定的帮助：

![image](http://images.gitbook.cn/f64697a0-c487-11e7-a68e-3d5e4f9f8dae)

### 设备树自测题

由于文章结构的安排和图文展示的方式有限，本篇关于设备树的课程先介绍到这里。下面有几道自测题，答案在读者圈里。

1.u-boot 中既可以读取，也可以修改 device tree?

```
A.是                   B.否
```

2.找出如下 DT 节点中不恰当的地方，并回答原因：

```
usb1:usb@e0003000{
    compatible = "chipdea,usb2", "xlnx,zynq-usb-2.20a";
    static = "disabled";
    clocks = <&clkc 29>;
    reg = <0xe0003000 0x1000>;
}
```
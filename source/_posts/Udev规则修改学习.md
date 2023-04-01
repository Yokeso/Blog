---
title: Udev规则修改学习
date: 2023-03-12 10:03:41
tags: [Udev规则]
---

# Udev规则修改学习

## 0x01 udev简介

udev的全称是Dynamic device management,也就是曾经的`devfs`的继任者。`udev`的主要目的是为动态`/dev`目录提供用户空间的解决方案，以及实现持久的设备命名。

在典型的linux系统上，`/dev`目录主要用于存储类似文件的设备节点，在这个目录下的每个节点都指向系统设备的一部分，这部分可能存在，也可能不存在。用户的应用程序就是通过使用这些设备节点来与系统的硬件进行交互的。这里引用中文文档中的一些原话

> 最初的/dev目录只是用可能出现在系统中的每个设备填充。由于这个原因，/dev目录通常非常大。devfs提供了一种更易于管理的方法(值得注意的是，它只使用插入系统的硬件来填充/dev)，以及一些其他功能，但是系统被证明存在一些难以修复的问题。
>
> udev是管理/dev目录的“新”方法，旨在清除以前/dev实现中的一些问题，并提供一个健壮的前进路径。为了创建和命名与系统中存在的设备相对应的/dev设备节点，udev依赖于sysfs提供的信息与用户提供的规则进行匹配。

那说了这么多，udev的真正作用是什么呢？对于标准设备而言，udev很可能是一个永远触碰不到的东西。但是对于新的或者外来的设备而言，如果不进行配置修改，这些设备可能会导致无法访问。或者说linux本身会给这些设备分配不恰当的名字，所属或者权限来创建设备文件。包括RS-232串口以及音视频设备的属组或者权限都可以在udev中进行更改。

## 0x02 规则文件和语义

### 0x21 规则文件介绍

在决定如何命名设备以及执行哪些附加操作时，udev会读取一系列的规则文件。这些文件保存在`/etc/udev/rules.d`中。文件的后缀名均为`.rules`。

其中要注意的是`50-udev.rules`。这个文件中存放的是默认的udev存储规则，所以用户不应该将规则直接写入这个文件中。在这个文件中包含了一些示例以及一些证明`devfs`样式`/dev`布局的默认规则。

`rules.d`中的文件按照此法顺序解析。所以在某些情况下，解析的规则非常重要。这里还是引用一下文档中的说法

> 通常，您希望在缺省值之前解析您自己的规则，因此我建议您在/etc/udev/rules.d/10-local.rules上创建一个文件，并将所有规则写入该文件。

在规则文件中，以`#`开头的行被视为注释。每隔一个非空行就是一个规则，规则之间不能跨越多行。

一个设备可以由多个规则机型匹配。udev在发现匹配规则时不会停止处理，会继续搜索并且尝试应用它所知道的每个规则。

### 0x22 语法规则

每个规则都由一系列的`key-value`对组成。这些对之间由都好分割。识别规则所适用设备的条件是键的匹配。一个规则至少要有一个匹配键和一个赋值键。

下面举一个简单的例子：

```shell
KERNEL=="hdb",NAME="my_spare_disk"
```

这里包括了一个匹配键和一个赋值键。匹配键使用的相等运算符进行匹配`==`（后续会进行详细介绍）。赋值键通过赋值运算符`=`进行值的赋予。

> 注意udev不支持任何形式的行延续。不要在您的规则中插入任何换行符，因为这将导致udev将您的一个规则视为多个规则，并且不能按预期工作。

### 0x23 udev key/value操作符

|        操作符         | 匹配或赋值 |                      解释                      |
| :-------------------: | :--------: | :--------------------------------------------: |
|          ==           |    匹配    |                    相等比较                    |
|          !=           |    匹配    |                    不等比较                    |
|           =           |    赋值    | 分配一个特定的值给该键，他可以覆盖之前的赋值。 |
|          +=           |    赋值    |           追加特定的值给已经存在的键           |
| :=      赋值       。 |    赋值    | 分配一个特定的值给该键，后面的规则不可能覆盖它 |

### 0x24 udev规则匹配键

> ACTION： 事件 (uevent) 的行为，例如：add( 添加设备 )、remove( 删除设备 )。
>
> KERNEL： 内核设备名称，例如：sda, cdrom。
>
> DEVPATH：设备的 devpath 路径。
>
> SUBSYSTEM： 设备的子系统名称，例如：sda 的子系统为 block。
>
> BUS： 设备在 devpath 里的总线名称，例如：usb。
>
> DRIVER： 设备在 devpath 里的设备驱动名称，例如：ide-cdrom。
>
> ID： 设备在 devpath 里的识别号。
>
> SYSFS{filename}： 设备的 devpath 路径下，设备的属性文件“filename”里的内容。
>
> 例如：SYSFS{model}==“ST936701SS”表示：如果设备的型号为 ST936701SS，则该设备匹配该 匹配键。
>
> 在一条规则中，可以设定最多五条 SYSFS 的 匹配键。
>
> ENV{key}： 环境变量。在一条规则中，可以设定最多五条环境变量的 匹配键。
>
> PROGRAM：调用外部命令。
>
> RESULT： 外部命令 PROGRAM 的返回结果。

例如：

```shell
PROGRAM=="/lib/udev/scsi_id -g -s $devpath", RESULT=="35000c50000a7ef67"
```

可以解释为调用外部命令 /lib/udev/scsi_id查询设备的 SCSI ID，如果返回结果为 35000c50000a7ef67，则该设备匹配该 匹配键。

### 0x25 值和可替换操作符

介绍完操作符以及键后，我们还要来介绍规则文件中的值。在udev中，用户可以直接定制udev规则文件的值，也可以引用下列操作替换符来进行。

> $kernel, %k：设备的内核设备名称，例如：sda、cdrom。
>
> $number, %n：设备的内核号码，例如：sda3 的内核号码是 3。
>
> $devpath, %p：设备的 devpath路径。
>
> $id, %b：设备在 devpath里的 ID 号。
>
> $sysfs{file}, %s{file}：设备的 sysfs里 file 的内容。其实就是设备的属性值。
>
> 例如：$sysfs{size} 表示该设备 ( 磁盘 ) 的大小。
>
> $env{key}, %E{key}：一个环境变量的值。
>
> $major, %M：设备的 major 号。
>
> $minor %m：设备的 minor 号。
>
> $result, %c：PROGRAM 返回的结果。
>
> $parent, %P：父设备的设备文件名。
>
> $root, %r：udev_root的值，默认是 /dev/。
>
> $tempnode, %N：临时设备名。
>
> %%：符号 % 本身。
>
> $$：符号 $ 本身。

## 0x03 规则文件的编写

从上面的介绍和学习来说，可以见到udev的规则和语法都较为简单。只要有了匹配键和赋值键就能编写我们想要的规则文件

```shell
KERNEL=="tty", NAME="%k", GROUP="tty", MODE="0666", OPTIONS="last_rule"
```

该规则说明：如果有一个设备的内核设备名称为tty(KERNEL=="tty")，那么设置新的权限为0600(MODE="0666")，所在的组是tty(GROUP="tty")。它也设置了一个特别的设备文件名:%K。在这里例子里，%k代表设备的内核名字。那也就意味着内核识别出这些设备是什么名字，就创建什么样的设备文件名。

在这里的关键就是`==`的匹配情况。那么对于一个设备怎么获取设备的属性呢？udevadm提供了一种方式

```shell
udevadm info -q path -n $(filepath) 
udevadm info -a -p $(filepath)
```

其中`udevadm info -q path -n $(filepath) ` 能够返回sysfs中的设备路径，将这一设备路径放入`udevadm info -a -p $(filepath)`的`filepath`中就能获得设备的结果信息。例子如下

```shell
[root@localhost samples]# udevadm info -q path -n /dev/nr0lun0
/devices/virtual/block/nr0lun0
[root@localhost samples]# udevadm info -a -p /devices/virtual/block/nr0lun0

Udevadm info starts with the device specified by the devpath and then
walks up the chain of parent devices. It prints for every device
found, all possible attributes in the udev rules key format.
A rule to match, can be composed by the attributes of the device
and the attributes from one single parent device.

  looking at device '/devices/virtual/block/nr0lun0':
    KERNEL=="nr0lun0"
    SUBSYSTEM=="block"
    DRIVER==""
    ATTR{alignment_offset}=="0"
    ATTR{capability}=="10"
    ATTR{discard_alignment}=="0"
    ATTR{ext_range}=="64"
    ATTR{hidden}=="0"
    ATTR{inflight}=="       0        0"
    ATTR{range}=="64"
    ATTR{removable}=="0"
    ATTR{ro}=="0"
    ATTR{size}=="15628107776"
    ATTR{stat}=="    4457        0    47216      616     6364        0  8391544 11071107        0    21985 11071723        0        0        0        0 "

```

这样我们就可以通过信息去进行匹配。比如上面这个设备我们就可以匹配 `KERNEL=="nr[0-9]lun0"`
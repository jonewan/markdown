# Linux设备，驱动，总线

Linux软件架构的思想 —— 高内聚，低耦合。

## 驱动需要跨平台

Linux提倡一份驱动适配多个电路板，同一份驱动代码需要在不同的电路板上运行。因此那些板级互联信息不应该出现在驱动里，驱动仅仅只做它自己该做的流程，因此Linux将这些包含设备基地址、中断号等等的板级互联信息分离出来，称之为`设备`。`设备`用来描述基地址、中断号、时钟、DMA、复位等信息，其代码存在于`arch/xxx`目录下。`驱动`用来完成外设的功能，如网卡收发包、声卡录放、SD卡读写，其代码存在于`drivers/xxx`目录下。

## 统一的纽带——总线（bus）

当驱动与设备剥离开来之后，需要有一个统一的纽带将设备与驱动联系起来，为驱动提供设备的信息，并且匹配驱动与设备，完成二者的关联，这个统一的纽带就是`总线`。其代码存在于`drivers/base/platform.c`,`drivers/pci/pci-driver.c`...等目录下。总线类似于一个管理器，设备与驱动都挂载在总线上，在Linux中注册设备的时候，系统会去寻找与其匹配的驱动，注册驱动的时候系统就会去寻找与其匹配的设备。匹配完成后在`/sys/bus/xxx`下就会存在两个子目录`drivers`与`device`。

## 设备端代码：arch/xxx

设备端代码使用一个统一的数据结构`struct resource`用来描述具体硬件的起始地址、中断号等信息，（以platform总线为例）并将这些recourse信息填入`struct platform_device`设备数据结构里。最后通过一个函数`platform_add_devices()`将设备注册到platform总线上。例如下面代码：

```c
static struct resource dm9000_resource1[] = {
    {
        .start = 0x20100000,
        .end = 0x20100000 + 1,
        .flags = IORESOURCE_MEM
        …
        .start = IRQ_PF15,
        .end = IRQ_PF15,
        .flags = IORESOURCE_IRQ | IORESOURCE_IRQ_HIGHEDGE
    }
};
static struct platform_device dm9000_device1 = {
    .name = "dm9000",
    .id = 0,
    .num_resources = ARRAY_SIZE(dm9000_resource1),
    .resource = dm9000_resource1,
};
static struct platform_device *ip0x_devices[] __initdata = {
    &dm9000_device1,
    &dm9000_device2,
};
static int __init ip0x_init(void)
{
    platform_add_devices(ip0x_devices, ARRAY_SIZE(ip0x_devices));
}
```

## 驱动端代码：drivers/xxx

驱动端代码则通过一个统一的标准API:`platform_get_resource()`获取这些设备的recourse。

```c
static int dm9000_probe(struct platform_device *pdev)
{
    …
    db->addr_res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    db->data_res = platform_get_resource(pdev, IORESOURCE_MEM, 1);
    db->irq_res = platform_get_resource(pdev, IORESOURCE_IRQ, 0);
    …
}
static struct platform_driver dm9000_driver = {
    .driver = {
        .name = "dm9000",
        .pm = &dm9000_drv_pm_ops,
        .of_match_table = of_match_ptr(dm9000_of_matches),
    },
    .probe = dm9000_probe,
    .remove = dm9000_drv_remove,
};
```

这样一来驱动再也不用去关心这些板级信息了，因为所有的钣金信息都是通过标准API拿出来的，而板级信息又是在设备端填写的。设备端的代码在`arch/<board-xxx>`下，这部分代码本身就不具备可移植性。

## 总线match函数

总线通过match函数来匹配设备与驱动，`of_driver_match_device`用来进行设备树风格的匹配，`acpi_driver_match_device`用来进行ACPI风格的匹配，`platform_match_id`进行id_table风格的匹配，还有一种是原始的直接通过`设备名称`进行匹配。

```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);
    /* When driver_override is set, only bind to the matching driver */
    if (pdev->driver_override)
    return !strcmp(pdev->driver_override, drv->name);
    /* Attempt an OF style match first */
    if (of_driver_match_device(dev, drv))
    return 1;
    /* Then try ACPI style match */
    if (acpi_driver_match_device(dev, drv))
    return 1;
    /* Then try to match against the id table */
    if (pdrv->id_table)
    return platform_match_id(pdrv->id_table, pdev) != NULL;
    /* fall-back to driver name match */
    return (strcmp(pdev->name, drv->name) == 0);
}
```

## 一个驱动如何匹配多个设备

在c++中，一个`class`的实例可以有很多个，c++中通过指向自己的指针`this`来识别实例中具体调用哪个属性或方法。Linux也是类似的，其使用私有数据来模拟c++中this指针的功能。在驱动的`probe`函数中，通过分配设备，而不是直接创建设备，这样无论有多少个设备过来都是通过分配的，之后通过访问设备的私有数据来读取其recourse，并将其注册到总线上。通过私有数据就可以知道具体的设备信息。
注：Linux习惯使用alloc，Linux习惯使用私有数据。

## sysfs目录解释

在sysfs目录下的其中三个子目录`/sys/devices`，`/sys/bus`，`/sys/class`。
硬件的级联信息全部展现在`/sys/devices`目录下，而`/sys/bus/platform/devices`是扁平的，代表挂载在该总线上的设备，这些设备其实是`/sys/devices`的符号链接。
`/sys/class`是一种分类视角，其将所有的设备进行分类，例如`rtc`设备在class目录下的一个子目录，而该目录下有多个`rtc`设备，每个设备都是`/sys/devices`下真实设备的一个符号链接。
例如一个LED在`/sys/devices`和`/sys/class`:
> /sys/class/leds/v2m:green:user1
> /sys/devices/platform/smb/smb:motherboard/smb:motherboard:leds/leds/vm:green:user1

## fucking pain in the ass

Linus在arm社区指出在`arch/arm`里面的代码就是“fucking pain in the ass”，因为把所有的基地址，中断号等recourse信息写进内核里面简直就是污染内核，因此Linux采用了设备树的概念。提出了`设备在脚本，驱动在c里`的概念。每个设备使用`device tree`的脚本语言进行描述，成为了dts里的一个node，设备的板级互联信息就成为node下的一个属性。开机过程中执行的类似语句会帮忙从dts节点生成platform_device：
`of_platform_populate(NULL, of_default_bus_match_table,NULL, NULL);`

## dts和driver的匹配

在dts脚本中有一个compatible（兼容性）的属性，与驱动中相对应：

```dts
eth: eth@4,c00000 {
compatible = "davicom,dm9000";
…
};
```

```c
static const struct of_device_id dm9000_of_matches[] = {
    { .compatible = "davicom,dm9000", },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, dm9000_of_matches);

```

`of_platform_populate`函数会最终分配和展开platform_device。

```c

struct platform_device *of_device_alloc(struct device_node *np, const char *bus_id, struct device *parent)
{
    struct platform_device *dev;
    int rc, i, num_reg = 0, num_irq;
    struct resource *res, temp_res;
    /*分配platform_device*/
    dev = platform_device_alloc("", -1);

    if (!dev)
        return NULL;
    /* 自动填充recourse */
    while (of_address_to_resource(np, num_reg, &temp_res) == 0)
        num_reg++;

    num_irq = of_irq_count(np);

    /* Populate the resource table */
    if (num_irq || num_reg) {
        res = kzalloc(sizeof(*res) * (num_irq + num_reg), GFP_KERNEL);
        ...
        dev->num_resources = num_reg + num_irq;
        dev->resource = res;
        for (i = 0; i < num_reg; i++, res++) {
            rc = of_address_to_resource(np, i, res);
            WARN_ON(rc);
        }
        /*将中断号填充到platform recourse里*/
        if (of_irq_to_resource_table(np, res, num_irq) != num_irq)
            ...
    }
    ...
}
```

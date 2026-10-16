<!-- 源：/home/h-000-h/project/mydriver/4/hello4.c -->

```c
#include <linux/platform_device.h>
#include <linux/init.h>
#include <linux/module.h>
/*
        [   64.540799] helloplat: probe! name=hello_platform id=-1
        [   64.540807] helloplat: 这个设备是内核给我的,不是我造的
        [   64.540829] helloplat: 假设备已注册
        [ 1904.873698] helloplat: 假设备已注册
        [ 1904.873737] helloplat: probe! name=hello_platform id=-1
        [ 1904.873740] helloplat: 这个设备是内核给我的,不是我造的
        由此可见kerne platform 同时探测 driver device -> 两个都齐了才走 really_probe

------------------------------------------------------------------------------------------
        lrwxrwxrwx    - root 15 10月 20:47  driver -> ../../../bus/platform/drivers/hello_platform
        drwxr-xr-x    - root 15 10月 20:47  power
        lrwxrwxrwx    - root 15 10月 20:47  subsystem -> ../../../bus/platform
        .rw-r--r-- 4.1k root 15 10月 20:47 󰡯 driver_override
        .r--r--r-- 4.1k root 15 10月 20:47 󰡯 modalias
        .rw-r--r-- 4.1k root 15 10月 20:47 󰡯 uevent
        lrwxrwxrwx    - root 15 10月 20:47  hello_platform -> ../../../../devices/platform/hello_platform
        lrwxrwxrwx    - root 15 10月 20:47  module -> ../../../../module/hello4
        .-w------- 4.1k root 15 10月 20:47 󰡯 bind
        .-w------- 4.1k root 15 10月 20:47 󰡯 uevent
        .-w------- 4.1k root 15 10月 20:47 󰡯 unbind
        正向是`dev->driver = &drv->driver` ，反向是驱动用 klist 挂着所有绑定的设备
*/
#define DEV_NAME "hello_platform"

static struct platform_device *pfake_dev;

static int helloplat_probe(struct platform_device* pdev)
{
    pr_info("helloplat: probe! name=%s id=%d\n", pdev->name, pdev->id);
	pr_info("helloplat: 这个设备是内核给我的,不是我造的\n");
    return 0;
}

static void helloplat_remove(struct platform_device *pdev)
{
    pr_info("helloplat: remove! name=%s\n", pdev->name);
}

static struct platform_driver helloplat_driver =
{
    .probe =helloplat_probe,
    .remove = helloplat_remove,
    .driver=
    {
        .name = DEV_NAME, /* ★ 匹配就靠它 */
        /* .owner 由 platform_driver_register() 宏自动填 THIS_MODULE */
    },
};

static int __init helloplat_init(void)
{
#if 0
    /*
        [ 1904.873698] helloplat: 假设备已注册
        [ 1904.873737] helloplat: probe! name=hello_platform id=-1
        [ 1904.873740] helloplat: 这个设备是内核给我的,不是我造的
    */
    int ret;
    /* 1) 先注册"设备"（模拟硬件已存在） */
    pfake_dev = platform_device_register_simple(DEV_NAME,-1,NULL, 0);
    if(IS_ERR(pfake_dev))
    {
        ret = PTR_ERR(pfake_dev);
        pr_err("helloplat: 建设备失败: %d\n", ret);
		return ret;
    }
    pr_info("helloplat: 假设备已注册\n");

    ret = platform_driver_register(&helloplat_driver);
    if(ret)
    {
        pr_err("helloplat: 注册驱动失败: %d\n", ret);
		platform_device_unregister(pfake_dev);
		return ret;
    }
#else
    /*
        [   64.540799] helloplat: probe! name=hello_platform id=-1
        [   64.540807] helloplat: 这个设备是内核给我的,不是我造的
        [   64.540829] helloplat: 假设备已注册

    */
    int ret;
    ret = platform_driver_register(&helloplat_driver);
    if(ret)
    {
        pr_err("helloplat: 注册驱动失败: %d\n", ret);
		platform_device_unregister(pfake_dev);
		return ret;
    }

    pfake_dev = platform_device_register_simple(DEV_NAME,-1,NULL, 0);
    if(IS_ERR(pfake_dev))
    {
        ret = PTR_ERR(pfake_dev);
        pr_err("helloplat: 建设备失败: %d\n", ret);
		return ret;
    }
    pr_info("helloplat: 假设备已注册\n");
#endif
    return 0;
}

static void __exit hello_exit(void)
{
	platform_device_unregister(pfake_dev);          /* 先摘设备(顺序也可反过来) */
	platform_driver_unregister(&helloplat_driver);  /* ★ unregister! */
}

module_init(helloplat_init);
module_exit(hello_exit);

MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("h-000-h");
MODULE_DESCRIPTION("platform_driver + 假设备");
```

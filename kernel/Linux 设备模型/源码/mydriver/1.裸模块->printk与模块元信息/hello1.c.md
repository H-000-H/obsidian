<!-- 源：/home/h-000-h/project/mydriver/1/hello1.c -->

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init hellow_init(void)
{
    printk(KERN_INFO "加载了 hello 模块\n");
    return 0;
}

static void __exit hellow_exit(void)
{
    printk(KERN_INFO "卸载了 hello 模块\n");
}

module_init(hellow_init);
module_exit(hellow_exit);

MODULE_LICENSE("MIT");
MODULE_DESCRIPTION("hellow demo");
MODULE_VERSION("1.0.0");
```

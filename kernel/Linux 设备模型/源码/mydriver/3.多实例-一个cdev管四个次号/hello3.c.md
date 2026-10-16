<!-- 源：/home/h-000-h/project/mydriver/3/hello3.c -->

```c
#include <linux/fs.h>        /* alloc_chrdev_region / file_operations */
#include <linux/cdev.h>      /* struct cdev / cdev_add */
#include <linux/device.h>    /* class_create / device_create */
#include <linux/uaccess.h>   /* copy_to_user */
#include <linux/kdev_t.h>   /* MKDEV / MAJOR / MINOR */
#include <linux/module.h>
#include <linux/string.h>    /* strlen */
#define DEV_NAME "hello"
#define DEV_MINOR_COUNT 4 /* ★ 4 个次号 */
/* 三个全局：模块活着就一直存在 */
static dev_t hello_devt; //设备号
static struct cdev hello_cdev;/* I/O 层：dev_t -> fops */
static struct class* hello_class;/* 前端：让 /dev 节点出现 */

/* ---- 每个实例一份数据 ---- */
struct hello_inst
{
    const char* msg;
    int index;
};

static const struct hello_inst hello_insts[DEV_MINOR_COUNT]=
{
    { "I am hello0\n", 0 },
	{ "I am hello1\n", 1 },
	{ "I am hello2\n", 2 },
	{ "I am hello3\n", 3 },
};

static int hello_open(struct inode *inode, struct file* filp)
{
    int minor = iminor(inode);/* ★ 次号 = 第几个实例 */

    if (minor>=DEV_MINOR_COUNT) 
        return -ENODEV;

    filp->private_data = (void*)&hello_insts[minor];/* ★ 挂上本实例 */
    
    pr_info("hello: open minor=%d\n", minor);

    return 0;
}

static ssize_t hello_read(struct file*filp,char __user *buf,size_t len,loff_t*off)
{
    const struct hello_inst *inst = filp->private_data;   /* ★ 直接取 */

    const char*msg = inst->msg;
    
    const size_t total = strlen(msg);

    if(*off >=total)
        return 0;

    size_t n = min_t(size_t, len,total - *off );

    if(copy_to_user(buf,msg+ *off, n))
        return -EFAULT;
    *off +=n;
    return n;
}

static const struct file_operations hello_fops =
{
    .owner = THIS_MODULE,
    .read  = hello_read,
    .open  = hello_open,
};

static int __init hello_init(void)
{
	int ret, i;                                    /* ★ int i */

	ret = alloc_chrdev_region(&hello_devt, 0, DEV_MINOR_COUNT, DEV_NAME);
	if (ret) { pr_err("hello: alloc_chrdev_region 失败: %d\n", ret); return ret; }

	pr_info("hello: 占号 %d:%d .. %d:%d\n",
		MAJOR(hello_devt), MINOR(hello_devt),
		MAJOR(hello_devt), MINOR(hello_devt) + DEV_MINOR_COUNT - 1);

	cdev_init(&hello_cdev, &hello_fops);
	hello_cdev.owner = THIS_MODULE;
	ret = cdev_add(&hello_cdev, hello_devt, DEV_MINOR_COUNT);
	if (ret) { pr_err("hello: cdev_add 失败: %d\n", ret); goto err_chrdev; }

	hello_class = class_create(DEV_NAME);
	if (IS_ERR(hello_class)) { ret = PTR_ERR(hello_class); goto err_cdev; }

	for (i = 0; i < DEV_MINOR_COUNT; i++) {
		struct device *d = device_create(hello_class, NULL,
						 MKDEV(MAJOR(hello_devt), i),
						 NULL,             /* ★★ drvdata 别漏! */
						 "%s%d", DEV_NAME, i);
		if (IS_ERR(d)) { ret = PTR_ERR(d); goto err_device; }
	}

	pr_info("hello: 就绪\n");
	return 0;

err_device:
	while (i--)                                    /* ★ 先判断再自减 */
		device_destroy(hello_class, MKDEV(MAJOR(hello_devt), i));
	class_destroy(hello_class);
err_cdev:
	cdev_del(&hello_cdev);
err_chrdev:
	unregister_chrdev_region(hello_devt, DEV_MINOR_COUNT);   /* ★ 4 个 */
	return ret;
}

static void __exit hello_exit(void)
{
	int i;

	for (i = 0; i < DEV_MINOR_COUNT; i++)
		device_destroy(hello_class, MKDEV(MAJOR(hello_devt), i));
	class_destroy(hello_class);
	cdev_del(&hello_cdev);
	unregister_chrdev_region(hello_devt, DEV_MINOR_COUNT);   /* ★ 4 个 */
	pr_info("hello: 已卸载\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("h-000-h");
MODULE_DESCRIPTION("一个 cdev 管 4 个次号");
MODULE_VERSION("1.0.0");
```

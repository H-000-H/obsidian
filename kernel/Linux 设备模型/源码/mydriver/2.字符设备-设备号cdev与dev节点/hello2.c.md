<!-- 源：/home/h-000-h/project/mydriver/2/hello2.c -->

```c
#include <linux/fs.h>        /* alloc_chrdev_region / file_operations */
#include <linux/cdev.h>      /* struct cdev / cdev_add */
#include <linux/device.h>    /* class_create / device_create */
#include <linux/uaccess.h>   /* copy_to_user */
#include <linux/module.h>
#define DEV_NAME "hello"

/* 三个全局：模块活着就一直存在 */
static dev_t hello_devt; //设备号
static struct cdev hello_cdev;/* I/O 层：dev_t -> fops */
static struct class* hello_class;/* 前端：让 /dev 节点出现 */

static ssize_t hello_read(struct file*filp,char __user *buf,size_t len,loff_t*off)
{
    static const char msg[] = "read from hello\n";   

    const size_t total = sizeof(msg) -1;

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
};

static int __init hello_init(void)
{
    int ret ;
    /* 1) 要一个设备号（动态主号） */
    ret = alloc_chrdev_region(&hello_devt,0,1,DEV_NAME);
    if(ret)
    {
        pr_err("hello: alloc_chrdev_region 失败: %d\n", ret);
        return ret;
    }

    pr_info("hello:设备号 %d : %d\n",MAJOR(hello_devt),MINOR(hello_devt));
    /* 2) 后端：dev_t -> fops 登记进 cdev_map */
    cdev_init(&hello_cdev, &hello_fops);
    hello_cdev.owner = THIS_MODULE;
    ret = cdev_add(&hello_cdev, hello_devt, 1);

    if(ret)
    {
        pr_err("hello: cdev_add 失败: %d\n", ret);
        goto err_chrdev;
    }

    /* 3) 前端：建 class + device -> udev 建 /dev/hellow */
    hello_class = class_create(DEV_NAME);
    if(IS_ERR(hello_class))
    {
        ret = PTR_ERR(hello_class);
        goto err_cdev;
    }

    if(IS_ERR(device_create(hello_class, NULL, hello_devt, NULL, DEV_NAME)))
    {
        ret = -ENODEV;
        goto err_class;
    }

    pr_info("hello: 就绪 —— cat /dev/%s\n", DEV_NAME);
	return 0;
err_class:  
    class_destroy(hello_class);
err_cdev:
    cdev_del(&hello_cdev);
err_chrdev:
    unregister_chrdev_region(hello_devt,1);
    return ret;
}

static void __exit hello_exit(void)
{
    device_destroy(hello_class, hello_devt);
    class_destroy(hello_class);
    cdev_del(&hello_cdev);
    unregister_chrdev_region(hello_devt, 1);
    pr_info("hello: 已卸载\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("h-000-h");
MODULE_DESCRIPTION("字符设备 + /dev 节点");
MODULE_VERSION("1.0.0");
```

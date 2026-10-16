<!-- 源：/home/h-000-h/project/mydriver/5/hello5.c -->

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/platform_device.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/kdev_t.h>
#include <linux/uaccess.h>
#include <linux/slab.h>          /* kzalloc / kfree */
#define DRV_NAME "hello5"
/*
    lrwxrwxrwx    - root 16 10月 11:16  driver -> ../../../bus/platform/drivers/hello5
    drwxr-xr-x    - root 16 10月 11:16  power
    lrwxrwxrwx    - root 16 10月 11:16  subsystem -> ../../../bus/platform
    .rw-r--r-- 4.1k root 16 10月 11:16 󰡯 driver_override
    .r--r--r-- 4.1k root 16 10月 11:16 󰡯 modalias
    .rw-r--r-- 4.1k root 16 10月 11:16 󰡯 uevent
*/
struct hello5_priv
{
    dev_t devt;
    struct cdev cdev;
    struct device* dev;
};

static ssize_t hello5_read(struct file *filp, char __user *buf, size_t len,loff_t *off)
{
    static const char msg[] = "read from hello5\n";
    const size_t total = sizeof(msg) - 1;
    size_t n;

   if (*off >= total)
        return 0;

    n = min_t(size_t,len,total-*off);
    if(copy_to_user(buf,msg+*off, n))
        return -EFAULT;
    *off +=n;
    return n;   
}

static const struct file_operations hello5_fops=
{
    .owner = THIS_MODULE,
    .read = hello5_read,
};
static struct class *hello5_class;      /* ★ 全局:每驱动一份 */

static int hello5_probe(struct platform_device *pdev)
{
	struct hello5_priv *priv;
	int ret;

	priv = kzalloc(sizeof(*priv), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;

	ret = alloc_chrdev_region(&priv->devt, 0, 1, DRV_NAME);
	if (ret) {
		dev_err_probe(&pdev->dev, ret, "alloc_chrdev_region 失败\n");
		goto err_mem;
	}

	cdev_init(&priv->cdev, &hello5_fops);
	priv->cdev.owner = THIS_MODULE;
	ret = cdev_add(&priv->cdev, priv->devt, 1);
	if (ret) {
		dev_err_probe(&pdev->dev, ret, "cdev_add 失败\n");
		goto err_chrdev;
	}

	priv->dev = device_create(hello5_class, &pdev->dev, priv->devt, priv,"%s%d", DRV_NAME, pdev->id);   
	if (IS_ERR(priv->dev)) 
    {
		ret = PTR_ERR(priv->dev);
		dev_err_probe(&pdev->dev, ret, "device_create 失败\n");
		goto err_cdev;
	}

	platform_set_drvdata(pdev, priv);

	pr_info("hello5: probe ok, devt=%d:%d\n",
		MAJOR(priv->devt), MINOR(priv->devt));
	return 0;

err_cdev:
	cdev_del(&priv->cdev);
err_chrdev:
	unregister_chrdev_region(priv->devt, 1);
err_mem:
	kfree(priv);
	return ret;
}

static void hello5_remove(struct platform_device *pdev)
{
	struct hello5_priv *priv = platform_get_drvdata(pdev);

	if (!priv)
		return;

	device_destroy(hello5_class, priv->devt);   /* ★ 只拆设备,不拆 class */
	cdev_del(&priv->cdev);
	unregister_chrdev_region(priv->devt, 1);
	kfree(priv);

	pr_info("hello5: remove ok\n");
}

static struct platform_device *fake_dev[2];

static struct platform_driver hello5_driver = {
	.probe  = hello5_probe,
	.remove = hello5_remove,      /* ★ void */
	.driver = { .name = DRV_NAME },
};


static int __init hello5_init(void)
{
	int ret;

	/* 1) class 建一次 */
	hello5_class = class_create(DRV_NAME);
	if (IS_ERR(hello5_class))
		return PTR_ERR(hello5_class);

	fake_dev[0] = platform_device_register_simple(DRV_NAME, 0, NULL, 0);
	if (IS_ERR(fake_dev[0])) {
		ret = PTR_ERR(fake_dev[0]);
		pr_err("hello5: 建设备0失败: %d\n", ret);
		goto err_class;
	}

	fake_dev[1] = platform_device_register_simple(DRV_NAME, 1, NULL, 0);
	if (IS_ERR(fake_dev[1])) {
		ret = PTR_ERR(fake_dev[1]);
		pr_err("hello5: 建设备1失败: %d\n", ret);
		goto err_dev0;
	}

	ret = platform_driver_register(&hello5_driver);
	if (ret) {
		pr_err("hello5: 注册驱动失败: %d\n", ret);
		goto err_dev1;
	}

	pr_info("hello5: 模块就绪\n");
	return 0;

	/* ★ 逆序回滚 */
err_dev1:
	platform_device_unregister(fake_dev[1]);
err_dev0:
	platform_device_unregister(fake_dev[0]);
err_class:
	class_destroy(hello5_class);
	return ret;
}


static void __exit hello5_exit(void)
{
	/* 逆序拆:设备 → 驱动 → class(★ class 必须最后) */
	platform_device_unregister(fake_dev[0]);   /* → remove → device_destroy(还要用 class!) */
	platform_device_unregister(fake_dev[1]);
	platform_driver_unregister(&hello5_driver);
	class_destroy(hello5_class);               
}


module_init(hello5_init);
module_exit(hello5_exit);

MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("h-000-h");
MODULE_DESCRIPTION("probe 里建字符设备");
```

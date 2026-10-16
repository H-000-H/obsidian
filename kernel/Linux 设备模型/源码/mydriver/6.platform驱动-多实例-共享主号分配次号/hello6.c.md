<!-- 源：/home/h-000-h/project/mydriver/6/hello6.c -->

```c
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt  
#include <linux/module.h>
#include <linux/init.h>
#include <linux/platform_device.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/kdev_t.h>
#include <linux/uaccess.h>
#include <linux/slab.h>          /* kzalloc / kfree */
#define DRV_NAME "hello6"
#define DEV_MAX 2
static dev_t hello6_dev;
static struct class *hello6_cls;
static struct platform_device *fake_dev[2];

struct hello6_priv
{
    struct cdev cdev;
    struct device* dev;
	dev_t 		devt;
};

static ssize_t hello5_read(struct file *filp, char __user *buf, size_t len,loff_t *off)
{
    static const char msg[] = "read from hello6\n";
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

static const struct file_operations hello6_fops=
{
    .owner = THIS_MODULE,
    .read = hello5_read,
};
static int hello6_probe(struct platform_device *pdev)
{
	struct hello6_priv *priv;
	int idx = pdev->id;
	int ret;

	if (idx < 0 || idx >= DEV_MAX) {
		dev_err(&pdev->dev, "id %d 超出范围\n", idx);
		return -EINVAL;
	}

	priv = kzalloc(sizeof(*priv), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;

	priv->devt = MKDEV(MAJOR(hello6_dev), MINOR(hello6_dev) + idx);

	cdev_init(&priv->cdev, &hello6_fops);     
	priv->cdev.owner = THIS_MODULE;
	ret = cdev_add(&priv->cdev, priv->devt, 1);
	if (ret) {
		dev_err_probe(&pdev->dev, ret, "cdev_add 失败\n");
		goto err_mem;
	}

	priv->dev = device_create(hello6_cls, &pdev->dev, priv->devt, priv,
				  "%s%d", DRV_NAME, idx);
	if (IS_ERR(priv->dev)) {
		ret = PTR_ERR(priv->dev);
		dev_err_probe(&pdev->dev, ret, "device_create 失败\n");
		goto err_cdev;
	}

	platform_set_drvdata(pdev, priv);
	pr_info("probe ok, devt=%d:%d\n", MAJOR(priv->devt), MINOR(priv->devt));

	return 0;                    /* ★ 成功路径的 return 放最后 */

err_cdev:
	cdev_del(&priv->cdev);
err_mem:
	kfree(priv);
	return ret;
}


/* ---------- remove:不再 unregister!号归 init/exit 管 ---------- */
static void hello6_remove(struct platform_device *pdev)
{
	struct hello6_priv*priv = platform_get_drvdata(pdev);

	if(!priv)
	{
		pr_err("err cound't get priv data");
		return;
	}

	device_destroy(hello6_cls, priv->devt);
	cdev_del(&priv->cdev);
	kfree(priv);
	priv =NULL;
	pr_info("hello6: remove ok\n");
} 

static struct platform_driver hello6_driver=
{
	.probe = hello6_probe,
	.remove = hello6_remove,
	.driver =
			{
				.name = DRV_NAME,
			},
};

static __init int hello6_init(void)
{
	int ret;

	ret = alloc_chrdev_region(&hello6_dev, 0, DEV_MAX, DRV_NAME);
	if (ret) {
		pr_err("alloc_chrdev_region 失败: %d\n", ret);
		return ret;
	}
	pr_info("占号 %d:%d .. %d:%d\n", MAJOR(hello6_dev), MINOR(hello6_dev),
		MAJOR(hello6_dev), MINOR(hello6_dev) + DEV_MAX - 1);

	hello6_cls = class_create(DRV_NAME);
	if (IS_ERR(hello6_cls)) {
		ret = PTR_ERR(hello6_cls);
		pr_err("class_create 失败: %d\n", ret);
		goto err_chr;
	}

	fake_dev[0] = platform_device_register_simple(DRV_NAME, 0, NULL, 0);
	if (IS_ERR(fake_dev[0])) {
		ret = PTR_ERR(fake_dev[0]);
		pr_err("建设备0失败: %d\n", ret);
		goto err_class;
	}

	fake_dev[1] = platform_device_register_simple(DRV_NAME, 1, NULL, 0);
	if (IS_ERR(fake_dev[1])) {
		ret = PTR_ERR(fake_dev[1]);
		pr_err("建设备1失败: %d\n", ret);
		goto err_dev0;
	}

	/* ★★★ 你漏掉的就是这一步 */
	ret = platform_driver_register(&hello6_driver);
	if (ret) {
		pr_err("注册驱动失败: %d\n", ret);
		goto err_dev1;
	}

	pr_info("模块就绪\n");
	return 0;                    

err_dev1:
	platform_device_unregister(fake_dev[1]);
err_dev0:
	platform_device_unregister(fake_dev[0]);
err_class:
	class_destroy(hello6_cls);
err_chr:
	unregister_chrdev_region(hello6_dev, DEV_MAX);   
	return ret;
}

static __exit void hello6_exit(void)
{
	platform_device_unregister(fake_dev[1]);
	platform_device_unregister(fake_dev[0]);
	platform_driver_unregister(&hello6_driver);
	class_destroy(hello6_cls);
	unregister_chrdev_region(hello6_dev, DEV_MAX);    /* ★ 正确函数 */
}

module_init(hello6_init);
module_exit(hello6_exit);
MODULE_VERSION("v1.0.0");
MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("h-000-h");
MODULE_DESCRIPTION("probe 一个主设备号管理多个设备");
```

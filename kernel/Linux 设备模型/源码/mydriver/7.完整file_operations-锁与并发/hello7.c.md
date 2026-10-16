<!-- 源：/home/h-000-h/project/mydriver/7/hello7.c -->

```c
#define pr_fmt(fmt) KBUILD_MODNAME ":" fmt
#include <linux/module.h>
#include <linux/init.h>
#include <linux/platform_device.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/kdev_t.h>
#include <linux/uaccess.h>
#include <linux/slab.h>
#include <linux/mutex.h>
#include <linux/ioctl.h>
#include <linux/kernel.h>
#include <linux/printk.h>

#define DRV_NAME "hello7"
#define DEV_MAX   2
#define BUF_MAX   128

/*
 * 编译期锁开关（A/B 对照验证用）：
 *   ./build.sh TEST_MUTEX=1  → 带 mutex（正常行为）
 *   ./build.sh TEST_MUTEX=0  → 无锁（不传参数同样无锁）
 * 无锁版仅用于观察竞态撕裂，勿用于生产。
 *
 * 注意：内核编译启用了 -Wundef，未定义宏参与 #if 会告警，
 * 因此给默认值兜底（不传参数时 = 0，即无锁）。
 */
#ifndef TEST_MUTEX
#define TEST_MUTEX 0
#endif

/* 自定义 ioctl 命令：魔术数 'H' + 序号 0 */
#define HELLO_IOC_MAGIC  'H'
#define HELLO_IOCRESET   _IO(HELLO_IOC_MAGIC, 0)

static dev_t hello7_devt;
static struct class* phello7_cls; 
static struct platform_device *pfake_dev[DEV_MAX];

static struct hello7_priv
{
    struct cdev     cdev;
    struct device * pdev;
    dev_t           devt;
    struct mutex    mutex_lock;
    char            buf[BUF_MAX];
    size_t          buf_len;       
};

static int hello7_open(struct inode*pinode,struct file*pfile)
{
    /* 从 inode 里的 cdev 反查出它所属的 priv */
    struct hello7_priv* priv = container_of(pinode->i_cdev,struct hello7_priv,cdev);
    pfile->private_data = priv;/* 之后的 read/write 都靠它拿 priv */
    pr_info("open %d:%d\n", MAJOR(priv->devt), MINOR(priv->devt));
	return 0;
}

static ssize_t hello7_read(struct file *pfile,char * __user pbuf , size_t len ,loff_t* poff)
{
    struct hello7_priv* priv = pfile->private_data;
    size_t n;
    int ret;
#if TEST_MUTEX
    if(mutex_lock_interruptible(&priv->mutex_lock))
        return -ERESTARTSYS;
#endif
    if(*poff >= priv->buf_len)
    {
        ret = 0;
        goto out;
    }
    n = min_t(size_t,len,priv->buf_len);
    if(copy_to_user(pbuf,priv->buf,n))
    {
        ret = -EFAULT;
        goto out;
    }
    *poff += n;
    ret = n;
out:
#if TEST_MUTEX
    mutex_unlock(&priv->mutex_lock);
#endif
    return ret;
}

static ssize_t hello7_write(struct file* pfile,const char __user *pbuf, size_t len,loff_t*poff)
{
    struct hello7_priv* priv = pfile->private_data;
    int ret;
    if(len > BUF_MAX)
        len = BUF_MAX -1;/* 截断，留一个位置给 '\0' */
#if TEST_MUTEX
    if(mutex_lock_interruptible(&priv->mutex_lock))
        return -ERESTARTSYS;
#endif
    if(copy_from_user(priv->buf,pbuf,len))
    {
        ret = -EFAULT;
        goto out;
    }

    priv->buf[len] = '\0';
    priv->buf_len = len;
    ret = len;
out:
#if TEST_MUTEX
    mutex_unlock(&priv->mutex_lock);
#endif
    return ret;
}

static loff_t hello7_llseek(struct file *pfile, loff_t off, int whence)
{
	struct hello7_priv *priv = pfile->private_data;
	loff_t newopos;

	switch (whence) {
	case SEEK_SET:
		newopos = off;
		break;
	case SEEK_CUR:
		newopos = pfile->f_pos + off;
		break;
	case SEEK_END:
		newopos = priv->buf_len + off;
		break;
	default:
		return -EINVAL;
	}                                 

	if (newopos < 0)                  
		return -EINVAL;

	pfile->f_pos = newopos;           
	return newopos;
}


static long hello7_ioctl(struct file* pfile, unsigned int cmd,unsigned long arg)
{
    struct hello7_priv *priv = pfile->private_data;
    if (_IOC_TYPE(cmd) != HELLO_IOC_MAGIC)
        return -ENODEV;
    switch (cmd) 
    {
    case HELLO_IOCRESET:
#if TEST_MUTEX
        mutex_lock(&priv->mutex_lock);
#endif
        priv->buf_len = 0;
#if TEST_MUTEX
        mutex_unlock(&priv->mutex_lock);
#endif
        pr_info("ioctl: buffer 已清空\n");
        break;
	default:
		return -EINVAL;
    }
    return 0;
}

static int hello7_release(struct inode *pinode,struct file*pfile)
{
    pr_info("release\n");
	return 0;
}

const static struct file_operations hello7_fops=
{
	.owner		= THIS_MODULE,
	.open		= hello7_open,
	.read		= hello7_read,
	.write		= hello7_write,
	.unlocked_ioctl	= hello7_ioctl,
	.release	= hello7_release,
    .llseek     = hello7_llseek,
};

static int hello7_probe(struct platform_device*pdev)
{
    struct hello7_priv* priv;
    int idx = pdev->id;
    int ret;

    if(idx < 0 || idx >= DEV_MAX)
    {
        dev_err(&pdev->dev, "id %d 超出范围\n", idx);
        ret = -EINVAL;
        goto err_final;
    }

    priv = kzalloc(sizeof(*priv), GFP_KERNEL);
	if (!priv)
    {
        ret = -ENOMEM;
		goto err_final;
    }

#if TEST_MUTEX
    mutex_init(&priv->mutex_lock);
#endif
    priv->devt = MKDEV(MAJOR(hello7_devt), MINOR(hello7_devt)+idx);

    cdev_init(&priv->cdev,&hello7_fops);
    priv->cdev.owner = THIS_MODULE;
    ret = cdev_add(&priv->cdev,priv->devt,1);
    if(ret)
    {
        dev_err_probe(&pdev->dev, ret, "cdev_add 失败\n");
		goto err_mem;
    }

    priv->pdev = device_create(phello7_cls,NULL,priv->devt,priv,"%s%d", DRV_NAME, idx);
    if (IS_ERR(priv->pdev)) 
    {
        ret = PTR_ERR(priv->pdev);
		dev_err_probe(&pdev->dev, ret, "device_create 失败\n");
		goto err_cdev;
    }

    platform_set_drvdata(pdev,priv);
    pr_info("probe ok, devt=%d:%d\n", MAJOR(priv->devt), MINOR(priv->devt));
	return 0;
err_cdev:
    cdev_del(&priv->cdev);
err_mem:
    kfree(priv);
    priv =NULL;
    pr_err("%s err mem has been free",DRV_NAME);
err_final:
    return ret;
}

static void hello7_remove(struct platform_device *pdev)
{
    struct hello7_priv *priv = platform_get_drvdata(pdev);
    if (!priv) 
    {
		pr_err("拿不到 priv\n");
		return;
	}
    device_destroy(phello7_cls,priv->devt);
    cdev_del(&priv->cdev);
    kfree(priv);
    priv =NULL;
	pr_info("remove ok\n");
}

static struct platform_driver hello7_driver=
{
    .probe =hello7_probe,
    .remove = hello7_remove,
    .driver=
    {
        .name = DRV_NAME,
    },
};

static int __init hello7_init(void)
{
    int ret;
    ret = alloc_chrdev_region(&hello7_devt,0,DEV_MAX,DRV_NAME);
    pr_info("占号 %d:%d .. %d:%d\n", MAJOR(hello7_devt), MINOR(hello7_devt),MAJOR(hello7_devt), MINOR(hello7_devt) + DEV_MAX - 1);
    phello7_cls = class_create(DRV_NAME);
    if (IS_ERR(phello7_cls)) 
    {
        ret = PTR_ERR(phello7_cls);
        pr_err("class_create 失败: %d\n", ret);
		goto err_chr;
    }

    pfake_dev[0] = platform_device_register_simple(DRV_NAME, 0, NULL, 0);
    if (IS_ERR(pfake_dev[0])) 
    {
		ret = PTR_ERR(pfake_dev[0]);
		pr_err("建设备0失败: %d\n", ret);
		goto err_class;
	}

	pfake_dev[1] = platform_device_register_simple(DRV_NAME, 1, NULL, 0);
	if (IS_ERR(pfake_dev[1])) 
    {
		ret = PTR_ERR(pfake_dev[1]);
		pr_err("建设备1失败: %d\n", ret);
		goto err_dev0;
	}

    ret =platform_driver_register(&hello7_driver);
    if (ret) 
    {
		pr_err("注册驱动失败: %d\n", ret);
		goto err_dev1;
	}

	pr_info("模块就绪\n");
	return 0;

err_dev1:
	platform_device_unregister(pfake_dev[1]);
err_dev0:
	platform_device_unregister(pfake_dev[0]);
err_class:
	class_destroy(phello7_cls);
err_chr:
    unregister_chrdev_region(hello7_devt, DEV_MAX);
    return ret;
}

static void __exit hello7_exit(void)
{
	platform_device_unregister(pfake_dev[1]);
	platform_device_unregister(pfake_dev[0]);
	platform_driver_unregister(&hello7_driver);
	class_destroy(phello7_cls);
	unregister_chrdev_region(hello7_devt, DEV_MAX);
}

module_init(hello7_init);
module_exit(hello7_exit);
MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("h-000-h");
MODULE_DESCRIPTION("完整 file_operations");
MODULE_VERSION("v1.0.0");
```

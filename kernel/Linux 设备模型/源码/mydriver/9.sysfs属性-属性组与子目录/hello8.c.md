<!-- 源：/home/h-000-h/project/mydriver/8/hello8.c -->

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
#include <linux/sysfs.h>
#include <linux/kstrtox.h>   /* kstrtobool */
#define DRV_NAME "hello8"
#define DEV_MAX   2
#define BUF_MAX   128

/*
 * 编译期锁开关（A/B 对照验证用）：
 *   ./build.sh TEST_MUTEX=1  → 带 mutex（正常行为）
 *   ./build.sh TEST_MUTEX=0  → 无锁（不传参数同样无锁）
 * 无锁版仅用于观察竞态撕裂，勿用于生产。
 *
 * 注意：内核编译启用了 -Wundef，未定义宏参与 #if 会告警，
 * 因此给默认值兜底（不传参数时 = 0，即无锁）
 * 在 /sys/class/hello8/hello80/ 下暴露几个属性文件，用户态用 cat/echo 就能查看状态、下发命令，不用再写 ioctl。
 */
#ifndef TEST_MUTEX
#define TEST_MUTEX 0
#endif
#ifndef DIR_ATTR
#define DIR_ATTR 0
#endif
/* 自定义 ioctl 命令：魔术数 'H' + 序号 0 */
#define HELLO_IOC_MAGIC  'H'
#define HELLO_IOCRESET   _IO(HELLO_IOC_MAGIC, 0)

static dev_t hello8_devt;
static struct class* phello8_cls; 
static struct platform_device *pfake_dev[DEV_MAX];

static struct hello8_priv
{
    struct cdev     cdev;
    struct device * pdev;
    dev_t           devt;
    struct mutex    mutex_lock;
    char            buf[BUF_MAX];
    size_t          buf_len;       
};

static int hello8_open(struct inode*pinode,struct file*pfile)
{
    /* 从 inode 里的 cdev 反查出它所属的 priv */
    struct hello8_priv* priv = container_of(pinode->i_cdev,struct hello8_priv,cdev);
    pfile->private_data = priv;/* 之后的 read/write 都靠它拿 priv */
    pr_info("open %d:%d\n", MAJOR(priv->devt), MINOR(priv->devt));
	return 0;
}

static ssize_t hello8_read(struct file *pfile,char * __user pbuf , size_t len ,loff_t* poff)
{
    struct hello8_priv* priv = pfile->private_data;
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
    n = min_t(size_t,len,priv->buf_len-*poff);
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

static ssize_t hello8_write(struct file* pfile,const char __user *pbuf, size_t len,loff_t*poff)
{
    struct hello8_priv* priv = pfile->private_data;
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

static loff_t hello8_llseek(struct file *pfile, loff_t off, int whence)
{
	struct hello8_priv *priv = pfile->private_data;
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


static long hello8_ioctl(struct file* pfile, unsigned int cmd,unsigned long arg)
{
    struct hello8_priv *priv = pfile->private_data;
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

static int hello8_release(struct inode *pinode,struct file*pfile)
{
    pr_info("release\n");
	return 0;
}

const static struct file_operations hello8_fops=
{
	.owner		= THIS_MODULE,
	.open		= hello8_open,
	.read		= hello8_read,
	.write		= hello8_write,
	.unlocked_ioctl	= hello8_ioctl,
	.release	= hello8_release,
    .llseek     = hello8_llseek,
};

/*=================前置============================*/
/*
在 /sys/class/hello7/hello70/ 下暴露几个属性文件，用户态用 cat/echo 就能查看状态、下发命令，不用再写 ioctl。
DEVICE_ATTR_RO/WO/RW(name)	一组宏，自动生成 name_show/name_store 函数对和 struct device_attribute。权限分别是 0444 / 0200 / 0644
show/store 约定	show 返回写进 buf 的字节数；store 返回 count（表示全部消费）或负 errno
生命周期	属性属于某个 struct device，所以必须在 device_create() 之后挂载，在 device_destroy() 之前摘除
/sys/class/hello8/hello80/
├── buf_len     只读，当前缓冲区长度
├── content     只读，当前缓冲区内容
└── reset       只写，echo 1 清空缓冲区

*/
/*

*/
/* ================= sysfs 属性 ================= */
static ssize_t buf_len_show(struct device *pdev, struct device_attribute *pattr, char *pbuf)
{
	struct hello8_priv *priv = dev_get_drvdata(pdev);
	size_t len;

#if TEST_MUTEX
	mutex_lock(&priv->mutex_lock);
#endif
	len = priv->buf_len;
#if TEST_MUTEX
	mutex_unlock(&priv->mutex_lock);
#endif

	return sysfs_emit(pbuf, "%zu\n", len);   /* ★ 返回写入的字节数 */
}
static DEVICE_ATTR_RO(buf_len);

static ssize_t content_show(struct device *pdev,struct device_attribute *pattr,char*pbuf)
{
    struct hello8_priv* priv = dev_get_drvdata(pdev);
    int ret;
#if TEST_MUTEX
    if (mutex_lock_interruptible(&priv->mutex_lock))
    {
        return -ERESTARTSYS;
    }
#endif
    ret = sysfs_emit(pbuf,"%.*s",(int)priv->buf_len,priv->buf);
#if TEST_MUTEX
    mutex_unlock(&priv->mutex_lock);
#endif
    return ret;
}
static DEVICE_ATTR_RO(content);

static ssize_t reset_store(struct device*pdev,struct device_attribute*pattr,const char*pbuf,size_t count)
{
    struct hello8_priv*priv = dev_get_drvdata(pdev);
    bool val;
    if(kstrtobool(pbuf,&val))/*解析用户输入用 kstrto*/
        return -EINVAL;
    if(!val)
        return count; /* 用户写了 0，什么都不做，但要"消费"掉输入 */

#if TEST_MUTEX       
    mutex_lock(&priv->mutex_lock);
#endif
    priv->buf_len=0;
    priv->buf[0]='\0';
#if TEST_MUTEX
	mutex_unlock(&priv->mutex_lock);
#endif
    return count; 
}
static DEVICE_ATTR_WO(reset);

/*=====================可见性===============================*/
#if TEST_MUTEX
static umode_t my_is_visible(struct kobject *pkobj, struct attribute *pattr, int n)
{
	return pattr->mode;        /* ★ 参数是 pattr；返回 0 表示隐藏该项 */
}
#endif

#if DIR_ATTR
/* ---------- 模式 A：属性放进子目录 ---------- */
static struct attribute *hello8_debug_attrs[] = 
{
	&dev_attr_buf_len.attr,
	&dev_attr_content.attr,
	NULL,
};

static struct attribute_group hello8_debug_group = 
{
	.name  = "debug",
	.attrs = hello8_debug_attrs,
};

static struct attribute *hello8_ctrl_attrs[] = 
{
	&dev_attr_reset.attr,
	NULL,
};                                   

static struct attribute_group hello8_ctrl_group = 
{
	.name  = "ctrl",
	.attrs = hello8_ctrl_attrs,
#if TEST_MUTEX
	.is_visible = my_is_visible,          /* ★ is_visible 挂在"组"上 */
#endif
};

static const struct attribute_group *hello8_groups[] = 
{
	&hello8_debug_group,
	&hello8_ctrl_group,
	NULL,
};

#else
static struct attribute *hello8_attrs[]=
{
    &dev_attr_buf_len.attr,
    &dev_attr_content.attr,
    &dev_attr_reset.attr,
    NULL,
};
//ATTRIBUTE_GROUPS 宏只会生成“平铺且无 is_visible”的组所以is_visible不能进这里
ATTRIBUTE_GROUPS(hello8);        /* 生成 hello8_group 和 hello8_groups */
#endif
int hello8_probe(struct platform_device*pdev)
{
    struct hello8_priv* priv;
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
    priv->devt = MKDEV(MAJOR(hello8_devt), MINOR(hello8_devt)+idx);

    cdev_init(&priv->cdev,&hello8_fops);
    priv->cdev.owner = THIS_MODULE;
    ret = cdev_add(&priv->cdev,priv->devt,1);
    if(ret)
    {
        dev_err_probe(&pdev->dev, ret, "cdev_add 失败\n");
		goto err_mem;
    }
/*====================================================================================================*/
//加父
// class/hello8/hello80🔒 
// ❯ ls -l
// drwxr-xr-x    - root 16 10月 20:55  ctrl
// drwxr-xr-x    - root 16 10月 20:55  debug
// lrwxrwxrwx    - root 16 10月 20:55  device -> ../../../hello8.0
// drwxr-xr-x    - root 16 10月 20:55  power
// lrwxrwxrwx    - root 16 10月 20:55  subsystem -> ../../../../../class/hello8
// .r--r--r-- 4.1k root 16 10月 20:55 󰡯 dev
// .rw-r--r-- 4.1k root 16 10月 20:55 󰡯 uevent

//不加父
// drwxr-xr-x    - root 16 10月 20:47  ctrl
// drwxr-xr-x    - root 16 10月 20:47  debug
// drwxr-xr-x    - root 16 10月 20:47  power
// lrwxrwxrwx    - root 16 10月 20:47  subsystem -> ../../../../class/hello8
// .r--r--r-- 4.1k root 16 10月 20:47 󰡯 dev
// .rw-r--r-- 4.1k root 16 10月 20:47 󰡯 uevent
/*====================================================================================================*/
    // priv->pdev = device_create(phello8_cls,NULL,priv->devt,priv,"%s%d", DRV_NAME, idx);
    /*此时改为parent就有device(用platform先当父亲) 可知内核创建这条链接的代码是有条件的（device_add_class_symlinks()）：有父亲就创建没有就不创建*/
    priv->pdev = device_create(phello8_cls, &pdev->dev, priv->devt, priv, "%s%d", DRV_NAME, idx);
    if (IS_ERR(priv->pdev)) 
    {
        ret = PTR_ERR(priv->pdev);
		dev_err_probe(&pdev->dev, ret, "device_create 失败\n");
		goto err_cdev;
    }
/*================================SYS ATTR================================================*/
    ret = device_add_groups(priv->pdev,hello8_groups);
    if(ret)
    {
        dev_err_probe(&pdev->dev,ret, "添加 sysfs 属性失败\n");
        goto err_dev;
    }
/*========================================================================================*/
    platform_set_drvdata(pdev,priv);
    pr_info("probe ok, devt=%d:%d\n", MAJOR(priv->devt), MINOR(priv->devt));
	return 0;
err_dev:
    device_destroy(phello8_cls,hello8_devt);
err_cdev:
    cdev_del(&priv->cdev);
err_mem:
    kfree(priv);
    priv =NULL;
    pr_err("%s err mem has been free",DRV_NAME);
err_final:
    return ret;
}

static void hello8_remove(struct platform_device *pdev)
{
    struct hello8_priv *priv = platform_get_drvdata(pdev);
    if (!priv) 
    {
		pr_err("拿不到 priv\n");
		return;
	}
    device_remove_groups(priv->pdev, hello8_groups); 
    device_destroy(phello8_cls,priv->devt);
    cdev_del(&priv->cdev);
    kfree(priv);
    priv =NULL;
	pr_info("remove ok\n");
}

static struct platform_driver hello8_driver=
{
    .probe =hello8_probe,
    .remove = hello8_remove,
    .driver=
    {
        .name = DRV_NAME,
    },
};

static int __init hello8_init(void)
{
    int ret;
    ret = alloc_chrdev_region(&hello8_devt,0,DEV_MAX,DRV_NAME);
    pr_info("占号 %d:%d .. %d:%d\n", MAJOR(hello8_devt), MINOR(hello8_devt),MAJOR(hello8_devt), MINOR(hello8_devt) + DEV_MAX - 1);
    phello8_cls = class_create(DRV_NAME);
    if (IS_ERR(phello8_cls)) 
    {
        ret = PTR_ERR(phello8_cls);
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

    ret =platform_driver_register(&hello8_driver);
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
	class_destroy(phello8_cls);
err_chr:
    unregister_chrdev_region(hello8_devt, DEV_MAX);
    return ret;
}

static void __exit hello8_exit(void)
{
	platform_device_unregister(pfake_dev[1]);
	platform_device_unregister(pfake_dev[0]);
	platform_driver_unregister(&hello8_driver);
	class_destroy(phello8_cls);
	unregister_chrdev_region(hello8_devt, DEV_MAX);
}

module_init(hello8_init);
module_exit(hello8_exit);
MODULE_LICENSE("Dual BSD/GPL");
MODULE_AUTHOR("h-000-h");
MODULE_DESCRIPTION("sys 属性添加");
MODULE_VERSION("v1.0.0");
```

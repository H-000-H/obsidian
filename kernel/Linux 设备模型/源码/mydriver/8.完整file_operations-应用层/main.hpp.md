<!-- 源：/home/h-000-h/project/mydriver/7/main.hpp -->

```cpp
#ifndef MAIN_HPP
#define MAIN_HPP
#ifdef __KERNEL__
#include <linux/ioctl.h>
#else
#include <sys/ioctl.h>
#endif
#define HELLO_IOC_MAGIC 'H'
#define HELLO_IOCRESET _IO(HELLO_IOC_MAGIC,0)

#endif
```

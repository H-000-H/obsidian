<!-- 源：/home/h-000-h/project/mydriver/Makefile -->

```makefile
KDIR ?= /lib/modules/$(shell uname -r)/build #软链接到 /usr/src/linux-headers-7.0.0-38-generic换版本一个字不需要换
#uname -r 内核版本号 

#记住"你的模块源码在哪",待会儿要传给内核 Makefile
PWD := $(shell pwd)

#-C $(KDIR) = 切到内核目录再执行(由内核的顶层 Makefile 接管);
# M=$(PWD) = 告诉内核 Makefile:"模块源码在 PWD,编完把产物放回这里";
# modules = 内核 Makefile 的目标(对应的还有 clean);
# 用 $(MAKE) 而不是 make = 传递命令行参数(这样你 make V=1 看详细编译过程才生效);
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

#cd /home/h-000-h/project/mydriver
# python3 /usr/src/linux-headers-$(uname -r)/scripts/clang-tools/gen_compile_commands.py \
#     -d /home/h-000-h/project/mydriver \
#     -o /home/h-000-h/project/mydriver/compile_commands.json \
#     /home/h-000-h/project/mydriver
# -d/--directory	写进 JSON 的 directory 字段(编译时的工作目录)
# -o/--output	输出路径,默认 compile_commands.json
# 位置参数(paths)	去哪找 .cmd 文件;给目录就递归扫
```

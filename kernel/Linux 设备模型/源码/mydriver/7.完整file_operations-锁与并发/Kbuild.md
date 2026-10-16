<!-- 源：/home/h-000-h/project/mydriver/7/Kbuild -->

```makefile
obj-m += hello7.o

# 编译期锁开关：
#   make TEST_MUTEX=1 → ccflags 加 -DTEST_MUTEX=1（带锁）
#   不传或 TEST_MUTEX=0 → 不加 -D（无锁，#if TEST_MUTEX 求值 0）
ifdef TEST_MUTEX
ccflags-y += -DTEST_MUTEX=$(TEST_MUTEX)
endif
```

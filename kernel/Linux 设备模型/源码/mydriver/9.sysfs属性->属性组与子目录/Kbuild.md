<!-- 源：/home/h-000-h/project/mydriver/8/Kbuild -->

```makefile
obj-m +=hello8.o
ifdef TEST_MUTEX
ccflags-y += -DTEST_MUTEX=$(TEST_MUTEX)
endif
ifdef DIR_ATTR
ccflags-y += -DDIR_ATTR=$(DIR_ATTR)
endif
```

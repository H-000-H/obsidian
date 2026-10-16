<!-- 源：/home/h-000-h/project/mydriver/build.sh -->

```bash
#! /usr/bin/env bash
# 用法：
#   ./build.sh                无锁构建（#if TEST_MUTEX 求值 0）
#   ./build.sh TEST_MUTEX=1   带锁构建
#   ./build.sh TEST_MUTEX=0   显式无锁构建
# 其他 make 变量同理透传，如 ./build.sh V=1 看详细编译过程
make clean
make -w "$@"
python3 /usr/src/linux-headers-$(uname -r)/scripts/clang-tools/gen_compile_commands.py \
    -d "$PWD" -o "$PWD/compile_commands.json" "$PWD"
```

# 作用域枚举、constexpr、多态与 GCC 扩展

全部标准语法最小示例，能读懂用法即可。

## 一、作用域枚举 `enum class`

### 1.1 旧式 enum 的两个坑
```cpp
enum Color { red, green };      // 名字 red 污染全局
enum Other  { red };            // 冲突！
int x = green;                  // 竟能隐式转成整数
```

### 1.2 enum class 修好这两点
```cpp
enum class Color { red, green };
enum class Status { ok, error };

// Status::ok 不会和 Color::red 打架
// Color c = 0;                 // 错误：不能隐式转整数
int x = static_cast<int>(Color::red);        // 拿整数要显式转换
Color c = static_cast<Color>(3);             // 整型回来也要显式
```

## 二、constexpr：编译期就算出来

### 2.1 constexpr 常量 / 函数
```cpp
constexpr int kMax = 100;                  // 编译期常量，能定数组大小
int arr[kMax];

constexpr int square(int x) { return x * x; }  // constexpr 函数
int v = square(5);                             // 编译期就算出 25
```

### 2.2 inline 变量：头文件里定义不报重复
C++17 前，`const/static` 在头文件里定义会"重复定义"报错；`inline` 允许头文件直接定义：
```cpp
// mycfg.hpp（被多个 .cpp include）
inline constexpr int kTimeoutMs = 1000;   // 放心写在头文件
```

## 三、多态：纯虚函数 + override / final

### 3.1 纯虚 function 交约定，subclass 实现
```cpp
class Shape
{
public:
    virtual ~Shape() = default;              // 基类析构要 virtual
    virtual double area() const = 0;         // 纯虚：约定"都要能算面积"
};

class Circle : public Shape
{
public:
    double area() const override { return 3.14 * m_r * m_r; }  // override
private:
    double m_r = 1.0;
};

void print(const Shape& s) { std::cout << s.area(); }   // 传啥都按自己实现算
```

### 3.2 override 与 final
- `override`：声明"我在覆写基类函数"——写错签名编译器立刻报错，而不是悄悄新开一个。
- `final`：声明"到此不能再被覆写"。

```cpp
class Special final : public Circle { };   // final：不能再被继承
```

## 四、using 类型别名

```cpp
using Callback = void (*)(int, void*);          // 函数指针别名
using MyVector = std::vector<int>;              // 容器别名
template <typename T> using OwnPtr = std::unique_ptr<T>;  // 模板别名
```

## 五、前向声明与匿名命名空间

### 5.1 前向声明：只用"类型存在"，不用 include
```cpp
struct Device;     // 只声明，别 include 大头部文件
class Led { Device* m_dev; };   // 用指针/引用即可
```

### 5.2 匿名命名空间：仅本文件可见
```cpp
namespace {              // 仅本翻译单元可见
    const int kFactor = 3;
    void helper() {}
}
```

## 六、GCC 扩展与内建函数

GCC / Clang 可用以下语法辅助嵌入式优化。

### 6.1 属性 __attribute__
```cpp
__attribute__((always_inline)) int fast(void);   // 强制内联
struct Foo { } __attribute__((aligned(4)));       // 结构体 4 字节对齐
__attribute__((used)) int entry(void);            // 强制保留，避免被丢弃
```

### 6.2 内建位运算：__builtin_ctz / __builtin_popcount
```cpp
__builtin_ctz(0b1000);         // 尾随 0 的个数 = 3（最小置位位位置）
__builtin_popcount(0b1101);    // 置位个数 = 3

// 位图分配最小可用位置，常用套路
int nextFree(unsigned used) { return __builtin_ctz(~used) + 1; }
```

## 七、C/C++ 混编：extern "C"

C 和 C++ 函数名可兼容器 (mangle) 规则不同。把 C 符号用 `extern "C"` 包起来即不会有链接对不上：
```cpp
extern "C" void system_init();          // C++ 里声明要按 C 链接规则找
#ifdef __cplusplus
extern "C" {
#endif
void c_impl(void);
#ifdef __cplusplus
}
#endif
```

## 关联
- [[C 语言高级用法]]
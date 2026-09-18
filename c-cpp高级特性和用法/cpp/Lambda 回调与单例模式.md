# Lambda、回调与单例模式

全部标准 C++ 最小示例，能读懂用法即可。

## 一、Lambda：随手写个"函数对象"

### 1.1 基本形态
```cpp
auto f = [](int a, int b) { return a + b; };   // [捕获] (参数) { 函数体 }
int r = f(1, 2);                               // 3
```

### 1.2 捕获：lambda 能不能用外部变量
```cpp
int base = 10;
auto add = [base](int x) { return x + base; };   // 拷贝捕获：外部 base 只有只读副本
int counter = 0;
auto inc = [&counter]() { ++counter; };          // 引用捕获：能改外部 counter
```

- `[x]`：拷贝带进来；`[&x]`：引用带进来（能改）。
- `[=]` / `[&]`：把所有外部变量按拷贝 / 按引用带进来。

### 1.3 易错点：按值捕获的变量无法修改
```cpp
int n = 0;
auto a = [n]() { n++; };        // 报错：n 是只读副本
auto b = [n]() mutable { n++; }; // 加 mutable 才允许改这份副本（不影响外部 n）
```

## 二、回调：把"做什么"交给调用方

### 2.1 是什么
把一个函数当作参数传进另一个函数，让对方在合适时机调它，常用于"事件发生后通知我"。

### 2.2 三种写法
写法 A：函数指针（C 风格）
```cpp
void onPress() { std::cout << "按下\n"; }
void registerHandler(void (*cb)()) { cb(); }   // 存起来按需调用
registerHandler(onPress);
```

写法 B：`std::function`（类型擦除，能装 lambda/成员函数）
```cpp
std::function<void()> onEvent;
onEvent = []() { std::cout << "事件\n"; };
onEvent();                                     // 统一调用
```

写法 C：模板 + 万能引用（零开销、支持完美转发）
```cpp
template <typename Callback>
void registerHandler(Callback&& cb)
{
    auto f = std::forward<Callback>(cb);   // 原样保留，不拷贝
    f();
}
registerHandler([]{ std::cout << "hi\n"; });
```

## 三、单例模式：整个程序只有一个实例

### 3.1 最简洁写法：函数内 `static` 局部对象
```cpp
class Logger
{
public:
    static Logger& getInstance()
    {
        static Logger instance;   // static 局部对象：只构造一次
        return instance;
    }
    void log(const std::string& s) { /* ... */ }

private:
    Logger() {}                        // 构造私有：别人 new 不了
    Logger(const Logger&) = delete;    // 拷贝禁止
    Logger& operator=(const Logger&) = delete;
};

Logger::getInstance().log("hello");    // 每次拿的都是同一个
```

要点：C++11 起，函数内 `static` 局部变量的初始化自带线程安全，不用自己加锁；构造私有 + 拷贝删除封死了创建副本的途径。

### 3.2 模板化：让每个子类自动单例
```cpp
template <typename T>
class SingletonBase
{
public:
    static T& instance()
    {
        static T obj;          // 每个 T 独立一份
        return obj;
    }
};

class Config : public SingletonBase<Config> { /* ... */ };
Config::instance();            // 只有一份 Config
```

## 四、`= delete` / `= default` / `explicit`

### 4.1 `= delete`：禁止某操作
```cpp
class A
{
public:
    A(int x) : v(x) {}
    A(const A&) = delete;               // 禁止拷贝构造
    A& operator=(const A&) = delete;    // 禁止拷贝赋值
    int v;
};
A a(1);
A b = a;     // 编译错误
```

### 4.2 `= default`：明确"用编译器默认的"
```cpp
class B
{
public:
    B() = default;        // 明确使用默认构造
    ~B() = default;       // 明确使用默认析构
};
```

### 4.3 `explicit`：禁止单参构造的隐式转换
```cpp
class Celsius
{
public:
    explicit Celsius(double deg) : m_deg(deg) {}
private:
    double m_deg;
};
Celsius c = 30.0;   // 错误：explicit 禁止隐式转换
Celsius c2(30.0);   // 正确：显式构造
```

## 关联
- [[类型擦除与轻量 RTTI]]
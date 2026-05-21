# 05 STL 够用就行

> STL = Standard Template Library，C++ 标准库。
> 控制代码里**只需要 4 个东西**：`vector`、`array`、`string`、范围 for。

---

## 1. `std::vector`：会自动变大的数组

```cpp
#include <vector>

std::vector<int> v;          // 空数组
v.push_back(10);             // 尾部加元素 → [10]
v.push_back(20);             // → [10, 20]
v.push_back(30);             // → [10, 20, 30]

std::cout << v.size();       // 3
std::cout << v[0];           // 10
v[0] = 99;                   // 改元素
```

**和 C 数组的区别：**

| 特性 | C 数组 `int a[10]` | `std::vector<int>` |
|------|-------------------|---------------------|
| 大小 | 固定 | 动态增长 |
| 自动释放 | 否（栈数组自动；malloc 要 free） | ✅ 自动 |
| 边界检查 | 没有 | `at()` 有；`[]` 没有 |
| 知道自己多大 | ❌ | ✅ `size()` |

**初始化方式：**
```cpp
std::vector<int> v1;                    // 空
std::vector<int> v2(5);                 // 5 个元素，都是 0
std::vector<int> v3(5, 100);            // 5 个元素，都是 100
std::vector<int> v4 = {1, 2, 3, 4};     // 列表初始化
std::vector<float> samples = {0.1f, 0.2f, 0.3f};
```

**遍历方式：**
```cpp
std::vector<float> samples = {1.0f, 2.0f, 3.0f};

// 方式 1：传统 for + 下标
for (int i = 0; i < samples.size(); i++) {
    std::cout << samples[i] << '\n';
}

// 方式 2：范围 for（推荐）
for (float x : samples) {
    std::cout << x << '\n';
}

// 方式 3：要修改元素时用引用
for (float& x : samples) {
    x *= 2.0f;
}

// 方式 4：只读时加 const
for (const float& x : samples) {
    std::cout << x;
}
```

**控制代码场景：采样平均**
```cpp
std::vector<float> readings;
for (int i = 0; i < 100; i++) {
    readings.push_back(read_sensor());
}

float sum = 0;
for (float r : readings) sum += r;
float avg = sum / readings.size();
```

> ⚠️ **嵌入式注意**：`vector` 用动态内存（堆），很多 MCU 项目禁用堆。这种场景用下面的 `std::array`。

---

## 2. `std::array`：固定大小数组（嵌入式友好）

```cpp
#include <array>

std::array<float, 4> motor_speeds = {0, 0, 0, 0};   // 4 个 float
motor_speeds[0] = 5.0f;
std::cout << motor_speeds.size();    // 4
```

**和 C 数组 `float a[4]` 比好在哪？**
- 知道自己多大（`size()`）
- 能整体赋值、拷贝、传引用
- 不会退化成指针

**常见用法（机器人）：**
```cpp
struct Quadruped {
    std::array<float, 4> joint_angles;
    std::array<float, 4> motor_torques;
};

void apply_torques(const std::array<float, 4>& torques) {
    for (size_t i = 0; i < torques.size(); i++) {
        set_motor_torque(i, torques[i]);
    }
}
```

---

## 3. `std::string`：会变长的字符串

```cpp
#include <string>

std::string name = "left_wheel";
name += "_motor";                    // "left_wheel_motor"

std::cout << name.length();          // 16
std::cout << name[0];                // 'l'

// 比较
if (name == "left_wheel_motor") { /* ... */ }

// 拼数字（C++20 之前用 stringstream，C++20 用 std::format）
std::string log = "Motor " + std::to_string(0) + " speed=" + std::to_string(5.5f);
```

**比 C 字符串 `char[]` 好在哪？**
- 不用考虑长度上限和 `'\0'`
- `+` 直接拼接
- `==` 直接比较

> ⚠️ 嵌入式裸机也很少用 `std::string`（用堆）。但 ROS、上位机里非常常用。

---

## 4. `auto` + 范围 for：神器组合

```cpp
std::vector<std::pair<std::string, float>> motor_data = {
    {"left",  5.0f},
    {"right", -3.0f}
};

// 不用 auto 的版本（类型超长）
for (const std::pair<std::string, float>& p : motor_data) {
    std::cout << p.first << ": " << p.second << '\n';
}

// 用 auto 的版本（简洁）
for (const auto& p : motor_data) {
    std::cout << p.first << ": " << p.second << '\n';
}
```

**口诀**：
- 只读 → `for (const auto& x : v)`
- 要改 → `for (auto& x : v)`
- 小类型（int/float）→ `for (auto x : v)` 也行

---

## 5. 实战例子 1：滑动窗口均值滤波

```cpp
#include <vector>
#include <iostream>

class MovingAverage {
    std::vector<float> buffer_;
    size_t max_size_;
    size_t index_ = 0;
    bool full_ = false;

public:
    MovingAverage(size_t size) : buffer_(size, 0.0f), max_size_(size) {}

    float update(float x) {
        buffer_[index_] = x;
        index_ = (index_ + 1) % max_size_;
        if (index_ == 0) full_ = true;

        size_t count = full_ ? max_size_ : index_;
        if (count == 0) return 0;
        
        float sum = 0;
        for (size_t i = 0; i < count; i++) sum += buffer_[i];
        return sum / count;
    }
};

int main() {
    MovingAverage filter(5);
    float samples[] = {1, 2, 3, 4, 5, 6, 7, 8};
    for (float x : samples) {
        std::cout << "in=" << x << " avg=" << filter.update(x) << '\n';
    }
}
```

---

## 6. 实战例子 2：日志一组采样

```cpp
#include <vector>
#include <string>
#include <iostream>

struct Sample {
    float t;
    float y;
    float u;
};

int main() {
    std::vector<Sample> log;
    float current = 0.0f;
    for (int i = 0; i < 10; i++) {
        float u = 1.0f;             // 假装是 PID 输出
        current += u * 0.1f;
        log.push_back({i * 0.1f, current, u});
    }

    for (const auto& s : log) {
        std::cout << "t=" << s.t << " y=" << s.y << " u=" << s.u << '\n';
    }
}
```

---

## 7. STL 还有什么（简单一看）

下面这些你**控制代码里几乎用不到**，看一眼能认就行：

| 容器 | 一句话说 |
|------|----------|
| `std::map<K,V>` | 字典（键值对，有序） |
| `std::unordered_map<K,V>` | 字典（哈希表，更快） |
| `std::set<T>` | 集合（不重复，有序） |
| `std::list<T>` | 双向链表（很少用） |
| `std::deque<T>` | 双端队列 |

| 算法 | 一句话说 |
|------|----------|
| `std::sort(v.begin(), v.end())` | 排序 |
| `std::find(v.begin(), v.end(), x)` | 查找 |
| `std::accumulate` | 求和 |

> 💡 这些都在 `<algorithm>` 和 `<numeric>` 头文件里。**用到再查**。

---

## 8. 嵌入式 C++ 警告 ⚠️

**裸机 / RTOS 上**（STM32、ESP-IDF 等）：
- ❌ 一般不用 `std::vector`、`std::string`、`std::map`（用堆）
- ❌ 不用异常
- ✅ 可以用 `std::array`（栈上固定大小）
- ✅ 可以用 class、模板、`const`/`constexpr`

**有操作系统的环境**（Linux、ROS、上位机）：
- ✅ 全 STL 随便用

下一篇详细讲嵌入式风格。

---

## 动手任务

写一个函数 `mean(const std::vector<float>& v)`，返回均值。
要求：用范围 for，参数用 const 引用。

参考答案：
```cpp
float mean(const std::vector<float>& v) {
    if (v.empty()) return 0;
    float sum = 0;
    for (float x : v) sum += x;
    return sum / v.size();
}

// 用法：
std::vector<float> data = {1.0f, 2.0f, 3.0f};
std::cout << mean(data);    // 2.0
```

---

## 下一篇

→ [06 嵌入式 C++ 风格](06_嵌入式CPP风格.md)

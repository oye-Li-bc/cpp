# 02 引用与 const（控制代码最常用）

> 这两个东西看起来不起眼，但你将来 80% 的函数参数都会带它们。

---

## 1. 函数传参的三种方式

| 方式 | 写法 | 调用 | 能改外面变量吗？ | 拷贝成本 |
|------|------|------|------------------|----------|
| 传值 | `void f(int x)` | `f(a)` | ❌ 不能 | 有（拷贝一份） |
| 传指针 | `void f(int* x)` | `f(&a)` | ✅ 用 `*x` 修改 | 无 |
| 传引用 | `void f(int& x)` | `f(a)` | ✅ 直接 `x=` 修改 | 无 |

**对比代码：**
```cpp
void by_value(int x)    { x = 999; }      // 改的是副本，外面不变
void by_pointer(int* x) { *x = 999; }     // 改外面变量，但要写 *
void by_ref(int& x)     { x = 999; }      // 改外面变量，写法和普通变量一样

int main() {
    int a = 1;
    by_value(a);    std::cout << a;       // 输出 1
    by_pointer(&a); std::cout << a;       // 输出 999
    by_ref(a);      std::cout << a;       // 输出 999
}
```

---

## 2. 什么时候用哪种？

**经验法则（控制代码）：**

```cpp
// ✅ 小类型（int/float/bool），只读 → 传值
void set_kp(float kp);

// ✅ 大类型（结构体/数组），只读 → 传 const 引用（避免拷贝）
void log_state(const RobotState& state);

// ✅ 要修改外部变量 → 传引用
void update_motor(Motor& m, float dt);

// ⚠️ 要传 nullptr 或可选参数时才用指针
void set_callback(void (*cb)(int));      // 函数指针
```

**核心原则**：
- **能不拷贝就不拷贝**（嵌入式上 CPU 紧张）
- **不想被改 → 加 const**（编译期防呆）

---

## 3. const 的三种用法

### 用法一：常量

```cpp
const float PI = 3.14159f;          // 编译后 PI 不能再改
PI = 3.0f;                           // ❌ 编译报错
```

**例子（控制参数）：**
```cpp
const float WHEEL_RADIUS = 0.06f;    // 60mm 轮半径
const float MAX_SPEED = 5.0f;        // m/s
```

> 💡 比 `#define` 更安全：编译器知道类型，能查错。

### 用法二：const 引用参数（**最常用**）

```cpp
void print_state(const RobotState& s) {
    std::cout << s.angle;            // ✅ 可读
    s.angle = 0;                     // ❌ 不可写，编译报错
}
```

**为什么爱用它？**
- **没拷贝**（性能好）
- **编译器保护你**（防止意外修改）

### 用法三：const 成员函数

```cpp
class PID {
    float kp_;
public:
    float get_kp() const {           // 末尾的 const 表示"我不修改成员"
        return kp_;
    }
    void set_kp(float kp) {          // 没有 const，可以修改
        kp_ = kp;
    }
};
```

**作用**：const 对象只能调用 const 成员函数。

```cpp
const PID pid;
pid.get_kp();        // ✅ ok
pid.set_kp(1.0);     // ❌ 报错，pid 是 const
```

---

## 4. constexpr：编译期常量（嵌入式爱用）

`const` 是"运行时不变"，`constexpr` 更狠：**编译期就算出来**。

```cpp
constexpr float DT = 0.001f;             // 1ms 控制周期
constexpr int BUFFER_SIZE = 256;         // 缓冲区大小
constexpr float deg2rad(float d) {
    return d * 3.14159f / 180.0f;
}

constexpr float MAX_ANGLE = deg2rad(90.0f);   // 编译期就算好了！
```

**为什么嵌入式喜欢 `constexpr`？**
- 不占运行时 CPU
- 不占运行时内存（直接内联进代码）
- 数组大小可以用：`int buf[BUFFER_SIZE];` ✅

---

## 5. 实战例子：PID 函数签名进化

**第一版（C 风格）：**
```c
void pid_update(float *integral, float *prev_err,
                float kp, float ki, float kd,
                float target, float current, float dt,
                float *output);
```
☠️ 8 个参数，用着头大。

**第二版（C++ 引用 + 结构体）：**
```cpp
struct PIDState {
    float integral = 0.0f;
    float prev_err = 0.0f;
};
struct PIDGains {
    float kp, ki, kd;
};

void pid_update(PIDState& state,
                const PIDGains& gains,    // 只读 → const 引用
                float target, float current, float dt,
                float& output);            // 要被改 → 引用
```
☑️ 状态打包，参数只读，输出用引用——意图清晰。

**第三版（用类，下一篇讲）：**
```cpp
PIDController pid(1.0f, 0.1f, 0.05f);
float u = pid.update(target, current, dt);
```
☑️☑️ 一行搞定。

---

## 6. const 修饰指针（顺便提一下）

```cpp
const int* p1;       // p1 指向的内容不能改（"指向 const 的指针"）
int* const p2 = &x;  // p2 不能指向别处（"const 指针"）
const int* const p3; // 都不能改
```

记忆口诀：**`const` 在 `*` 左边管内容，在 `*` 右边管指针**。

控制代码里**很少用**这种写法，看到能认识就行。

---

## 7. 完整可运行例子

```cpp
#include <iostream>

constexpr float DT = 0.01f;          // 编译期常量
const float MAX_OUTPUT = 12.0f;      // 运行期常量

struct Motor {
    float speed = 0.0f;
    float target = 0.0f;
};

// 只读 → const 引用
void print_motor(const Motor& m) {
    std::cout << "speed=" << m.speed << " target=" << m.target << '\n';
}

// 要修改 → 普通引用
void simple_update(Motor& m) {
    float err = m.target - m.speed;
    m.speed += err * 0.2f;
    if (m.speed > MAX_OUTPUT) m.speed = MAX_OUTPUT;
}

int main() {
    Motor m{0.0f, 5.0f};
    for (int i = 0; i < 5; i++) {
        simple_update(m);
        print_motor(m);
    }
    return 0;
}
```

---

## 动手任务

写一个函数 `clamp(float& x, float min_val, float max_val)`，把 `x` 限制在 [min_val, max_val] 区间内。要求：
1. `x` 用引用（要修改）
2. `min_val`, `max_val` 用值传递（小类型）
3. 想想要不要加 const

参考答案：
```cpp
void clamp(float& x, float min_val, float max_val) {
    if (x < min_val) x = min_val;
    if (x > max_val) x = max_val;
}
```

> 💭 思考：`min_val`/`max_val` 是值传递，本来就是副本，加 `const` 没意义。但如果它们是大对象（比如 `const Vec3&`），就该加 const。

---

## 下一篇

→ [03 类与对象](03_类与对象.md)

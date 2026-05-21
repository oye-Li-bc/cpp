# 06 嵌入式 C++ 风格

> 你的实物板子（STM32 / ESP32 / DSP）上写 C++ 有特殊讲究。
> 这一篇告诉你**哪些能用、哪些别碰**。

---

## 1. 嵌入式开发的特殊约束

**资源紧**：
- RAM 几十 KB ~ 几百 KB（PC 上是几 GB）
- Flash 几百 KB ~ 几 MB
- CPU 主频几十 ~ 几百 MHz

**实时性强**：
- 控制周期 1ms（甚至 0.1ms），不能有不可预测的卡顿
- 不能有"垃圾回收"或长时间停顿

**没操作系统或者轻 RTOS**：
- 没有 `printf`（要重定向）
- 没有 `malloc/new`（或者 heap 很小）
- 没有标准输入输出

---

## 2. ❌ 不要用什么

### 2.1 异常（exceptions）

```cpp
try {
    do_something();
} catch (std::exception& e) {
    // 别这么写
}
```

**为什么禁用**：
- 异常机制本身要占 RAM/Flash
- 抛异常的时刻不可预测，破坏实时性
- 大多数嵌入式编译器选项里 `-fno-exceptions` 是默认

**怎么处理错误？** → 返回错误码、bool、`std::optional`，或者干脆 assert。

### 2.2 RTTI（运行时类型识别）

```cpp
if (typeid(a) == typeid(b)) { ... }
dynamic_cast<Derived*>(p);
```

**为什么禁用**：占空间、慢、控制代码用不到。
通常编译选项加 `-fno-rtti`。

### 2.3 动态内存（new/delete、std::vector、std::string）

```cpp
int* p = new int[100];        // ❌ 用堆
std::vector<float> v;          // ❌ 内部用堆
std::string name = "motor";    // ❌ 长字符串用堆
```

**为什么禁用**：
- 堆碎片化会导致系统不稳定
- malloc/free 时间不确定
- MCU 上堆很小，容易爆

**怎么替代？**
- 固定大小数组：`float buf[256];` 或 `std::array<float, 256>`
- 字符串：`char name[32];` 或自己写一个 fixed-string 类

### 2.4 复杂模板和虚函数滥用

虚函数（多态）能用，但**别滥用**——每个虚函数调用要查虚表（多一次内存访问）。
高频中断（比如 1kHz 控制循环）里，能不用虚函数就别用。

---

## 3. ✅ 推荐用什么

### 3.1 `class` + 封装

✅ 没有运行时开销（编译后和 C 一样）。
✅ 代码组织好。

```cpp
class Motor {
    float speed_;
public:
    void update(float dt);
};
```

### 3.2 `const` 和 `constexpr`

✅ 编译期计算，不占运行时资源。

```cpp
constexpr int CONTROL_HZ = 1000;
constexpr float DT = 1.0f / CONTROL_HZ;
constexpr int MAX_MOTORS = 4;
```

### 3.3 `std::array`

✅ 栈上数组，知道自己大小，不退化成指针。

```cpp
std::array<Motor, 4> motors;
for (auto& m : motors) m.update(DT);
```

### 3.4 引用、`auto`

✅ 零开销抽象。

### 3.5 模板（适度）

✅ 编译期生成代码，没运行时开销。

```cpp
template<size_t N>
class FIRFilter {
    std::array<float, N> taps_;
    // ...
};

FIRFilter<5> f1;       // 编译期生成 5 阶
FIRFilter<10> f2;      // 编译期生成 10 阶
```

---

## 4. 一份典型的嵌入式 C++ 代码

**motor_controller.h**
```cpp
#pragma once
#include <array>
#include <cstdint>

constexpr float kDt = 0.001f;        // 1kHz 控制周期
constexpr float kMaxOutput = 12.0f;
constexpr size_t kMotorCount = 4;

class PIDController {
public:
    PIDController(float kp, float ki, float kd)
        : kp_(kp), ki_(ki), kd_(kd) {}

    void reset() { integral_ = 0; prev_err_ = 0; }
    float update(float target, float current);

private:
    const float kp_, ki_, kd_;
    float integral_ = 0;
    float prev_err_ = 0;
};

class MotorController {
public:
    MotorController();
    void update();
    void set_target(size_t i, float target) { targets_[i] = target; }

private:
    std::array<PIDController, kMotorCount> pids_;
    std::array<float, kMotorCount> targets_{};
};
```

**motor_controller.cpp**
```cpp
#include "motor_controller.h"
#include "hal_motor.h"     // 假装有这么个硬件抽象层

float PIDController::update(float target, float current) {
    float err = target - current;
    integral_ += err * kDt;
    float deriv = (err - prev_err_) / kDt;
    prev_err_ = err;
    float out = kp_*err + ki_*integral_ + kd_*deriv;
    if (out >  kMaxOutput) out =  kMaxOutput;
    if (out < -kMaxOutput) out = -kMaxOutput;
    return out;
}

MotorController::MotorController()
    : pids_{
        PIDController{2.0f, 0.1f, 0.05f},
        PIDController{2.0f, 0.1f, 0.05f},
        PIDController{2.0f, 0.1f, 0.05f},
        PIDController{2.0f, 0.1f, 0.05f}
      }
{}

void MotorController::update() {
    for (size_t i = 0; i < kMotorCount; i++) {
        float current = hal_read_speed(i);
        float u = pids_[i].update(targets_[i], current);
        hal_set_pwm(i, u);
    }
}
```

**注意特点：**
- 所有大小都是 `constexpr`，编译期确定
- 用 `std::array` 而不是 `std::vector`
- 没有 `new`、没有异常
- PID 增益是 `const`，初始化列表赋值
- HAL（硬件抽象层）函数是普通 C 函数，常见做法

---

## 5. 看懂 RM 代码的一些"方言"

机器人比赛队（如东北大学 RM）代码常见风格：

### 5.1 全局单例模式

```cpp
class ChassisController {
public:
    static ChassisController& instance() {
        static ChassisController inst;       // 局部静态，线程安全
        return inst;
    }
    void update();
private:
    ChassisController() = default;
};

// 用法：
ChassisController::instance().update();
```

意思是"整个程序只有一个 ChassisController 实例"。

### 5.2 命名空间分组

```cpp
namespace robot {
namespace control {
    class PID { /* ... */ };
}
}

// 使用：
robot::control::PID pid(1.0f, 0.1f, 0.05f);
```

### 5.3 初始化列表的"花式"用法

```cpp
class Chassis {
    Motor lf_, rf_, lb_, rb_;       // 四个轮电机
public:
    Chassis()
        : lf_(MotorId::LF),
          rf_(MotorId::RF),
          lb_(MotorId::LB),
          rb_(MotorId::RB) {}
};
```

### 5.4 `enum class`（强类型枚举）

```cpp
enum class MotorId { LF, RF, LB, RB };
enum class State { Idle, Running, Error };

MotorId id = MotorId::LF;
if (id == MotorId::LF) { /* ... */ }
```

比 C 的 `enum` 安全：不会和 int 隐式转换、不会污染命名空间。

### 5.5 Functor / 仿函数（看到能认就行）

```cpp
class SquareOp {
public:
    float operator()(float x) const { return x * x; }
};

SquareOp sq;
float y = sq(3.0f);   // 输出 9
```

写起来像函数，本质是个"重载了 () 的对象"。
现代代码更多用 lambda：

```cpp
auto sq = [](float x) { return x * x; };
float y = sq(3.0f);
```

---

## 6. 一份风格清单（自检用）

写完一段控制代码，对照这个表查：

- [ ] 编译选项加了 `-fno-exceptions -fno-rtti` 吗？
- [ ] 用了 `new`/`malloc` 吗？能改成 `std::array` 吗？
- [ ] 控制周期内有没有不必要的虚函数？
- [ ] 增益、时间常数等参数是 `constexpr` 还是写死的魔术数字？
- [ ] 函数参数：大对象是不是 `const T&`？要修改的是不是 `T&`？
- [ ] 类成员变量初始化用了初始化列表吗？

---

## 7. 一些常见迷惑

### "我看到代码里有 `volatile`，是干嘛的？"

```cpp
volatile uint32_t* GPIO_REG = (volatile uint32_t*)0x40021000;
```

告诉编译器"这个内存可能被硬件改，别给我优化掉"。
寄存器、中断里改的全局变量都要加 `volatile`。

### "为什么有些类后面跟 `final`？"

```cpp
class Motor final { ... };
```

意思"这个类不允许被继承"。控制代码里用得不多，看到能认就行。

### "`override` 是什么？"

```cpp
class Base { virtual void run(); };
class Derived : public Base {
    void run() override;     // 显式说"我在覆盖父类方法"
};
```

防止你打错名字、改错参数还以为自己覆盖了父类方法。

---

## 8. 推荐看的真实代码

1. **东北大学 RM 开源代码**：你已经在参考的，主看 `chassis/`、`gimbal/`、`pid.cpp`
2. **Modern Embedded C++ Cookbook**（书）：网上找得到 PDF
3. **CMSIS-DSP**：ARM 出的嵌入式 C++ 风格 DSP 库

阅读时**别怕看不懂**，先抓主线：
- 找 main()
- 跟着调用栈看
- 遇到模板、虚函数、奇怪的 macro 先跳过

---

## 动手任务

阅读你参考的 RM 代码里的一份 `pid.h` / `pid.cpp`，回答：
1. 它的 PID 类成员变量怎么命名？
2. 用了初始化列表吗？
3. update 函数返回值是什么？

写下来对比本系列教程的写法，找出 3 处异同。

---

## 下一篇

→ [07 PID 实战](07_PID实战.md)

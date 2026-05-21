# 07 PID 实战

> 把前 6 篇学到的全部用上，写一份**能直接抄进项目**的 PID 库。

---

## 1. PID 是什么（30秒回顾）

PID = **比例 + 积分 + 微分**，让被控量逼近目标值。

```
误差 e(t) = 目标 - 当前
输出 u(t) = Kp·e(t) + Ki·∫e(t)dt + Kd·de(t)/dt
```

| 项 | 直觉 | 调大会 |
|----|------|---------|
| P（比例） | 当前差多少就纠正多少 | 响应快，但容易振荡 |
| I（积分） | 长期偏差累积 | 消除稳态误差，但易超调 |
| D（微分） | 误差变化率 | 抑制振荡，但放大噪声 |

---

## 2. 位置式 vs 增量式

### 2.1 位置式（最常见）
```
u(k) = Kp·e(k) + Ki·Σe + Kd·(e(k) - e(k-1)) / dt
```
直接算"输出多少"。**适合**：电压、角度、PWM 占空比这种"绝对量"。

### 2.2 增量式
```
Δu(k) = Kp·(e(k)-e(k-1)) + Ki·e(k) + Kd·(e(k)-2e(k-1)+e(k-2))
u(k) = u(k-1) + Δu(k)
```
算"输出变化多少"。**适合**：步进电机、阀门这种"累计执行"的对象。

> 💡 **轮腿机器人/RM 步兵**几乎都用**位置式**。本篇主讲位置式。

---

## 3. 一个最简版本（先跑起来）

```cpp
class SimplePID {
    float kp_, ki_, kd_;
    float integral_ = 0;
    float prev_err_ = 0;

public:
    SimplePID(float kp, float ki, float kd)
        : kp_(kp), ki_(ki), kd_(kd) {}

    float update(float target, float current, float dt) {
        float err = target - current;
        integral_ += err * dt;
        float deriv = (err - prev_err_) / dt;
        prev_err_ = err;
        return kp_*err + ki_*integral_ + kd_*deriv;
    }
};
```

**用法：**
```cpp
SimplePID pid(2.0f, 0.1f, 0.05f);
float u = pid.update(target, sensor_read(), 0.001f);
set_pwm(u);
```

这个版本能跑，但**有几个坑**：
1. 输出没有限幅 → 可能超过电机/驱动器极限
2. 积分项会无限累积 → **积分饱和**（windup）
3. 微分项对噪声敏感
4. dt 如果传成 0 会除零
5. 没有重启时清零的机制

---

## 4. 工业级版本（推荐抄走）

```cpp
#include <algorithm>          // for std::clamp (C++17)

class PIDController {
public:
    struct Params {
        float kp = 0.0f;
        float ki = 0.0f;
        float kd = 0.0f;
        float output_limit   = 0.0f;     // 输出绝对值上限
        float integral_limit = 0.0f;     // 积分项绝对值上限（防 windup）
        float deriv_filter   = 1.0f;     // 微分滤波系数 0~1，1=不滤
    };

    explicit PIDController(const Params& p) : params_(p) {}

    void reset() {
        integral_ = 0.0f;
        prev_error_ = 0.0f;
        prev_deriv_ = 0.0f;
        first_run_ = true;
    }

    float update(float target, float current, float dt) {
        if (dt <= 0.0f) return 0.0f;          // 防除零

        float error = target - current;

        // P 项
        float p_term = params_.kp * error;

        // I 项 + 积分限幅
        integral_ += error * dt;
        integral_ = std::clamp(integral_,
                               -params_.integral_limit,
                                params_.integral_limit);
        float i_term = params_.ki * integral_;

        // D 项 + 一阶低通滤波
        float raw_deriv = first_run_ ? 0.0f : (error - prev_error_) / dt;
        first_run_ = false;
        float deriv = params_.deriv_filter * raw_deriv
                    + (1.0f - params_.deriv_filter) * prev_deriv_;
        prev_deriv_ = deriv;
        prev_error_ = error;
        float d_term = params_.kd * deriv;

        float output = p_term + i_term + d_term;
        return std::clamp(output,
                          -params_.output_limit,
                           params_.output_limit);
    }

    void set_params(const Params& p) { params_ = p; }
    const Params& params() const { return params_; }

private:
    Params params_;
    float integral_ = 0.0f;
    float prev_error_ = 0.0f;
    float prev_deriv_ = 0.0f;
    bool  first_run_ = true;
};
```

**为什么这样写：**
| 设计 | 解决的问题 |
|------|-----------|
| `Params` 结构体 | 增益太多，打包传更清楚 |
| `output_limit` | 防止超过电机极限 |
| `integral_limit` | 防止积分饱和（windup） |
| `deriv_filter` | 微分项对噪声敏感，加一阶低通 |
| `first_run_` | 第一次调用时上次误差还没值，跳过 D |
| `dt <= 0` 检查 | 防止除零，比如刚启动定时器没起 |

---

## 5. 用法示例

```cpp
PIDController::Params p;
p.kp = 5.0f;
p.ki = 0.5f;
p.kd = 0.1f;
p.output_limit = 12.0f;       // ±12V
p.integral_limit = 8.0f;
p.deriv_filter = 0.3f;        // 滤掉 70% 高频

PIDController pid(p);

// 控制循环（1kHz）
while (running) {
    float target = get_target();
    float current = read_sensor();
    float u = pid.update(target, current, 0.001f);
    set_motor_voltage(u);
    delay_ms(1);
}
```

---

## 6. 串级 PID（Cascade PID）

轮腿机器人很常见：**外环**控位置，**内环**控速度。

```
位置目标 → [位置PID] → 速度目标 → [速度PID] → 力矩输出
```

**为什么要串级？**
- 内环响应快（高频），外环响应慢（低频）
- 内环帮外环屏蔽扰动（电机摩擦、负载变化）

**代码示例：**

```cpp
class CascadePositionVelocityPID {
public:
    CascadePositionVelocityPID(const PIDController::Params& outer,
                                const PIDController::Params& inner)
        : pos_pid_(outer), vel_pid_(inner) {}

    void reset() {
        pos_pid_.reset();
        vel_pid_.reset();
    }

    float update(float pos_target, float pos_current,
                 float vel_current, float dt) {
        // 外环：位置 PID 输出"期望速度"
        float vel_target = pos_pid_.update(pos_target, pos_current, dt);
        // 内环：速度 PID 输出"力矩 / 电压"
        float torque = vel_pid_.update(vel_target, vel_current, dt);
        return torque;
    }

private:
    PIDController pos_pid_;
    PIDController vel_pid_;
};
```

**调参建议**：先调内环（速度环），后调外环（位置环）。
口诀："里面调好，外面才稳"。

---

## 7. 调参流程（Ziegler-Nichols 简化版）

1. 把 Ki, Kd 设为 0
2. 慢慢增大 Kp，直到系统**临界震荡**（持续等幅振荡）
3. 此时的 Kp 叫 Ku，振荡周期叫 Tu
4. 经验值：
   - 只用 P：Kp = 0.5·Ku
   - PI：Kp = 0.45·Ku, Ki = 1.2·Kp/Tu
   - PID：Kp = 0.6·Ku, Ki = 2·Kp/Tu, Kd = Kp·Tu/8

实战中常用**手调**：
1. Kp 从小往大调，直到响应快但不振荡
2. 加 Ki 直到稳态误差消失（小心超调）
3. 加 Kd 直到震荡被抑制（小心放大噪声）

---

## 8. 完整可运行例子（PC 仿真）

```cpp
#include <iostream>
#include <fstream>
#include <algorithm>

class PIDController { /* 同上文，省略 */ };

// 模拟一个一阶系统：dy/dt = (-y + u) / tau
class FirstOrderPlant {
    float y_ = 0.0f;
    const float tau_ = 0.5f;
public:
    float step(float u, float dt) {
        y_ += (-y_ + u) / tau_ * dt;
        return y_;
    }
};

int main() {
    PIDController::Params p{
        2.0f, 0.5f, 0.05f,    // kp, ki, kd
        10.0f,                 // output_limit
        5.0f,                  // integral_limit
        0.5f                   // deriv_filter
    };
    PIDController pid(p);
    FirstOrderPlant plant;

    std::ofstream log("pid_log.csv");
    log << "t,target,y,u\n";

    float dt = 0.01f;
    float target = 5.0f;
    for (int i = 0; i < 500; i++) {
        if (i == 250) target = 2.0f;       // 中途改目标，看响应
        float y = plant.step(0, 0);         // 当前值（不输入也走一步）
        float u = pid.update(target, y, dt);
        plant.step(u, dt);
        log << i*dt << "," << target << "," << y << "," << u << '\n';
    }
}
```

编译运行后用 Python 画 `pid_log.csv` 就能看到响应曲线。

---

## 9. 嵌入式版本注意事项

把上面代码移植到 STM32 上时：
1. 浮点：F4/F7/H7 有 FPU，能直接 `float`；M0 没有 FPU，用 `int32_t` 定点更快
2. `std::clamp` 要 C++17，老编译器手写：
   ```cpp
   if (x > hi) x = hi; else if (x < lo) x = lo;
   ```
3. 把 PID 实例放成 **static 全局** 或类成员，**不要在中断里 new**
4. dt 一般是常数（控制频率固定），可以做成 `constexpr` 模板参数节省一次乘法

---

## 10. 课后练习

### 练习 1
给上文的 `PIDController` 增加一个"前馈"项：
```cpp
float update(float target, float current, float feedforward, float dt);
```
让输出 = 前馈 + PID 修正。

### 练习 2
写一个 `BalanceController` 类，输入：俯仰角、俯仰角速度、目标角度（一般 0），输出力矩。
内部用一个 `PIDController`。这是轮腿机器人**自平衡**的核心。

### 练习 3
把 `PIDController` 改成模板：
```cpp
template<typename T>
class PIDController { /* 用 T 替换 float */ };
```
让它能用 `float` 或 `double`（甚至定点数类型）。

---

## 学到这里你就能写控制代码了

接下来：
- 拿你已经在玩的 RM 代码，**对着每个文件读**，分辨它用了哪些 C++ 特性
- 用本篇的 `PIDController` 模板替换或对照它们的实现
- 试着自己改写一个模块（电机控制、底盘控制）

---

## 下一篇

→ [08 术语速查表](08_术语速查表.md)

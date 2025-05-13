---
title: "Calculus-2: Derivative"
date: 2025-05-12
---

# 微积分-2:导数

## 1. 导数的定义

导数是描述函数瞬时变化率的工具，也是微分学的核心。

**基本定义：**  
函数 \( f(x) \) 在点 \( x \) 处的导数定义为：

$$
f'(x) = \lim_{h \to 0} \frac{f(x + h) - f(x)}{h}
$$

前提是该极限存在。

## 2. 常用导数法则

### 和差法则

$$
(f(x) \pm g(x))' = f'(x) \pm g'(x)
$$

### 乘法法则（积的求导）

$$
(f(x) \cdot g(x))' = f'(x)g(x) + f(x)g'(x)
$$

### 商法则（商的求导）

$$
\left( \frac{f(x)}{g(x)} \right)' = \frac{f'(x)g(x) - f(x)g'(x)}{[g(x)]^2}, \quad g(x) \ne 0
$$

### 链式法则

如果 \( f(x) = h(g(x)) \)，则：

$$
f'(x) = h'(g(x)) \cdot g'(x)
$$

## 3. 导数的基本公式

### 幂函数导数

$$
(x^n)' = n x^{n-1}, \quad \text{其中 } n \text{ 为任意实数}
$$

### 指数和对数函数

$$
(e^x)' = e^x, \quad (\ln x)' = \frac{1}{x} \quad (x > 0)
$$

### 三角函数

$$
(\sin x)' = \cos x, \quad (\cos x)' = -\sin x
$$

## 4. 微分的几何意义

### 微分概念 

导数 \( f'(x) \) 表示函数在该点处切线的斜率，局部线性近似为：

$$
\Delta y \approx f'(x) \cdot \Delta x
$$

### 应用场景

- 计算瞬时变化率
- 寻找函数极值（最大值、最小值）
- 解决相关变化率问题（related rates）

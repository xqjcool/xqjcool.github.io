---
title: "Calculus-1: Limit"
date: 2025-05-10
---

# 微积分-1:极限

## 1. 极限的定义
极限是微积分中最基本的概念之一，它为定义导数和积分提供了理论基础。

**直观定义：** 当 \(x\) 接近某个值 \(a\) 时，函数 \(f(x)\) 趋近于某个定值 \(L\)，记作：

$$
\lim_{x \to a} f(x) = L
$$

## 2. 极限的四则运算

### 加法法则：

$$
\lim_{x \to a} [f(x) + g(x)] = \lim_{x \to a} f(x) + \lim_{x \to a} g(x)
$$

### 减法法则：

$$
\lim_{x \to a} [f(x) - g(x)] = \lim_{x \to a} f(x) - \lim_{x \to a} g(x)
$$

### 乘法法则：

$$
\lim_{x \to a} [f(x) \cdot g(x)] = \lim_{x \to a} f(x) \cdot \lim_{x \to a} g(x)
$$

### 除法法则：

$$
\lim_{x \to a} [\frac{f(x)} {g(x)}] = \frac{\lim_{x \to a} f(x)} {\lim_{x \to a} g(x)}, \quad \text{前提是 } \lim_{x \to a} g(x) \ne 0
$$

## 3. 极限的连续性

### 定义

一个函数 \( f(x) \) 在点 \( a \) 处连续，当且仅当：

$$
\lim_{x \to a} f(x) = f(a)
$$

### 连续函数的特性

- 在闭区间内连续的函数必定有最大值和最小值（极值定理）。
- 连续函数在区间上可作多项式近似（Weierstrass近似定理）。

## 4. 常用极限的公式

$$
\lim_{x \to 0} \frac{\sin x}{x} = 1
$$

$$
\lim_{x \to 0} \frac{1 - \cos x}{x^2} = \frac{1}{2}
$$

$$
\lim_{x \to \infty} \left(1 + \frac{1}{x}\right)^x = e
$$

$$
\lim_{x \to 0} \frac{e^x - 1}{x} = 1
$$

$$
\lim_{x \to 0} \frac{\ln(1 + x)}{x} = 1
$$

## 5. x趋于a 时的定值极限

### 直接代入法

### 因式分解法

### 渐近线 + 扭动法

### 共轭表达式法

## 6. x趋于无穷时的极限

### 首项主导法

## 7 无穷小

### 特性

- 无穷小加减后还是无穷小
- 无穷小相乘还是无穷小
- 无穷小与有界函数相乘还是无穷小


---
layout:     post
mathjax:    true
title:      "RSA 算法原理全面解析"
subtitle:   ""
date:       2018-11-18 22:43:00 +0800
author:     "ssj"
header-img: "img/post/rsa-algrithm.png"
categories: 密码学
catalog: true
comments: true
tags:
    - RSA
    - 算法
    - HTTPS
---

本文最早发于 [简书](https://www.jianshu.com/p/6aa7b59be872)。

> 网上写 RSA 算法原理的文章不少，但是基本上要么忽略了数学原理的说明，要么缺少实际的可运行的例子，为此特写了此文，将 RSA 需要用到的数学概念和定理都总结了一番，并基于算法原理使用 python 实现了 RSA 密钥生成和加解密的 demo，希望对大家深入了解 RSA 算法有所帮助。本文第 1 节为 RSA 相关的数论基础，第 2 节是 RSA 算法原理描述和证明，第 3 节通过 openssl 生成 RSA 密钥来分析实际应用中的密钥格式。可以直接从第 2 节开始，遇到相关定理可以转到第 1 节处查阅即可。本文所有代码都在 [shishujuan:rsa-algrithm](https://github.com/shishujuan/rsa-algrithm)，如有错误，也恳请指正。

# 1 数论基础
本节内容中可能用到的符号说明如下：

- $\Bbb Z$: 指代整数集合，包括正负整数和0。
- $\Bbb Z_n$: 指代 0 到 n-1 之间的整数集合。
- $\|a\|$：数 a 的绝对值。
- $⌊a/n⌋$: 指代小于等于 a/n 的最大整数，比如 ⌊5/3⌋ = 1，⌊8/3⌋ = 2 等。
- 质数：质数在有些地方又翻译为素数，本文统一用质数表述。
- $p^{-1}_q$：p 对模数 q 的模逆元素，即 $\small p^{-1}_q p \equiv 1(mod ~q)$
- 乘法符号：文中可能在证明和示例中用了不同的乘法符号如 $\times$，$\cdot$，意义一致。

## 1.1 质数和合数
**质数和合数：** 质数是指除了平凡约数1和自身之外，没有其他约数的大于1的正整数。大于1的正整数中不是素数的则为合数。如 7、11 是质数，而 4、9 是合数。在 RSA 算法中主要用到了质数相关性质，质数可能是上帝留给人类的一把钥匙，许多数学定理和猜想都跟质数有关。

## 1.2 除法定理
**[定理1] 除法定理：** 对任意整数 a 和 任意正整数 n，存在唯一的整数 q 和 r，满足 $0<=r<n，a = qn + r$。其中，$q = ⌊a/n⌋$ 称为除法的商，而 $r = a ~mod ~ n$ 称为除法的余数。

> [证明] 除法定理
> - 存在性证明
> 设集合 $S=\\{a - xn，x \in \Bbb Z \\}$，则可以知道集合 S 中一定存在非负元素，设这个非负整数集合为 T，根据自然数的良序法则，非空自然数集合中必然存在最小元素。设 $r = a-qn$ 为集合 T 中的最小元素，因为集合 T 是非负整数集合，因此 $r >= 0$。而 r 必然满足 $r < n$，因为 $ r >= n \implies r-n>=0$，则可以推出元素 $ r - n \in T$。但是  $r - n < r$，这与我们定义的 r 是集合 T 中最小的元素矛盾，因此 $r < n$。这也就证明了 q 和 r 的存在性，即一定存在 q 和 r，且 $ 0 <= r < n$。
> - 唯一性证明
> 假设有两组不同的整数$q_1, r_1$ 和 $q_2, r_2$满足条件，则有 $a = q_1n + r_1，a = q_2n + r_2( 0 <= r_1, r_2 < n)$
> 两个式子相减可得 
> $r_2 - r_1 = n(q_1 - q_2)$
> 左边绝对值 $\|r_2 - r_1\| < n$，但是右边绝对值 $\|n(q_1-q_2)\|$ 显然是 n 的整数倍，因此只能 $q_1 = q_2$，继而可得 $r_1 = r_2$，由此唯一性得证。

**整除：** 在除法定理中，当余数 $r = 0$ 时，表示 a 能被 n 整除，或者说 a 是 n 的倍数，用符号 $n\ \|\ a$ 表示。

## 1.3 约数、公约数和最大公约数
**约数和倍数**： 对于整数 d 和 a，如果 $d\ \|\ a$，且 $d >= 0$，则我们说 d 是 a 的约数，a 是 d 的倍数。

**公约数：** 对于整数 d，a，b，如果 d 是 a 的约数且 d 也是 b 的约数，则 d 是 a 和 b 的公约数。如 30 的约数有 1，2，3，5，6，10，15，30，而 24 的约数有 1，2，3，4，6，8，12，24，则 30 和 24 的公约数有 1，2，3，6。其中 1 是任意两个整数的公约数。

公约数的性质：

- 若 $d \| a$，且 $d \| b => d \| (a+b)$ 且 $d \| (a-b)$。更一般的，对任意整数 x 和 y，有：$d \| a$，且 $d \| b => d \| (ax + by)。$ $(1.1)$
- 若 $a \| b$，则要么 $b = 0$，要么 $\|a\| <= \|b\|$，即 $a \| b$ 且 $b \| a$ 可以推出 $\|a\| = \|b\|。$ $(1.2)$

**最大公约数：** 两个整数最大的公约数称为最大公约数，用 $gcd(a,b)$ 来表示，如 30 和 24 的最大公约数是 6。$gcd(a, b)$ 有一些显而易见的性质：

- $ gcd(a, b) = gcd(b, a) \quad(1.3)$
- $ gcd(a, 0) = \|a\| \quad(1.4) $
- $ gcd(ka, a) = \|a\| \quad(1.5) $

**[定理2] 最大公约数定理：** 如果 a 和 b 是不为0的整数，则 $gcd(a, b)$ 是 a 和 b 的线性组合集合 ${\lbrace ax + by: x, y \in \mathbb{Z} \rbrace}$ 中的最小正元素。

>[证明] $gcd(a, b)$ 是 a 和 b 线性组合集合最小正元素。
> 
> 设 s 是 a 和 b 的线性组合集合中最小正元素，且对某个 $x，y$，有 $s = ax + by$，设 $q = ⌊a/s⌋$，则 $a\ mod\ s = a - qs = a - q(ax + by) = a(1-qx) + b(-qy)$，因此 $a\ mod\ s$ 也是 a 和 b 的一种线性组合。由于 $0 <= a\ mod\ s < s$，所以只能 $a\ mod\ s = 0$，不然就与我们假设矛盾了。由 $a\ mod\ s = 0$，于是 $s \| a$，类似可以得到 $s \| b$，于是 s 是 a 和 b 的公约数，于是根据定义，$gcd(a, b) >= s$。
> 
> 另一方面，因为 $gcd(a, b)$ 可以同时被 a 和 b 整除，而 s 是 a 和 b 的一个线性组合，由性质 1.1 可得 $gcd(a, b) \| s$，另外 $s > 0$，由性质 1.2 可知 $gcd(a, b) <= s$。结合前面的 $gcd(a, b) >= s$，可证得 $s = gcd(a,b)$。

由定理2可以得到一个推论：

**[推论1]** 对任意整数 a 和 b，如果 $d\ \|\ a$ 且 $d\ \|\ b$，则 $d\ \|\ gcd(a, b)$。

> 推论1的证明比较简单，由定理2可知 $gcd(a,b)$ 是 a 和 b 的一个线性组合，故由性质 1.1 可得证 $d\ \|\ gcd(a,b)$ 。

## 1.4 互质数
**互质数：** 如果两个整数 a 和 b 只有公因数 1，即 $gcd(a, b) = 1$，则我们就称这两个数是互质数(coprime）。比如 4 和 9 是互质数，但是 15 和 25 不是互质数。

互质数的性质：

- 1与任何正整数都是互质数。$(1.6)$
- 任何两个质数都是互质数。 $(1.7)$
- 质数与小于它的每一个数都是互质数。$(1.8)$

## 1.5 欧几里得算法
欧几里得算法分为朴素欧几里得算法和扩展欧几里得算法，朴素法用于求两个数的最大公约数，而扩展的欧几里得算法则有更多广泛应用，如后面要提到的求一个数对特定模数的模逆元素等。

### 1.5.1 朴素欧几里得算法
求两个非负整数的最大公约数最有名的是 辗转相除法，最早出现在伟大的数学家欧几里得在他的经典巨作《几何原本》中。辗转相除法算法求两个非负整数的最大公约数描述如下：

- $gcd(a, 0) = a$
- $gcd(a, b) = gcd(b, a\ mod\ b)(a, b \in \mathbb{Z}，a,b >= 0)$

例如，$gcd(252, 105) = gcd(105, 42) = gcd(42, 21) = gcd(21, 0) = 21$，在求解过程中，较大的数缩小，持续进行同样的计算可以不断缩小这两个数直至其中一个变成零。

> [证明] 欧几里得辗转相除法求GCD
> 
> 由条件知 $a >= b >= 0$，令 $r = a % b，a = bt + r$，则需要证明 $gcd(a, b) = gcd(b, r)$。
> 
> - 首先，令 $d = gcd(a, b)$，则根据 gcd 定义，$d \| a$ 且 $d \| b$。即 $d \| (bt + r)$ 且 $d \| b$，由此可得 $d \| r$，则 $d \| gcd(b, r)$。即 $gcd(a, b) \| gcd(b, r)$。
> 
> - 其次，令 $c = gcd(b, r)$，则根据 gcd 定义，$c \| b，c \| r$。因为 $a = bt + r$，因此，$c \| a$。于是 $c \| a，c \| b$，可得 $c \| gcd(a,b)$，即 $gcd(b,r) \| gcd(a,b)$。
>
> - 由此可得 $gcd(a, b) \| gcd(b, r)$ 且 $gcd(b, r) \| gcd(a, b)$，由性质1.2 可得 $gcd(a, b) = gcd(b, r)$。
 
欧几里得算法的python实现如下：

```
gcd = lambda a, b: (gcd(b, a % b) if a % b else b)
```

### 1.5.2 扩展欧几里得算法
扩展欧几里得算法在 RSA 算法中求模反元素有很重要的应用，定义如下：

**定义：** 对于不全为 0 的非负整数 $a，b$，则必然存在整数对 $x，y$，使得

$$ax + by = gcd(a, b)$$

例如，a 为 3，b 为 8，则 $gcd(a, b) = 1$。那么，必然存在整数对 $x, y$，满足 $3x + 8y = 1$。简单计算可以得到 $x = 3, y = -1$ 满足要求。

>[证明] 扩展欧几里得算法
> 
> - 设 $a > b$，则当 $b = 0$ 时，$gcd(a, 0) = a$，则 $x = 1，y = 0$ 就是一组解。
> 
> - $a，b$ 都不为 0 时，令 $r = a\ mod\ b = a - ⌊a/b⌋b$，由定理2，设 $ax_1 + by_1 = gcd(a, b)$ 。此外，由定理2，设 $bx_2 + ry_2 = gcd(b, r)$，根据 r 的定义改写此等式，可以得到 $bx_2 + (a - ⌊a/b⌋b)y_2 = gcd(b, r)$，即 $gcd(b, r) = ay_2 + b(x_2-⌊a/b⌋y_2)$， 根据欧几里得算法，$gcd(a, b) = gcd(b, r)$，则令 $x_1 = y_2，y_1 = x_2-⌊a/b⌋y_2$ 即可。由此可以得到求解 $x，y$ 的递归解法。

扩展欧几里得算法的python实现如下：

```
# 扩展欧几里得算法实现，参数为 a 和 b，返回最大公约数 d 和 线性组合中的参数 x，y。
def ext_gcd(a, b):
    if b == 0:
        return a, 1, 0

    d2, x2, y2 = ext_gcd(b, a % b)
    d, x, y = d2, y2, x2-a/b*y2
    return d, x, y
```

## 1.6 同余和模运算
**同余：** 对于正整数 n 和 整数 a，b，如果满足 $\small n \| (a-b)$ ，即 a-b 是 n 的倍数，则我们称 a 和 b 对模 n 同余，记号如下：$$ a \equiv b (mod ~ n)$$ 例如，因为 $\small 12 \| (13-1)$，于是有 $\small 13 \equiv 1 (mod\ 12)$。
对于正整数 n，整数 $\small a_1, a_2, b_1, b_2$，如果 
$$a_1 \equiv a_2~(mod ~ n)，b_1 \equiv b_2~(mod ~n)$$
则我们可以得到如下性质：

- $a_1 + b_1 \equiv a_2 + b_2 ~ (mod ~n) \quad（1.9）$
- $a_1 \cdot b_1 \equiv a_2 \cdot b_2 ~ (mod ~n) \quad (1.10) $

譬如，因为 $\small 9 \equiv 4 ~ (mod ~ 5)，8 \equiv 3 ~ (mod ~ 5)$，则可以推出 $\small 17 \equiv 7 ~ (mod ~ 5)，72 \equiv 12 ~(mod ~5)$。
> [证明] 同余加法和乘法性质
> 因为 $a_1 \equiv a_2 ~(mod ~n)，b_1 \equiv b_2 ~ (mod ~n)$，则说明存在整数 c 和 d，满足 $a_2 = a_1 + cn，b_2 = b_1 + dn$，因此：
$$a_2 + b_2 = a_1 + b_1 + (c+d) n$$ 
加法性质得证，同理可得： 
$$a_2b_2 = (a_1 + cn)(b_1 + dn)=a_1b_1 + (a_1d+b_1c+cdn)n$$
乘法性质得证。

另外，若 p 和 q 互质，且 
$$a ≡ b (mod ~ p)，a ≡ b (mod ~ q)$$
则可推出：
$$ a ≡ b (mod ~pq)  \quad (1.11) $$ 
> [证明] 因为  $a ≡ b (mod ~ p) \implies s\|a-b，a ≡ b (mod ~ q) \implies q\|a-b$。而 p 和 q 互质，于是可得 $pq\|a-b$，因此可得 $a ≡ b (mod ~ pq)$。

此外，模的四则运算还有如下一些性质，证明也比较简单，略去。

- $(a+b) ~mod ~ n = (a ~mod~ n + b ~ mod ~n) ~mod~ n $
- $(a-b) ~mod ~ n = (a ~mod~ n - b ~ mod ~n) ~mod~ n $
- $(a \cdot b) ~mod ~ n = (a ~mod~ n \cdot b ~ mod ~n) ~mod~ n $
- $a^b ~mod~ n = ((a ~mod ~ n)^b) ~mod~ n$


**模逆元素：** 对整数 a 和正整数 n，a 对模数 n 的模逆元素是指满足以下条件的整数 b。
$$ab \equiv 1(mod\ n)$$ 
a 对 模数 n 的 模逆元素不一定存在，a 对 模数 n 的模逆元素存在的充分必要条件是 a 和 n 互质，这个在后面我们会有证明。若模逆元素存在，也不是唯一的。例如 a=3，n=4，则 a 对模数 n 的模逆元素为 7 + 4k，即 7，11，15，...都是整数 3 对模数 4 的模逆元素。如果 a 和 n 不互质，如 a = 2，n = 4，则不存在模逆元素。

**[推论2] 模逆元素存在的充分必要条件是整数 a 和 模数 n 互质。**

> [证明] 模逆元素存在性证明
> 
> - 若 a 和 n 互质，则 $gcd(a, n) = 1$，由扩展欧几里得算法知道，必然存在 x, y 使得 $ax + ny = gcd(a, n) = 1$，则模除 n 可得 $ax ≡ 1(mod\ n)$，显然 x 就是 a 对 模数 n 的模逆元素。事实上，$x + kn(k \in \mathbb{Z})$ 都是 a 对模 n 的模逆元素，取其中最小的就是 x。
> - 若 a 和 n 不互质，设 $gcd(a, n) = g$，则 $ax + ny = g (1 < g <= n)$，则在模除 n 时，$ax ≡ g$ 不会同余为 1，故此时不存在模逆元素。

## 1.7 唯一质数分解定理
**[定理3] 唯一质数分解定理：** 任何一个大于1的正整数 n 都可以 **唯一分解** 为一组质数的乘积，其中 $e_1，e_2... e_k$ 都是自然数(包括0)。比如 6000 可以唯一分解为 $\small 6000 = 2^4 \times 3 \times 5^3$ 。

$$n = 2^{e_1} \cdot 3^{e_2} \cdot 5^{e_3}  ... = \prod_{k=1}^r p_k^{e_k} $$

> **[证明] 唯一质数分解定理**
> 
> **存在性证明**
> - 使用反证法来证明，假设存在大于1的自然数不能写成质数的乘积，把最小的那个称为 n。自然数可以根据其可除性（是否能表示成两个不是自身的自然数的乘积）分成3类：质数、合数和 1。首先，按照定义，n大于1。其次，n 不是质数，因为质数 p 可以写成质数乘积：$p = p^1$，这与假设不相符合，因此n只能是合数。
> - 但每个合数都可以分解成两个严格小于自身而大于 1 的自然数的积。设 $n = a \cdot b(1 < a, b < n)$，由假设可知，a 和 b 都可以分解为质数的乘积，因此 n 也可以分解为质数的乘积，所以这与假设矛盾。由此证明所有大于 1 的自然数都能分解为质数的乘积。
>
> **唯一性证明**
> - 当 n=1 的时候，确实只有一种分解。
> - 假设对于自然数 n>1，存在两种因式分解: $n=p_1...p_m
 = q_1...q_k(p_1<=...<=p_m，q_1<=...<=q_k)$，其中 p 和 q 都是质数，我们要证明 $p_1=q_1, ..., p_m=q_k$。如果不相等，我们可以设 $p_1 < q_1$，从而 $p_1$ 小于所有的 $q$。由于 $p_1$ 和 $q_1$ 是质数，所以它们的最大公约数为1，由欧几里德算法可知存在整数 a 和 b 使得 $ap_1 + bq_1 = 1$。因此
$a p_1 q_2...q_k + b q_1 q_2...q_k = q_2 ... q_k$ (等式1)。由于 $q_1...q_k = n$，因此等式1左边是 $p_1$ 的整数倍，从而等式1右边的 $q_2...q_k$ 也必须是 $p_1$ 的整数倍，因此必然有 $p_1 = q_i(i > 1)$，而这与前面 $p_1$ 小于所有的 $q$ 矛盾。
> - 由对称性，对 $p_1 > q_1$ 这种情况可以得到类似结论，故可以证明 $p_1 = q_1$，同理可得 $p_2 = q_2, ..., p_m=q_k$，由此完成唯一性的证明。

由质数唯一分解定理可以得到一个推论：**质数有无穷多个**。
> **[证明] 质数有无穷多个。**
> - 用反证法。假设质数有限，为 $p_1, p_2, ..., p_k$，最大质数为 $p_k$，则令 $N$ 为所有质数之积+1，即 $N = p_1p_2...p_k + 1$，显然 $N > p_k$，且 $N$ 不能被已有的任何一个质数整除。
> 
> - 由质数唯一分解定理知道，$N$ 可以分解为质数的乘积，但是它无法分解为已有质数的乘积，那么要么 $N$ 自己是质数；要么 $N$ 是合数，但是存在比 $p_k$ 更大的质数分解因子。不管哪种情况，都与假设的最大质数为 $p_k$ 矛盾，故假设不成立，质数有无穷多个。


## 1.8 中国剩余定理
**[定理4] 中国剩余定理(Chinese remainder theorem，CRT)**，最早见于《孙子算经》(中国南北朝数学著作，公元420-589年)，叫物不知数问题，也叫韩信点兵问题。

> 有物不知其数，三三数之剩二，五五数之剩三，七七数之剩二。问物几何？

翻译过来就是已知一个一元线性同余方程组求 x 的解：
$$(S):\begin{cases}
 & x \equiv 3 \ (mod\ 2) \\
 & x \equiv 5 \ (mod\ 3) \\
 & x \equiv 7 \ (mod\ 2) 
 \end{cases}$$

宋朝著名数学家秦九韶在他的著作中给出了物不知数问题的解法，明朝的数学家程大位甚至编了一个《孙子歌诀》：

> 三人同行七十希，五树梅花廿一支，七子团圆正半月，除百零五便得知。

意思就是：将除以 3 的余数 2 乘以 70，将除以 5 的余数 3 乘以 21，将除以 7 的余数 2 乘以 15，最终将这三个数相加得到 $\small 2 \cdot 70 + 3 \cdot 21 + 2 \cdot 15 = 233$。再将 233 除以 3，5，7 的最小公倍数 105 得到的余数 $\small 233 % 105 = 23$，即为符合要求的最小正整数，实际上，$\small 233 + 105k(k \in \mathbb{Z})$ 都符合要求。

**物不知数问题解法本质**

- 从 5 和 7 的公倍数找到一个除以 3 余 1 的 $n_1$ (如70)，从 3 和 7 的公倍数中找一个除以 5 余 1 的 $n_2$(如21)，从 3 和 5 的公倍数找一个除以 7 余 1 的 $n_3$ (如15)，则 $\small 2n_1+3n_2+2n_3=140+63+30=233$，模除 105，得最小正整数解 23。
- 因为 $n_1$ 除以3 余1， $n_2$ 和 $n_3$ 都是 3 的倍数，则由性质 1.9 知，$\small n_1+n_2+n_3 = 106$ 满足除以 3 余 1，同理也可得它满足除以 5 余 1，除以 7 余 1的条件。而前面找的是余 1 的数，使用性质 1.9 和 1.10 可知，乘以余数后 $\small 2n_1+3n_2+2n_3$ 正好满足要求，即除以 3 余 2，除以 5 余 3，除以 7 余 2。为什么要先找余 1 的数，然后再乘以余数呢？这个可以在后面的通项公式求解知道其中缘由。

**求解通项公式**

中国剩余定理相当于给出了以下的一元线性同余方程组的有解的判定条件，并用构造法给出了解的具体形式。

$$(S):\begin{cases}
 & x \equiv a_1 \ (mod\ m_1) \\
 & x \equiv a_2 \ (mod\ m_2) \\
 & \vdots \\
 & x \equiv a_n \ (mod\ m_n) 
 \end{cases}$$

**模数 $m_1，m_2，...，m_n$ 两两互质**，则对任意的整数：$a_1，a_2，...，a_n$，方程组 $(S)$ 有解，且解可以由如下构造方法得到：

- 1）设 M为所有模数的乘积
 
 $$M = m_1 \cdot m_2 \cdot ... m_n = \prod _{i=1}^n m_i$$
 
   并设 $M_i$ 是除 $m_i$ 以外的其他 $n−1$ 个模数的乘积。
 
 $$M_i = M / m_i, ~ \forall i\in \\{ 1, 2, ... n \\}$$
 
- 2）设 $t_i$ 为 $M_i$ 对模数 $m_i$ 的模逆元素，即 

 $$t_i M_i \equiv 1 ( mod ~ m_i), ~ \forall i \in \\{ 1,2,...n\\}$$
 
- 3）则方程组 (S）的通解形式为：
 $$x = a_1t_1M_1 + a_2t_2M_2 + ... a_nt_nM_n + kM = kM + \sum_{i=1}^na_it_iM_i, ~ k \in \mathbb{Z}$$
 
- 4）以物不知数问题为例，则由 $\small a_1 = 2, a_2 = 3, a_3 = 2，m_1 = 3, m_2 = 5, m_3 = 7$，$\small M = 3 \cdot 5 \cdot 7 = 105，M_1 = M/m_1 = 35 $，同理得 $\small M_2 = 21，M_3 = 15$。求 $ \small M_1(35)$ 模除 $\small m_1(3)$ 的最小的模逆元素，可得 $\small t_1 = 2$，同理得 $\small t_2 = 1，t_3 = 1$。故而解为：

$\small x = a_1t_1M_1 + a_2t_2M_2 + a_3t_3M_3 + kM$
$\small = 2 \times 2 \times 35 + 3 \times 1 \times 21 + 2 \times 1 \times 15 + k \times 105$
$\small = 233 + k \times 105, \quad k \in \mathbb{Z}$

**中国剩余定理通项公式证明**

> [证明] 中国剩余定理
> - 1） 先证明简单的情况，假设只有两个方程，$m_1, m_2$互质，求解 x。
> $$(S):\begin{cases}
 & x \equiv a_1 \ (mod\ m_1) \\
 & x \equiv a_2 \ (mod\ m_2) \\
 \end{cases}$$
> 这里我们构造一个解：
> $$x = a_1t_1m_2 + a_2t_2m_1 \quad (t_1m_2 \equiv 1 ~(mod ~ m_1)，t_2m_1 \equiv 1 ~ (mod ~m_2))$$  
> 即 $t_1$ 是  对模数 $m_2$ 的模逆元素，$t_2$ 是 $m_1$ 对模数 $m_2$ 的模逆元素 (因为 m_1 和 m_2 互质，则对应的模逆元素一定存在)，则可以证明 x 是满足条件的解，证明如下： 
$x \equiv  (a_1t_1m_2 + a_2t_2m_1) \equiv a_1t_1m_2 \equiv a_1 (mod ~ m_1)$ 
$x \equiv  (a_1t_1m_2 + a_2t_2m_1) \equiv a_2t_2m_1 \equiv a_2 (mod ~ m_2)$
>
>   当然，对于两个方程的情况，我们也可以直接根据模运算的定义来求解，**这个方式也会在后面 RSA 算法优化中用到**。因为 $x \equiv a_2 (mod ~ m_2)$，于是可以令：
> $$x = km_2 + a_2$$ 
> 代入到第一个方程有：
> $$km_2 + a_2 \equiv a_1 ~ (mod ~ m_1)$$ 
> 根据性质 1.9，可得:
> $$a_1 - a_2 \equiv km_2 (mod ~ m_1)$$ 
> 因为 $m_1$，$m_2$ 互质，所以 $m_2$ 对于 $m_1$ 必然存在模逆元素 $t_2$，满足 $t_2m_2 \equiv 1~ (mod ~m_1)$，根据性质 1.10 可得：
> $$t_2(a_1 - a_2) \equiv kt_2m_2 \equiv k ~(mod ~ m_1)$$ 
> 由此可以得到 $ k = t_2(a_1-a_2) ~mod ~m_1$，从而得到 x 在 $[0, m_1m_2)$ 范围内的唯一解是:
> $$ x = a_2 + km_2 = a_2 + (t_2(a_1-a_2) ~ mod ~ m_1) \cdot m_2 \quad (1.12)$$
>
> - 2）推广到 n 个方程组，对 $\forall i \in \\{1, 2, ..., n \\}$，则对 $\forall j \in \\{1, 2, ..., n \\}，j \neq i$，因为 $m_i$ 与 $m_j$ 互质，则可得 $gcd(m_i, m_j) = 1$，且 $m_i$ 和 $M_i$ 互质，则 $gcd(m_i, M_i) = 1$。因为 $gcd(m_i, M_i) = 1$，由推论2 知，必然存在整数 $t_i$，满足：$t_iM_i \equiv 1(mod\ m_i)$，即 $t_i$ 是 $M_i$ 模 $m_i$ 的模逆元素。考察乘积 $a_it_iM_i$ 可知：
> $$a_it_iM_i \equiv a_i (mod \ m_i)$$ 
> 而$\forall j \in \\{1, 2, ..., n\\}，j \neq i，a_it_iM_i \equiv 0 (mod \ m_j)$ ，故而 $x = a_1t_1M_1 + a_2t_2M_2 + ... a_nt_nM_n$ 满足：
> $$x = a_it_iM_i + \sum_{j \neq i} a_jt_jM_j \equiv a_i + \sum_{j \neq i}0 \equiv a_i(mod \ m_i)$$ 
> 所以可以得出 x 就是方程组 $(S)$ 的一个解。
> 
> - 3）另外，假设 $x_1$ 和 $x_2$ 都是方程组的解，那么：
>  	$\forall i \in \\{1, 2, ..., n\\}, \quad x_1-x_2 \equiv 0 \ (mod\ m_i)$，而 $m_1, m_2, ..., m_n$ 互质，从而可得 $M\ \|\ (x_1 -x_2) $。即 两个解必然相差 $M$ 的整数倍，故而方程组通解为：
> $$\\{x =  \sum_{i=1}^na_it_iM_i+ kM, \ k \in \mathbb{Z}\\}$$

## 1.9 欧拉函数、欧拉定理和费马定理
### 1.9.1 欧拉函数
**欧拉函数定义**：对正整数 n，欧拉函数 $φ(n)$ 是指小于或等于 n 的正整数中与 n 互质的数的数目。比如 $φ(1)=1，φ(8) = 4$（因为 1，3，5，7 都与 8 互质）。由唯一质数分解定理可知，正整数 n 可以分解为质数乘积，据此可以得到欧拉函数的计算公式如下。

唯一质数分解：$n = \prod_{k=1}^r p_k^{e_k}$
欧拉函数计算公式：$φ(n) = n \prod_{k=1}^r (1 - \frac {1}{p_k})$ 

如 $72 = 2^3 \times 3^2$，故 $φ(72) = 72 \times (1-1/2)  \times (1-1/3)=24$。

> **欧拉函数计算公式推导：**
> 
> - 1）若 n 为 1，显然 $φ(1) = 1$。
> - 2）若 n 为质数，有性质 1.8 可知，每个小于 n 的数都与 n 是互质数。则 $φ(p) = p - 1$。
> - 3）若 n 为质数的幂，即 $n = p^k$ (p为质数，k为大于等于1的整数），则因为只有当一个数不包含质数 p，才可能与 n 互质。而包含质数 p 的数一共有 $p^{k-1}$ 个，故互质数目为 $p^k - p^{k-1}$。
> $$φ(n) = p^k - p^{k-1} = p^k(1 - \frac{1}{p})$$
> - 4）若 n 为两个互质数的乘积，即 $n = p \cdot p $ (p, q 为互质数)。则 $φ(n) = φ(p) \cdot φ(q)$，例如 $φ(56) = φ(8) \cdot φ(7) = 4 \cdot 6 = 24$。在证明之前，可以看个特例，即 p 和 q 都是质数，则 p 和 q 肯定是互质数，这种情况下与 n 互质的数不能包含质因子 p 和 q，p 的倍数有 q 个，q 的 倍数有 p 个，这里还多算了一个数 n，它即是 p 的倍数也是 q 的倍数，因此 1 到 n 中 p 和 q 的倍数的数一共有 $p+q-1$ 个，则可以推导如下：
>
>   $φ(n) = n - (p + q-1)$
> $ = pq - (p + q - 1)  = (p-1)(q-1)$
> $ = p(1 - \frac{1}{p})q(1-\frac{1}{q}) $
> $ = φ(p)φ(q)$
> 
>   若 p 和 q 互质，但是 p 和 q 不一定是质数。 则 $gcd(p, q) = 1$，设小于 p 的数中与 p 互质的正整数有 $φ(p)$ 个，则这些数的集合为 $R = \\{r_1, r_2, ..., r_{φ_p}\\}$。而小于 q 且与 q 互质的正整数有 $φ(s)$个，设这些数的集合为 $S = \\{s_1, s_2, ..., s_{φ_q}\\}$。从 集合 R 和 S中分别选取一个数 r 和 s，构造方程组 $x \equiv r(mod\ p)$ 以及 $x \equiv s(mod \ q)$，由中国剩余定理可知，则当 $x < pq$ 时方程组存在且仅存在一个正整数解，即 $x ~ mod ~ pq$ 有唯一解。且该方程组在 p 和 q相同的情况下，如果 r 和 s 不同，则解 x 也一定不同。下面证明解 x 满足 $gcd(x, pq) = 1$。
> 
>   因为 $x\ \%\ p = r$，由欧几里得算法可得 $gcd(x, p) = gcd(p, r)$，当 $gcd(p, r) = 1$ 时，则可得 $gcd(x, p) = 1$，同理，$gcd(x, q) = 1$。由 $gcd(x, p) =1$ 和 $gcd(x, q) = 1$，且 $p、q$ 互质，可得 $gcd(x, pq) = 1$，即 x 是满足欧拉函数的一个数。 而  $<r, s>$ 这样的数对一共有 $φ(p) φ(q)$ 个，而 $x$ 一共有 $φ(pq)$个，故而得证 $φ(pq) = φ(p)φ(q)$。
>
>   看一个实际的例子，设 $x= 15 = p \cdot q = 3 \cdot 5$，则对于 $x ~mod~ 3 = a, x ~mod~ 5 = b$，(a, b) 和 x 存在如下一一对应关系。即若 p 和 q 为互质数，则对于任意的数对 $<a, b> \in \Bbb Z_p \times \Bbb Z_q$，存在唯一的 $x \in \Bbb Z_{n}$(其中 $\Bbb Z_n$ 表示整数集合 $\\{0, 1, ..., n-1 \\}$)。

| (a, b) | x | (a, b) | x | (a, b) | x |
| ------ | ------ | ------ |------ |----- |----- |
| (0, 0) | 0 | (1, 0) | 10 | (2, 0) | 5 |
| (0, 1) | 6 | (1, 1) | 1 | (2, 1) | 11 |
| (0, 2) |12 | (1, 2) | 7 | (2, 2) | 2|
| (0, 3) | 3 | (1, 3) | 13 | (2, 3) | 8 |
| (0, 4) | 9 | (1, 4) | 4  | (2, 4) | 14 |

> - 5) 综上，可以得到通项公式：
>
>   设 $n = p_1^{e_1} \cdot p_2^{e_2} \cdot ... p_r^{e_r}$
>
>   $ φ(n) = φ(p_1^{e_1} \cdot p_2^{e_2} \cdot ... p_r^{e_r})$
>   $ = φ(p_1^{e_1}) \cdot φ(p_2^{e_2}) \cdot ... φ(p_r^{e_r}) $ (由4可得)
>   $ = p_1^{e_1} \cdot p_2^{e_2} ... p_r^{e_r}\cdot(1 - \frac{1} {p_1})\cdot(1-\frac{1}{p_2})...(1-\frac{1}{p_r}) $ (由3可得)
> $ = n\cdot (1 - \frac{1}{p_1})\cdot(1-\frac{1}{p_2})...(1-\frac{1}{p_r}) 
$ (由 n 分解式可得)

计算欧拉函数可以使用 python 的这个模块： [eulerlib](https://pythonhosted.org/eulerlib/_modules/eulerlib/numtheory.html#Divisors.phi)，一个示例代码如下：

```
import eulerlib
e = eulerlib.numtheory.Divisors(10000)
e.phi(21)
```

### 1.9.2 欧拉定理
欧拉定理基于欧拉函数而来，它是 RSA 算法的核心。内容如下：
**欧拉定理：**  如果两个正整数  a 和 n 互质，则 n 的欧拉函数 $φ(n)$ 满足下面的条件：

$$ a ^ {φ(n)} \equiv 1 (mod\ n)$$

基于前面的知识，我们来证明下欧拉定理。
> [证明] 欧拉定理
> - 根据欧拉函数定义，小于 $n$ 且与 $n$ 互质的数有 $φ(n)$ 个，设这些数的集合为 $U = \\{ u_1, u_2, ..., u_{φ(n)} \\}$。显然对于 U 中的任意数 $u_i$，都有 $u_i ~ mod ~ n = u_i$。若一个正整数 a 与 n 互质，且 a 比 n 大，同样满足 $a \ mod \ n$ 与 n 互质，即 $(a ~ mod ~ n) \in U$。证明如下：设 $a = kn + r(k>=1, 0<r<n)$，由于 kn 不会与 n 互质，故 r 必须和 n 互质，而 $r < n$，故 $ r \in U$，即 $(a ~ mod ~ n) \in U$。因此得到一个结论，只要正整数 a 与 n 互质，则有 $(a ~mod ~ n) \in U$。
>
> - 接着我们可以证明若 a 与 n 互质，且 u 与 n 互质，则 $au % n$ 与 n 互质，继而可得到 $(au ~ mod ~ n) \in U$。这是为什么呢？因为根据唯一质数分解定理，a 和 u 都可以分解为质数乘积，如 $\small a = \prod_{i=1}^s p_i^{e_i}$，而 $\small u = \prod_{i=1}^t q_i^{f_i}$，同理，n也可以分解为质因数乘积 $\small n=\prod_{i=1}^m r_i^{g_i}$。而由 a 和 u都和 n 互质，则 a 和 n 的质因子没有交集，且 u 和 n 的质因子没有交集，则 au 的质因子与 n 没有交集，故而 au 和 n 互质，由前面的结论，可得到 $(au ~ mod ~ n) \in U$。
>
> - 将集合 U 中的数都乘以 a，可以得到新的集合 $AU$，即 $AU = \\{a u_1, a u_2, ..., a u_{φ(n)} \\}$，由前面分析可知 $AU$ 中的每个数都与 n 互质。所以，对于任意的 $ u_i \neq u_j(i,~ j \in U，i \neq j)$，可得到 $au_i ~ mod ~ n \neq au_j ~ mod ~ n$，即 $AU$ 中每个数除以 n 的余数各不相同。这个用反证法不难证明，若存在  $ u_i \neq u_j(i,~ j \in U，i \neq j)$，满足 $au_i ~mod~ n = au_j ~mod ~n$，则可以设 $au_i = sn + r, au_j = tn + r$，故可得 $a(u_i - u_j) = (s-t)n$，而 a 与 n 互质，则要满足条件，只能 $u_i - u_j$ 是 n 的倍数，而我们知道 $\|u_i - u_j\|  < n$，与假设矛盾。由 $AU$ 中每个数与 n 互质，且它们除以 n 的余数与 n 互质且各不相同，故而 $AU$ 中每个数除以 n 得到的余数就是集合 $U$ 中的那 $φ(n)$ 个数，当然顺序不一定是一一对应。因此 $AU ~mod~  n = U$。
>
> - 有了上一步的结论，再加上模除的乘法结合律和消去率，就可以证明欧拉定理了：
> $au_1 \cdot au_2...au_{φ(n)} \equiv u_1 \cdot u_2...u_{φ(n)}(mod ~n)$
> $a^{φ(n)}\cdot u_1 \cdot u_2...u_{φ(n)} \equiv u_1 \cdot u_2...u_{φ(n)}(mod ~ n)$
> $a^{φ(n)} \equiv 1(mod ~ n)$ 

### 1.9.3 费马小定理
费马小定理：给定整数a和质数p，若a不是p的倍数，那么：
$$a^{p-1} \equiv 1(mod ~ p)$$

费马小定理可以看做是欧拉定理的特例。因为当 p 为 质数时，a 不是 p的倍数，则 a与p互质，且 p 的欧拉函数 $φ(p) = p-1$，代入欧拉定理即可得证。

# 2 RSA算法原理
RSA算法基于欧拉定理而来，如果只是为了证明RSA算法原理，只要明白欧拉定理即可。当然，欧拉定理的证明涉及数论中许多经典的定理，有兴趣的同学可以深入了解证明过程。话说回来，即便不看欧拉定理的证明过程，只要明白欧拉定理内容，看懂RSA算法原理也是没有问题的。

## 2.1 RSA算法
1）生成两个随机的不相等的大质数 p 和 q，它们的积 $n = pq$ 的二进制位数就是密钥的位数，通常设置的位数有 1024，2048，3072，4096等。

> [知识点] 大质数生成算法
> 
> 如何生成这两个大质数，这是个值得研究的问题。首先可以想到的是生成质数序列，从中随机选取2个大质数即可。一种改进的方法是: 除了 2 外的偶数肯定不是质数，而奇数可能是质数，可能不是，那就可以跳过2与3的倍数，即对于 6n，6n+1, 6n+2, 6n+3, 6n+4, 6n+5，我们只需要判断 6n+1 与 6n+5 是否是质数即可。而判断某个数m是否是质数，最基本的方法就是对 2 到 m-1 之间的数除 m，如果有一个数能够整除 m，则 m 就不是质数。判断 m 是否是质数还可以进一步改进，只需要对 2 到 $\small \sqrt m$ 之间的数除 m 就可以。继续优化，其实只用 2 到 $\small \sqrt m $之间的质数去除即可。上面生成大质数是很基础的方法，效率比较低，一般会采用更快的方式，比如随机生成一个 nbits 位的奇数 p，然后从 p 开始遍历之后的奇数，并通过 [Miller–Rabin 质数判定算法](https://en.wikipedia.org/wiki/Miller%E2%80%93Rabin_primality_test)对遍历到的数字进行质数判定，如果是质数，即可返回(当然，该算法有一定的误判率，不过在判定次数设置够大的情况下，误判率基本可以忽略)。

2）计算 p 和 q 的积 $n=pq$，以及欧拉函数 $φ(n) = (p-1)(q-1)$
	
3）选择一个整数  $e，1< e < φ(n)$，且 $gcd(e, φ(n))$ = 1，即 $e$ 与 $φ(n)$ 互质，在 openssl 中 $e$ 固定为 65537。
> [知识点]费马数 
> 对于e，通常可选 3, 5, 17, 257 和 65537。因为它们都是质数，且二进制中只有2位是1，可以加快 d 的计算。这几个e的可选值实际是费马数的前5个数，费马数定义是 $F_x = 2^{2^x} + 1$，$F_0$ 到 $F_4$ 都是质数，但是从 $F_5$ 开始就不是质数了。例如 $F_5 = 4294967297 = 641×6700417$。实际应用中通常选择 $F_4 = 65537$  作为 e。

4）计算 e 对于 $φ(n)$ 的模反元素 d。即找到整数d，1 < d < φ，且满足 $\small ed \equiv 1~(mod ~φ(n))$。

$ed ≡ 1~(mod~ φ(n))  $
$\implies ed - 1 = kφ(n)$
$\implies ed + φ(n)k'= 1$
	
根据扩展欧几里得算法，e 和 φ(n) 为已知量，可以求得一组解 d, k'，其中 d 就是我们需要找的模反元素。

5）n 和 e 封装为公钥，n 和 d 封装为私钥。
- n：通常称为 modulus。
- e：通常称为 public exponent。
- d：通常称为 secret exponent。

6）定义好了公私钥，则对信息 $m(0 <= m < n)$，加密和解密函数如下：
$ c = RsaPublic(m) = m^e ~ mod ~ n$ (公钥 n, e 加密)
$ m =RsaPrivate(c) = c^d ~ mod ~ n$ (私钥 n, d 解密)

需要证明： 

- $m = RsaPrivate(RsaPublic(m))$  (1)
- $m = RsaPublic(RsaPrivate(m))$  (2)    

经过分析可以发现这两个式子证明是一样的，这里证明（1）即可。（1）常用于客户端公钥加密数据然后服务端私钥解密获取原始数据，而（2）则常用于服务端私钥加密签名数据，客户端公钥解密获取签名数据并校验签名。

## 2.2 实例分析
在证明之前，先简单验证下。假定我们选择 $p=13， q=15，e = 17$，则

- $n = 13\times15 = 195$
- $φ(n) = (13-1)(15-1) = 168$
- $17d + 168y = 1$，可以求得一个 $d=89, k=-9$。

假定我们原始数据 m=16，则加密后数据为 ，对 168 解密，可以得到原始数据 12。

- $c = m^e ~ \% n = 16 ^{17} ~ \% ~ 195 = 61$
- $m = c^d ~ \% ~ n = 61^{89} ~ \% ~195 =16 $

## 2.3 算法正确性证明
由上一节可知，我们要证明的是：
$$((m^e ~mod ~ n) ^ d) ~mod~ n = m (0 <= m < n) $$
由模运算的幂性质，等式左边:  $((m^e ~ mod ~ n) ^ d) ~mod ~n= m^{ed} ~mod~ n$，即我们要证明的是：
$$m^{ed} ~ mod ~ n = m \iff m^{ed} \equiv m (mod ~ n)$$
- 由前面RSA算法构造条件知道 $ed \equiv 1(mod ~ φ(n))$，据此可设 $ed = 1 + kφ(n)$
- 先证明 m 和 n 互质这种特殊情况。因为 m 和 n 互质，由欧拉定理有 $m^{φ(n)} \equiv 1(mod ~n)$。由此推导如下：
 $m^{ed} \equiv m^{1 + kφ(n)} $
$\equiv (m \cdot ({m^{φ(n)}})^k) $
$\equiv (m \cdot 1^k) $ (由欧拉定理和模运算的乘法结合律)
$\equiv m(mod ~n)$
- 下面证明的是 m 和 n 不互质的情况，因为 n = pq，且 p 和 q都是质数，则 m 与 n 不互质只能是 p 或者 q 的倍数，不能同时为 p 和 q的倍数，因为 m < n。假设 m 是 q 的倍数，即 $q\|m$，则 $m \equiv 0 ( mod ~ q)$。可得 $m^{1 + kφ(n)} \equiv m \equiv 0 ~(mod ~ q)$。

  而因为 p 是质数，m 又不是 p 的倍数，故 p 和 m 互质。由费马小定理，可得到 :
$m^{p-1} \equiv 1 (mod ~ p)$
$\implies m^{(p-1)(q-1)} \equiv 1^{q-1} \equiv 1 (mod ~ p)$ (两边 q-1 的幂次)
$\implies m^{φ(n)} \equiv 1 (mod ~ p) $ (因为 $\small(φ(n) = φ(pq) = (p-1)(q-1)$)
$\implies m^{kφ(n)} \equiv 1^k \equiv 1(mod ~ p)$ (两边 k 次幂)
$\implies m^{1+kφ(n)} \equiv m(mod ~ p)$ (两边乘以 m)

  由 $m^{1+kφ(n)} \equiv m(mod ~ p)$ 以及 $ m^{1+kφ(n)} \equiv m(mod ~ q)$ ，从而可得 $m^{1+kφ(n)} \equiv m(mod ~ pq)$，因为 $n=pq$，故 $m^{1+kφ(n)} \equiv m(mod ~ n)$ ，因为 $ed = 1 + kφ(n)$，根据性质 1.11，故而得证 $m^{ed} \equiv m(mod ~ n)$。

- **注意：** 虽然定义中 $0 <= m < n$，但是实际上对于 $m = 0，1，n-1$ 加密并没有效果，很容易从加密过程知道，$0，1，n-1$ 加密后都是它们自身。

## 2.4 RSA Key Pair的生成方法及代码实现
RSA的Key Pair的生成方法的伪代码如下：


> INPUT:  模数 (即前面提到的N，N=pq) 的二进制位数 k
>
> OUTPUT:  RSA的密钥对 ((N,e),d)，其中 N 是 模数,  N=pq，其二进制长度不超过 k bit。 e 是选择的 public exponent，d 是 secret exponent。
> 
> ALG：RSA密钥对生成
>
> $\small Select ~a ~value ~of~ e \in 3,5,17,257,65537$
>
> $\small repeat$
>
> $\small \quad p \leftarrow genprime(k/2)$
>
> $\small until ~ (p~ mod ~ e) \neq 1$
>
> $\small repeat$
>
> $\small \quad q \leftarrow genprime(k - k/2)$
>
> $\small until ~ (q ~ mod ~ e) \neq 1$
>
> $\small N \leftarrow pq$
>
> $\small L \leftarrow (p-1)(q-1)$
>
> $\small d \leftarrow modinv(e, L)$
>
> $\small return ~ (N,e,d)$

1）首先就是找两个位数为 k/2 的大于0的大质数。在前面提到过，最朴素的方法是遍历正整数集，对于数 n，用 $\small2, 3, ..., \sqrt n$ 去除 n，但是这个方法的时间复杂度为 $\small \Theta(\sqrt n)$，这是正整数 n 的长度的幂(因为 n 可以表示为 b 位的二进制数，则 $ \small n = \lceil lg(n+1) \rceil$，故而 $\small \sqrt n = \Theta(2^{b/2})$)。这个复杂度是比较高的，而我们不用遍历所有的整数，完全可以随机找一个大整数，然后判断该数是不是质数即可。令人高兴的是，判断一个数是不是质数比对一个数进行质数因子分解要容易很多。

由费马小定理知道，如果 n 是 质数，则对于任意的数 $\small 1 <= a <= n-1$ ，都满足
$$\small a^{n-1} \equiv 1(mod~ n)(2.4.1)$$
则可以知道如果对于 $\small 1, 2, ..., n-1$ 中如果有一个数 a  和 n 不满足 2.4.1，则 n 一定是合数。当然我们在实际中也不会将所有小于 n 的正整数都遍历来计算一遍，通常会随机选取一个 a 作为基数，比如选择 a=2，则满足费马小定理而又不是质数的 n ，我们称 n 为基于 a 的一个伪质数。比如 341 满足 $\small 2^{(341-1)} \equiv 1(mod ~ 341)$，但是 341 是一个合数。如果将基数a换成3，那也不能保证，对于 $\small \forall a \in \\{1, 2, ...n-1\\}$，总有些合数(这些数也被称为 $\small Carmichael$ 数)满足2.4.1，如 561(a=2 满足)，1105(a=2和a=3都满足)等。

为此，我们可以使用 Miller-Rabin 算法来解决判断质数问题。其主要思想就是选取 s 个 a 的随机数，然后判断奇数 n 是否是质数(偶数可以直接排除)，如果其中任意一个 a 对应的 WITNESS(a, n) 为 true，则表示 n 肯定是合数。如果 s 次测试都通过，则可以近似判定 n 是 质数(误判的概率最大为 $\small 1/2^s$ ，一般实际应用中选s=50就可以了)。

> 
>INPUT: (n, s)，n > 2，n 是奇数，s 是测试次数
> OUTPUT: COMOSITE 表示为合数，PRIME 则表示极可能是质数
>
> $\small MILLER-RABIN(n, s)$
>
> $\small \quad for ~ j ~ 1 \leftarrow to ~ s$
>
> $\small \quad \quad do ~ a \leftarrow RANDOM(1, n-1)$
>
> $\small \quad \quad \quad if ~ WITNESS(a, n)$
>
> $\small \quad \quad \quad \quad then ~ return ~ COMPOSITE$ // 肯定不是质数
>
> $\small \quad return ~ PRIME$ // 极可能是质数


>INPUT: (a, n) 其中 a 为随机选择的基数，n为待判定的奇数
>OUTPUT: TRUE 表示肯定是合数， FALSE 表示可能是质数
> 
> $\small WITNESS(a, n)$
>
> $\small \quad let ~ n-1 = 2^tu, where ~ t >=1 ~ and ~ u ~ is ~ odd$
>
> $\small \quad x_0 = a^u ~ mod ~ n$
>
> $\small \quad for ~ i \leftarrow 1~ to ~ t$
>
> $\small \quad \quad do ~ x_i \leftarrow x_{i-1}^2 ~ mod ~ n$
>
> $\small \quad \quad \quad if ~ x_i = 1 ~ and ~ x_{i-1} \neq 1 ~ and ~  x_{i-1} \neq n-1$
>
> $\small \quad \quad \quad \quad then ~ return ~ TRUE$
>
> $\small \quad if ~ x_i \neq 1$
>
> $\small \quad \quad then ~ return ~ TRUE$
>
> $\small \quad return ~ FALSE$

而 WITNESS 就是根据 2.4.1 来判断 n 是否通过检测，不过做了一些优化，因为 n 是奇数，所以可以将 n-1分解为 $\small n-1 = 2^tu(t>=1, u为奇数)$，设 $\small x_0 = a^u ~ mod ~ n$，则对 x 的结果平方 t 次，即可计算出 $\small a^{n-1} ~ mod ~ n$。所计算的序列 $x_0, x_1, ...x_t$ 满足 $\small x_i \equiv a^{2^i}u(mod ~ n)(i = 0, 1, ...t)$。 如果前一个值 $\small x_{i-1}$ 不等于 1 或者 n-1，但是当前值 $\small x_{i} = 1$，则 n 一定是合数。证明如下：

> $\small WITNESS$ 正确性证明：
> - 首先可以证明这么一个结论：若 a ，n为正整数，且 n 为质数，$\small a^2 \equiv 1 (mod ~n)$，则有 $\small a \equiv \pm1(mod ~ n)$。
> - 这是因为 
  $\small a^2 \equiv 1(mod ~n) \implies a^2 - 1 \equiv 0(mod ~n)$
  $\small \quad \implies (a+1)(a-1) \equiv 0(mod ~ n)$
  $\small \quad \implies a + 1 \equiv 0(mod ~ n)$ 或者 $\small a - 1 \equiv 0(mod ~ n)$
  $\small \quad \implies a \equiv \pm1(mod ~ n)$(因为 $\small a > 0$)
> - 下面就很容易证明 WITNESS了，因为 $\small n-1 = a^{2^i}u$，如果 n 是质数的话，则由 $\small x_i = 1$，可以推出 $x_{i-1} $ 一定是1 或者 n-1，否则 n一定是合数。

2）第二步就是计算 N=pq，然后计算欧拉函数 L，最后根据 e 和 欧拉函数 L 计算出 e 模除 L 的模拟元素 d 即可。

3）基于该算法，我写了个 RSA 密钥生成的 python 实现供大家参考，见 [shishujuan:rsa-algrithm](https://github.com/shishujuan/rsa-algrithm) 。

## 2.5 提高 RSA 加解密效率
由前面分析知道 RSA 加解密效率严重依赖求幂和求模运算效率，为了减少大整数的求幂和求模的运算，可以对 RSA 模幂算法(modular exponentiation)进行一些优化。主要包括两方面，一是对模幂运算本身进行优化，提升运算效率。而是对RSA算法进行优化，基于中国剩余定理来提升RSA加解密效率。

加密：$c = m^e ~ mod ~ n$
解密：$m = c^d ~ mod ~ n$

### 2.5.1 提升模幂运算性能
**模幂运算-朴素法**

朴素法求模幂就是直接求幂然后模除，如下：

```
def pow_simple(a, e, n):
    """
    朴素法模幂运算：a^e % n
    """
    ret = 1
    for _ in xrange(e):
        ret *= a
    return ret % n
```

当幂 b 很大时，这个方法的模幂运算就很慢，在我的机器上 a=5, e=102400, n = 13284 大概需要1秒左右的时间。我们知道 RSA 私钥中的 d 的值可是远大于 102400 的，如果这么慢显然效率上会有问题。

一种简单的优化方法基于下面这个性质，可以很方便的迭代实现，而且每次计算会将参数减小，优化后的模幂运算效率大概提升了100倍左右。

$a \equiv c(mod ~ n)  \implies ab \equiv bc(mod ~n)$
$\implies ab ~mod~n = (b(a ~ mod ~ n))(mod ~ n)$

代码如下：

```
def pow_simple_optimized(a, e, n):
    """
    朴素法模幂运算优化：基于 a ≡ c(mod n) => ab ≡ bc(mod n)，
    即 ab mod n = (b*(a mod n)) mod m
    """
    ret = 1
    c = a % n
    for _ in xrange(e):
        ret = (ret * c) % n
    return ret
```

**模幂运算-二进制法**

虽然优化过的朴素方法的效率已经有了大幅提升，但是对于RSA私钥的超大的d值，仍然性能堪忧。比如当幂级数 e 大小为亿级别时，优化过的朴素方法在我的机器需要接近10秒的时间求模。一种广为应用的模幂方法就是二进制法，基于位运算效率会有惊人的提升，e为亿级别时模幂运算也只需要零点几毫秒。

对于一个n位的幂级数 e，我们可以写成如下二进制形式：
$e = e_{n-1}e_{n-2}...e_1e_0 = \prod_{i=0}^{n-1} e_i2^i$ (其中 $e_{n-1}$ 是最高位，$e_0$是最低位)
$\implies a^e = a^{\prod_{i=0}^{n-1}e_i2^i} = \prod_{i=0}^{n-1}(a^{2^i})^{e_i}$
$\implies c \equiv  \prod_{i=0}^{n-1}(a^{2^i})^{e_i} (~mod ~ n)$

从最低位 $e_0$ 开始遍历 $e_i(i=0, 1, ...n-1)$，如果 $e_i = 0$，则不影响模除结果，设置base值，继续处理下一位即可。在扫描到 $e_i$ 时，有 $base = a^{2^i} ~mod~ n$，根据模除的乘法结合律 $(a * b) ~mod ~n = ((a ~mod~ n) * (b ~mod~ n)) ~mod ~ n$，可以知道下面算法是正确的。Python的内建函数 $pow(x, y, z)$ 就是使用了类似的算法，因此在模幂运算时，使用 $pow(x, y, z)$ 函数比 $x**y ~\%~ z$ (直接幂运算跟优化过的朴素法性能相当)性能要高出几个数量级。

```
def pow_binary(a, e, n):
    """
    right-to-left binary method:基于位运算模幂运算优化。
    """
    number = 1
    base = a
    while e:
        if e & 1:
            number = number * base % n
        e >>= 1
        base = base * base % n
    return number
```

完整的代码见 [pow.py](https://github.com/shishujuan/rsa-algrithm/blob/master/pow.py)。

### 2.5.2 分解大整数提升模幂运算效率
从前面分析知道，通常使用过程中公钥的幂级数 e 的选取比较小，而且选取的是费马数，只有2位是1，其他位是0。而私钥的幂级数 d 很大，它的位数与模数 N 差不多，且有接近一半位是1。为了提高私钥 d 在模幂运算中性能，可以基于中国剩余定理(CRT)进行相关优化。

> 使用中国剩余定理提升私钥模幂运算效率(其中 $n^{-1} _m$ 代表 n 对模数 m 的模逆元素，即 $n^{-1}_m n \equiv 1(mod ~m)$)

- $dP = e^{-1} ~ mod ~ (p-1) = d ~mod~ (p-1)$ 
- $dQ = e^{-1} ~mod~ (q-1) = d ~mod ~ (q-1) $
- $m1 = c^{dP} ~ mod ~ p $
- $m2 = c^{dQ} ~ mod ~ q $
- $qInv = q^{-1}_p$
- $h = qInv \cdot (m1 - m2) ~mod~ p $
- $m = m2 + h \cdot q$

这里多出来的 dP，dQ，qInv的值在使用 openssl 生成的 rsa 密钥中也有，其目的就是为了加速私钥模幂运算。以 2.2 中的例子为例，直接求 m 可得 $\small m = c^d ~mod ~n = 61^{89} ~mod~ 195 = 16$，而使用中国剩余定理优化后求解流程如下，亦可得到正确的结果 m = 16。

- $dP = d ~ mod~ (p-1) = 89 ~ mod~(13-1) = 5$
- $dQ = d ~ mod ~ (q-1) = 89 ~ mod ~ (15-1) = 5$
- $m1 = c^{dP} ~ mod ~ p = 61^5 ~ mod ~ 13 = 3$
- $m2 = c^{dQ} ~ mod ~ q = 61^5 ~ mod ~ 15 = 1$
- $qInv = q^{-1}_p = 7$
- $h = qInv \cdot (m1-m2) ~ mod ~ p = 7 \cdot(3-1) ~ mod ~ 13 = 1$
- $m = m2 + h \cdot q = 1 + 1 \cdot 15 = 16$

> [证明] 
> - 由中国剩余定理知道，对于 $x ~ mod ~ p = x_1, x ~ mod ~ q = x_2(0 <= x_1 < p, 0 <= x_2 < q)$ 这个方程组在 p 和 q互质时，则必然存在一个唯一解 $0 =< x < pq$。因为 $n=pq$， 因此对于任意的 $ 0 <= x < n$，必然存在唯一的数对 $<x_1, x_2>$ 使之满足方程组。我们要计算的是 $m = c^d ~ mod ~ n=c^d ~mod~ pq$，如果我们知道了 $<c^d ~ mod ~ p, ~ c^d ~ mod ~ q>$，则由中国剩余定理知，必然存在一个唯一的值 $ c^d ~mod~ n$ 与之对应。
>
> - 由 1.12 中中国剩余定理的解的格式可知，若已知 
$c^d ~ mod ~ p = m_1，c^d ~ mod ~ q = m_2，n=pq$，则有
$x = c^d ~ mod ~n = m_2  + h \cdot q$
> $ h = (q^{-1}_p(m_1 - m_2)) ~ mod ~ p$ 
> 
> - 可以发现这与前面计算 $m_1$ 并不是直接用的 $c^d ~ mod ~ p$，而是 $c^{d ~mod ~(p-1)} ~mod~ p$，这是为何呢？为什么会有 
> $$c^d \equiv c^{d ~ mod ~ (p-1)} ~ (mod ~p)$$
> 这个可以用欧拉定理证明：设 $ d = kφ(p) + d ~mod~ φ(p)$，则有
> $$c^d = c^{kφ(p) + d ~mod~ φ(p)} = (c^{φ(p)})^k \cdot c^{d ~mod ~φ(p)}$$ 
> 由欧拉定理知道 
> $$c^{φ(p)} \equiv 1(mod ~ p)$$ 
> 于是有：
> $$c^d \equiv 1^k \cdot c^{d ~mod ~φ(p)} \equiv c^{d ~mod ~φ(p)} \equiv c^{d ~mod ~(p-1)} (mod ~ p)$$ 
> 得证。

使用剩余定理优化的python实现参见 [rsa.py](https://github.com/shishujuan/rsa-algrithm/blob/master/rsa.py) 中的 `decrypt_crt` 函数。

## 2.6 如何破解 RSA？
从RSA加解密原理可知，如果要破解RSA，那就是知道 d 即可，由 $\small ed \equiv 1(mod ~ n)$ 可知，要知道 d，就需要 e 和 φ(n)，e 是公开的，而 $\small φ(n) = (p-1)(q-1) $是未知的，想知道 φ(n) 就要知道 p 和 q 这两个大质数的值。我们知道 n 也是公开的，$\small n=pq$，所以只要能将 n 因式分解即可破解RSA算法。如果 N 是个小整数，那很好办，比如 N = 25777，知道 $\small \sqrt[2]{25777} < 161$ ，则可以从 161 开始找比它小的质数，然后判断是不是可以整除即可，很快可以得到 $\small 25777 = 149 * 173$。

不过不要担心，大整数的质数因式分解还是比较难的，虽然除了暴力破解外，还有一些效率比较高的整数因式分解方法，如 [二次筛分算法](http://mathworld.wolfram.com/QuadraticSieve.html) 和 [普通数域筛选法（GNFS）](https://en.wikipedia.org/wiki/General_number_field_sieve) 等，当你的RSA密钥位数在4096位以上，使用常规运算能力的电脑还是较难破解的。当然不排除某一天，数论研究出现了新的成果，若有一种通项公式可以分解大整数的话，那现有基于 RSA 的加密体系都会崩塌。此外，RSA 加密用的 e 不能太小，否则有潜在的风险，实际应用中常用 65537。

如果已知 N，e，d，可以很高效地对 N 实现因式分解，原理就不赘述了，实现代码见 [factor.py](https://github.com/shishujuan/rsa-algrithm/blob/master/factor.py) 。



# 3 RSA 密钥格式实例分析
本节对使用 openssl 等工具生成的 RSA 密钥格式进行简单分析，首先，使用 openssl 生成一对 RSA 密钥，如下：

```
# openssl genrsa -out rsa_demo.key 2048
# openssl rsa -in rsa_demo.key -pubout -out rsa_demo.pub
```

示例的私钥文件 [rsa_demo.key](https://github.com/shishujuan/rsa-algrithm/blob/master/rsa_key_pair_demo/rsa_demo.key)，公钥文件 [rsa_demo.pub](https://github.com/shishujuan/rsa-algrithm/blob/master/rsa_key_pair_demo/rsa_demo.pub)。

最常用的 RSA 密钥的模式是 PKCS#1 中规定的模式，即公钥包括 `N, e`，而私钥包括 `N, e, d, p, q, dP, dQ, qInv`。查看公私钥文件内容，可以看到采用的是 ASN.1 中的 DER(Distinguished Encoding Rules) 编码的，公钥解码后内容如下：

```
# grep -v '\-\-\-\-\-' rsa_demo.pub \| tr -d '\n'\|base64 -D\|hexdump
0000000 30 82 01 22 30 0d 06 09 2a 86 48 86 f7 0d 01 01
0000010 01 05 00 03 82 01 0f 00 30 82 01 0a 02 82 01 01
0000020 00 bd e4 43 1a d0 02 1e e6 12 34 14 91 84 4d 65
......
0000120 b1 02 03 01 00 01                              
0000126
```

DER 编码包括四部分：对象标识，数据长度域，数据域以及结束标志。不过在 openssl 生成的 rsa 密钥的编码中没有结束标志，在示例的 RSA 公钥中， 0x30 是 `Sequence` 类型标志，0x03 是 `Bit String` 类型标志，而 `0x02` 是 `Integer` 类型标志，`0x06`是 `Object ID`类型标识。长度如果超过255，则会先跟一个 `0x82`，然后是数据长度的值，没超过则就是长度值。整个结构是一个嵌套结构，公钥的 n 和 e 两个数值位于 `BIT STRING` 这块数据中，可以分析知道从 `00000020`开始的 `00bde4...b1`是 n，而从 `0000123`开始的 `010001` 是 e，即 65537。另外，`Object ID`是标识密钥的元信息的地方，对应的是 `2a 86 48 86 f7 0d 01 01 01` 这 9 个字节，这个代表什么含义呢？其实转义过来是 `1.2.840.113549.1.1.1`，这个 OID 表示的是 RSA 加密系统的公钥。私钥格式类似，这里不再赘述，openssl 生成的私钥中包括了 `p,q,dP,dQ,n,e,d,qInv`数据。

如果不想了解公私钥的格式，直接通过 openssl 的工具即可解析出公私钥的内容，容易验证，这些值正是基于前面的 RSA 原理计算得来的。

```
# openssl rsa -in rsa_demo.key -noout -text
Private-Key: (2048 bit)
modulus: # N
    00:bd:e4:43:1a:d0:02:1e:e6:12:34:14:91:84:4d:
    ...
publicExponent: 65537 (0x10001) # e
privateExponent: # d
    76:bb:8d:41:ec:a2:06:d3:f0:b9:e3:ca:81:21:2b:
    ...
prime1: # p
    00:fa:99:34:b8:b3:97:dc:09:37:29:dd:df:de:86:
    ...
prime2: # q
    00:c1:fc:13:f7:84:dd:f4:b5:05:2d:89:b9:a1:50:
    ...
exponent1: # dP
    00:c9:7d:61:cc:98:6a:23:bb:2d:25:76:86:47:cf:
    ...
exponent2: # dQ
    00:99:e8:43:7b:45:ea:c8:45:7b:57:37:07:95:ea:
    ...
coefficient: # qInv
    0c:53:5f:0c:a6:7b:33:69:32:b6:b9:65:f8:31:5f:
    ...
    
# openssl rsa -pubin -in rsa_demo.pub -noout -text
Public-Key: (2048 bit)
Modulus: # N
    00:bd:e4:43:1a:d0:02:1e:e6:12:34:14:91:84:4d:
    ...
Exponent: 65537 (0x10001) # e
```


# 参考资料
- 《算法导论》第31章
- 《具体数学》第4章
- [di-mgt:rsa_alg](https://www.di-mgt.com.au/rsa_alg.html)
- [di-mgt:rsa_theory](https://www.di-mgt.com.au/rsa_theory.html)
- [di-mgt:rsa_factorize_n](https://www.di-mgt.com.au/rsa_factorize_n.html)
- [di-mgt:crt](https://www.di-mgt.com.au/crt.html)
- [di-mgt:RSA算法的C语言实现](https://www.di-mgt.com.au/bigdigits.html)
- [wiki: 中国剩余定理](https://zh.wikipedia.org/wiki/%E4%B8%AD%E5%9B%BD%E5%89%A9%E4%BD%99%E5%AE%9A%E7%90%86)
- [欧拉函数](http://oeis.org/wiki/Euler%27s_totient_function)
- [关于欧拉函数及其一些性质的美妙证明（2）](https://zhuanlan.zhihu.com/p/37067555)
- [关于欧拉函数及其一些性质的美妙证明(1)
](https://zhuanlan.zhihu.com/p/36979522)
- [wiki: Modular_exponentiation](https://en.wikipedia.org/wiki/Modular_exponentiation)
- [pkcs--10-encoded-asn-1](https://docs.microsoft.com/zh-cn/windows/desktop/SecCertEnroll/pkcs--10-encoded-asn-1)
- [how-is-oid-2a-86-48-86-f7-0d-parsed-as-1-2-840-113549](https://crypto.stackexchange.com/questions/29115/how-is-oid-2a-86-48-86-f7-0d-parsed-as-1-2-840-113549)
- [latex公式参考资料](https://www.zybuluo.com/codeep/note/163962)

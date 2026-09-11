---
title: Uniswap
date: 2026-09-10T16:33:12+08:00
draft: false
description: Uniswap v1/v2 恒定乘积，v3 集中流动性，以及 v4 的架构变化
math: true
categories:
  - 技术教程
tags:
  - uniswap
  - defi
---

<!--more-->

# Uniswap

Uniswap 是以太坊上 DeFi 的基石。它用 AMM（自动做市商）代替传统订单簿，让用户在没有中间做市商的情况下直接兑换代币。

核心演进可以看成三步：

- **v1 / v2**：全区间恒定乘积 \(xy = k\)
- **v3**：把流动性集中到指定价格区间
- **v4**：AMM 算法基本不变，改的是合约架构（单体、闪电记账、hooks）

## Uniswap v1 / v2

v1 只允许 `ETH / ERC20` 组成交易对。v2 放开了限制，`ERC20 / ERC20` 也可以组 pair。

兑换、加池、撤池都围绕同一个公式：

$$
x \cdot y = k
$$

\(x\)、\(y\) 是池子里两种代币的储备，\(k\) 在一笔 swap 过程中保持不变（手续费会让 \(k\) 缓慢变大）。

下面用一组贯穿的数字：

```text
x = 2000
y = 100
k = 2000 * 100 = 200000
总 LP = 1000
```

### 添加流动性

添加流动性前后要保持 \(x / y\) 的比例不变。想加 \(100\) 个 \(x\)，就必须同时加 \(5\) 个 \(y\)：

$$
\frac{100}{2000} = \frac{5}{100}
$$

获得的 LP 数量取两侧份额的较小值：

$$
\begin{aligned}
\mathrm{lp}_x &= \frac{100}{2000} \times 1000 = 50 \\
\mathrm{lp}_y &= \frac{5}{100} \times 1000 = 50 \\
\mathrm{lp}   &= \min(\mathrm{lp}_x, \mathrm{lp}_y) = 50
\end{aligned}
$$

对应 v2 合约里的：

$$
L = \min\left(
  \frac{\Delta x \cdot S}{x},\;
  \frac{\Delta y \cdot S}{y}
\right)
$$

其中 \(S\) 是当前 `totalSupply`。

首次添加流动性时还没有 \(S\)，公式换成：

$$
\mathrm{lp} = \sqrt{xy} - L_{\min}
$$

\(L_{\min}\) 即 `MINIMUM_LIQUIDITY`，默认是 \(1000\)，会永久锁到 `address(0)`，避免池子被抽空后价格被任意操纵。

加完后池子变成：

```text
x = 2100
y = 105
总 LP = 1050
```

### 兑换

兑换前后保持 \(k\) 不变，但协议会先从输入里抽走 **0.3%** 手续费。

用户用 \(100\) 个 \(x\) 换 \(y\)：

$$
\begin{aligned}
2100 \times 105 &= \bigl(2100 + 100 \times (1 - 0.003)\bigr) \times (105 - \Delta y) \\
\Delta y &\approx 4.759
\end{aligned}
$$

等价于合约里的 \(997 / 1000\) 算法：

$$
\begin{aligned}
\text{amountInWithFee} &= 100 \times 997 \\
\text{amountOut}
  &= \frac{\text{amountInWithFee} \times 105}
         {2100 \times 1000 + \text{amountInWithFee}}
  \approx 4.759
\end{aligned}
$$

手续费留在池子里，所以实际 \(k\) 会略微增大，这就是 LP 的收益来源。

### 移除流动性

销毁自己持有的 LP，按份额取回两种代币。假设用户有 \(100\) LP，当时总供应 \(1000\)，池子 \(x = 2000\)、\(y = 100\)：

$$
\begin{aligned}
x_{\mathrm{out}} &= \frac{100}{1000} \times 2000 = 200 \\
y_{\mathrm{out}} &= \frac{100}{1000} \times 100 = 10
\end{aligned}
$$

## Uniswap v3

v2 的流动性均匀铺在 \((0, +\infty)\) 上，大部分永远用不上。v3 允许 LP 把流动性集中到自己选择的价格区间，同样的资金能提供更高的深度。

价格被离散成 tick：

$$
P(\mathrm{tick}) = 1.0001^{\mathrm{tick}}, \quad \mathrm{tick} \in [-887272,\, 887272]
$$

每个 fee tier 对应一个 `tickSpacing`，流动性只能加在能被 `tickSpacing` 整除的 tick 上：

| fee | tickSpacing |
| --- | ----------- |
| 0.01% | 1 |
| 0.05% | 10 |
| 0.30% | 60 |
| 1.00% | 200 |

v3 不再发可替换的 ERC20 LP token，每个头寸是一张 NFT（`tokenId`），因为不同区间、不同手续费档位无法互相替代。

### 添加流动性

区间内的虚拟流动性 \(L\) 和两种代币数量的关系：

当前价格在 \([P_a, P_b]\) 内：

$$
\begin{aligned}
\Delta y &= L \bigl(\sqrt{P} - \sqrt{P_a}\bigr) \\
\Delta x &= L \left(\frac{1}{\sqrt{P}} - \frac{1}{\sqrt{P_b}}\right)
\end{aligned}
$$

当前价格 \(P < P_a\)（区间在上方，只需要 \(x\)）：

$$
\Delta x = L \left(\frac{1}{\sqrt{P_a}} - \frac{1}{\sqrt{P_b}}\right), \quad \Delta y = 0
$$

当前价格 \(P > P_b\)（区间在下方，只需要 \(y\)）：

$$
\Delta y = L \bigl(\sqrt{P_b} - \sqrt{P_a}\bigr), \quad \Delta x = 0
$$

链上用 \(\mathrm{sqrtPriceX96} = \sqrt{P} \cdot 2^{96}\) 存价格，所以实际计算会多一个 \(Q96 = 2^{96}\) 的定点缩放：

$$
\begin{aligned}
\Delta y &= L \cdot \frac{\mathrm{sqrtPriceX96} - \mathrm{sqrtLowerX96}}{Q96} \\
\Delta x &= L \cdot Q96 \cdot
  \frac{\mathrm{sqrtUpperX96} - \mathrm{sqrtPriceX96}}
       {\mathrm{sqrtPriceX96} \cdot \mathrm{sqrtUpperX96}}
\end{aligned}
$$

例子：

```text
tickSpacing = 60
currentTick = 0          # 当前价格 = 1
tickLower   = -60
tickUpper   = 60
amount0     = 100

sqrtPriceX96(-60) = 78990846045029531151608375685
sqrtPriceX96(0)   = 79228162514264337593543950336
sqrtPriceX96(60)  = 79466191966197645195421774832
```

由 \(\Delta x = 100\) 反解 \(L\)，再代入 \(\Delta y\) 公式，整数除法得到 \(\Delta y = 99\)。区间关于 \(P = 1\) 对称，实数计算下两种代币几乎相等；链上 floor 之后差 1 个 wei 量级的单位。

### Swap

v3 的 swap 不再一次走完整条曲线，而是沿着 tick 一段一段走：

1. 确定方向：\(x \to y\)（`zeroForOne`）还是 \(y \to x\)
2. 沿该方向找到下一个**已初始化**（有人挂过流动性）的 tick
3. 用当前这段的 \(L\) 和上面的 \(\Delta x / \Delta y\) 公式，算出这段最多能换多少
4. 输入在这段就耗尽：直接结束
5. 还有剩余：跨过这个 tick，更新当前 \(L\)（加上 `liquidityNet`），继续下一段

跨 tick 时还要翻转该 tick 的 `feeGrowthOutside`，供后面计算区间内手续费用。

### 移除流动性

和添加是同一套公式，只是方向相反。同样要看当前价格落在哪：

在区间内：

$$
\begin{aligned}
\Delta x &= L \left(\frac{1}{\sqrt{P}} - \frac{1}{\sqrt{P_b}}\right) \\
\Delta y &= L \bigl(\sqrt{P} - \sqrt{P_a}\bigr)
\end{aligned}
$$

高于上限（只能取回 \(y\)）：

$$
\Delta y = L \bigl(\sqrt{P_b} - \sqrt{P_a}\bigr)
$$

低于下限（只能取回 \(x\)）：

$$
\Delta x = L \left(\frac{1}{\sqrt{P_a}} - \frac{1}{\sqrt{P_b}}\right)
$$

价格已经走出区间后，头寸不再承担成交，但之前累计的手续费仍可以领取。

### 手续费记账

v2 的手续费直接混进储备里，所有 LP 按份额均分。v3 的头寸区间不同，必须按「这段时间有没有在范围内」分别记账。分三层：

| 层级 | 记什么 |
| --- | --- |
| Pool | `feeGrowthGlobal0/1`：全池每单位流动性累计的手续费 |
| Tick | `feeGrowthOutside0/1`：该 tick **外侧**累计的手续费 |
| Position | 上次结算时的 `feeGrowthInside` |

一个 \([\mathrm{tickLower}, \mathrm{tickUpper}]\) 区间内部累计的手续费：

$$
\begin{aligned}
\text{feeGrowthInside}
  &= \text{feeGrowthGlobal} \\
  &\quad - \text{feeGrowthOutside}(\mathrm{lower}) \\
  &\quad - \text{feeGrowthOutside}(\mathrm{upper})
\end{aligned}
$$

`outside` 的含义取决于当前 tick 在该 tick 的哪一侧，跨 tick 时会翻转一次。头寸未领取的手续费：

$$
\text{unclaimed} = L \cdot (\text{feeGrowthInside}_{\mathrm{now}} - \text{feeGrowthInside}_{\mathrm{last}})
$$

## Uniswap v4

v4 的 AMM 数学和 v3 一样，仍然是集中流动性 + tick。变化在合约架构，主要是三点。

### 单体架构（Singleton）

v3 每个 pool 单独部署一个合约。v4 把所有池子放进同一个 `PoolManager`，用

```text
PoolId = hash(currency0, currency1, fee, tickSpacing, hooks)
```

区分。少了一次次 `CREATE2` 部署，创建池子和跨池路由的 gas 都更低。

### 闪电记账（Flash Accounting）

v3 每一步都立刻 `transfer` ERC20。v4 在一次 `lock` / `unlock` 里只记余额 delta（用 transient storage），最后再统一 `settle` / `take`。中间的加池、swap、跨池操作可以互相抵消，少很多次代币转账。

约束是：`unlock` 时所有 currency 的 delta 必须归零，否则 revert。

### Hooks

每个池子可以绑定一个 hook 合约，在生命周期前后插入自定义逻辑，例如：

- `before/afterInitialize`
- `before/afterSwap`
- `before/afterAddLiquidity` / `before/afterRemoveLiquidity`
- `before/afterDonate`

动态手续费、内部限价单、自定义 oracle、挂钩稳定币曲线，都可以做在 hook 里，而不用分叉整个 AMM。

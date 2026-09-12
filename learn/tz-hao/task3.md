# Task 3：使用 DEX Oracle 获取代币价格

> 修订说明（2026-09-09）：以下“最终实现与链上证据”替代本文件后半部分的首次尝试。首次尝试是自建 AMM，**不作为本 Task3 提交或验收依据**；保留仅用于说明修订原因。

## 最终实现与链上证据

### 1. 真实 Fuji DEX

本次使用 **Pangolin V2（Avalanche Fuji，Chain ID `43113`）**，而不是自建 AMM。

| 项目 | 地址 |
| --- | --- |
| Pangolin Factory | [`0xE4A575550C2b460d2307b82dCd7aFe84AD1484dd`](https://testnet.snowtrace.io/address/0xE4A575550C2b460d2307b82dCd7aFe84AD1484dd) |
| Pangolin Router | [`0x2D99ABD9008Dc933ff5c0CD271B88309593aB921`](https://testnet.snowtrace.io/address/0x2D99ABD9008Dc933ff5c0CD271B88309593aB921) |
| WAVAX | [`0xd00ae08403B9bbb9124bB305C09058E32C39A48c`](https://testnet.snowtrace.io/address/0xd00ae08403B9bbb9124bB305C09058E32C39A48c) |
| ABT | [`0xab8545161af1de74ef75f1cef4a7a1ceee596d49`](https://testnet.snowtrace.io/address/0xab8545161af1de74ef75f1cef4a7a1ceee596d49) |
| ABT/WAVAX Pair | [`0x0307bD1D36b65DeE1652c37490aB89201983Dca3`](https://testnet.snowtrace.io/address/0x0307bD1D36b65DeE1652c37490aB89201983Dca3) |

### 2. 真实流动性

通过 Pangolin Router 的 `addLiquidityAVAX` 成功注入 **100,000 ABT + 0.05 AVAX**：

- [添加流动性交易](https://testnet.snowtrace.io/tx/0xc0ea3c79ad835a8c5149ebcbd40581139cdeb13fd2f4c754f2f658bd89ddb8ad)，回执状态 `1`。
- Pair 当前 `getReserves()`：`100000000000000000000000` ABT / `50000000000000000` WAVAX，即 **100,000 ABT / 0.05 WAVAX**。

### 3. 独立业务合约：Router 价格直接决定发放量

部署的业务合约：[`PangolinPricedAbtSale`](https://testnet.snowtrace.io/address/0xdBb76af289bE5aE0C893fc12bE6d3Eb23652F93D)。构造参数已链上读取并确认：`saleToken = ABT`、`router = Pangolin Router`、`wavax = WAVAX`。

```solidity
function quoteAbtForAvax(uint256 avaxAmount) public view returns (uint256 abtAmount) {
    address[] memory path = new address[](2);
    path[0] = wavax;
    path[1] = address(saleToken);
    uint256[] memory amounts = router.getAmountsOut(avaxAmount, path);
    abtAmount = amounts[1];
}

function buy(uint256 minAbtOut) external payable nonReentrant returns (uint256 abtIssued) {
    abtIssued = quoteAbtForAvax(msg.value);
    if (abtIssued < minAbtOut) revert SlippageExceeded(abtIssued, minAbtOut);
    saleToken.safeTransfer(msg.sender, abtIssued);
    emit TokensPurchased(msg.sender, msg.value, abtIssued);
}
```

这里 `router.getAmountsOut([WAVAX, ABT])` 的返回值就是 `buy()` 的 `abtIssued`，随后由业务合约实际转账给购买者；价格不是事件记录，也不是手动输入或硬编码常量。

### 4. 链上业务使用证明

- [业务合约部署交易](https://testnet.snowtrace.io/tx/0x1c810c6ba4ad1c02183f13794e53302977f508649b2ce15a7aa97650ee424e5e)，合约地址 `0xdBb76af289bE5aE0C893fc12bE6d3Eb23652F93D`。
- ABT 库存注入交易：[`0xf3b8bd4294a5ef12b329fba5f786e609ce720a900fab96e206a8f7a7b4dfb13a`](https://testnet.snowtrace.io/tx/0xf3b8bd4294a5ef12b329fba5f786e609ce720a900fab96e206a8f7a7b4dfb13a)。
- [按实时价格购买交易](https://testnet.snowtrace.io/tx/0xde5d11cea89e74ae2b3767078fa7b0e1ea7ddd7a6298714cebc47c9e5331d06f)，回执状态 `1`：支付 **0.001 AVAX**，触发 `TokensPurchased`，实际发放 **1955.016961782065611702 ABT**。

购买交易同时包含 ABT `Transfer` 事件：从业务合约转出 `1955.016961782065611702 ABT` 给购买者。当前同金额 Router 报价仍为 `1955.016961782065611702 ABT`，与业务发放量一致，证明 DEX 实时价格已参与业务计算。

### 5. 本地验证

`PangolinPricedAbtSale` 本地 Hardhat 测试通过（`1 passing`）：在测试中更改 Router 报价后，同一支付金额的实际 ABT 发放量随之改变。

## 已废弃的首次尝试（不作为证据）

## 实现概览

- 网络：Avalanche Fuji C-Chain（Chain ID `43113`）
- DEX：自建的 Uniswap V2 风格常数乘积 AMM（`Task3FujiDex`）
- 定价方式：由 Pair 的实时储备推导 Spot Price，并在 Swap 业务中按同一组储备计算成交数量；不接受用户手动传入价格。

## Token 与交易对

| 项目 | 名称 / 地址 |
| --- | --- |
| Token A | [Avalanche Bootcamp Token（ABT）](https://subnets-test.avax.network/c-chain/address/0xab8545161af1de74ef75f1cef4a7a1ceee596d49) |
| Token B | [Task 3 Quote Token（QUT）](https://subnets-test.avax.network/c-chain/address/0x8E1eb0C764bf5B6b02d602D9e142CE7747892F4a) |
| ABT/QUT Pair | [Task3FujiDex](https://subnets-test.avax.network/c-chain/address/0x43f3869860D66FABdE7E1D2798C326014f74A858) |
| 初始流动性 | 100,000 ABT / 100,000 QUT |

初始流动性已在 Fuji 链上确认：[LiquidityAdded 交易](https://subnets-test.avax.network/c-chain/tx/0xdb0185bbde94d4d164063829f90967e4cd90f777271e76f1b52c6066f592bf3b)。

## 核心代码：从 Pair 储备读取价格

```solidity
function spotPriceQutPerAbtX18() public view returns (uint256) {
    if (reserveAbt == 0 || reserveQut == 0) revert PoolNotInitialized();
    return (reserveQut * 1e18) / reserveAbt;
}

function quoteAbtOut(uint256 qutAmountIn) public view returns (uint256 amountAbtOut) {
    if (qutAmountIn == 0) revert ZeroAmount();
    if (reserveAbt == 0 || reserveQut == 0) revert PoolNotInitialized();

    uint256 qutAmountInWithFee = qutAmountIn * 997;
    amountAbtOut = (qutAmountInWithFee * reserveAbt) /
        (reserveQut * 1000 + qutAmountInWithFee);
}
```

`spotPriceQutPerAbtX18` 返回每 1 ABT 对应的 QUT 数量（18 decimals）。`quoteAbtOut` 使用池内实际储备和 0.3% AMM 手续费计算成交输出，因此会包含交易滑点，而不是使用写死价格。

## 核心代码：在业务逻辑中使用价格

```solidity
function swapQuoteForAbt(uint256 qutAmountIn, uint256 minAbtOut)
    external
    nonReentrant
    returns (uint256 amountAbtOut)
{
    amountAbtOut = quoteAbtOut(qutAmountIn);
    if (amountAbtOut == 0 || amountAbtOut >= reserveAbt) revert InsufficientLiquidity();
    if (amountAbtOut < minAbtOut) revert SlippageExceeded(amountAbtOut, minAbtOut);

    uint256 priceBeforeSwap = spotPriceQutPerAbtX18();
    qut.safeTransferFrom(msg.sender, address(this), qutAmountIn);
    abt.safeTransfer(msg.sender, amountAbtOut);

    reserveQut += qutAmountIn;
    reserveAbt -= amountAbtOut;
    emit SwapQuoteForAbt(msg.sender, qutAmountIn, amountAbtOut, priceBeforeSwap);
}
```

Swap 先调用 `quoteAbtOut`，再转入 QUT 并转出 ABT；事件同时记录交易前 Spot Price。该业务逻辑的输出完全由 DEX Pair 储备决定。

## 链上部署与使用证明

- Pair 合约：[0x43f3869860D66FABdE7E1D2798C326014f74A858](https://subnets-test.avax.network/c-chain/address/0x43f3869860D66FABdE7E1D2798C326014f74A858)
- Swap 交易：[0x039e14b1d967a264fb6f1383c38394074720485fa23f137a8a371b13a0648a73](https://subnets-test.avax.network/c-chain/tx/0x039e14b1d967a264fb6f1383c38394074720485fa23f137a8a371b13a0648a73)
- 交易回执状态：成功（`0x1`）
- `SwapQuoteForAbt` 事件：输入 `100 QUT`，输出 `99.600698103990321649 ABT`，交易前 Spot Price 为 `1 QUT / ABT`。
- Swap 后 Pair 储备：`99,900.399301896009678351 ABT / 100,100 QUT`；最新 Spot Price 为 `1.001997996999999999 QUT / ABT`。

这次储备与 Spot Price 的变化证明：合约既从真实 Fuji Pair 读取价格，又在 Swap 中使用该 Pair 的储备完成定价和兑换。

## 实现说明

我使用 ABT 作为 Token A，并部署 QUT 作为 Token B。先将两种 Token 各 100,000 单位添加到 Pair，随后授权并用 100 QUT 执行 Swap。合约使用整数最小单位与 `uint256` 计算，避免浮点精度问题；Swap 提供 `minAbtOut` 作为滑点下限，且通过 `SafeERC20` 和 `ReentrancyGuard` 处理转账与重入风险。

该 Pair 是课程用的最小 AMM。生产环境不应仅依赖瞬时储备 Spot Price，重要清算或高价值业务还应加入 TWAP、独立预言机、价格有效期和更严格的滑点限制。

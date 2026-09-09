# Task 3：使用 DEX Oracle 获取代币价格

> 对应课程：第三章 Solidity 合约实战  
> 学员：EzraSheep  
> 钱包地址：`0x064F885e29223118E339cD5f6313CD8288B28454`  
> 目标网络：Avalanche Fuji C-Chain (`43113`)  

---

## 一、使用的 DEX 与核心地址

本项目选用了 Avalanche Fuji 测试网上最经典的 Uniswap V2 架构去中心化交易所 —— **Pangolin Exchange**。

| 项目 | 内容 / 合约地址 | 区块浏览器链接 |
| :--- | :--- | :--- |
| **DEX 名称** | **Pangolin Exchange (Uniswap V2)** | [Pangolin App](https://app.pangolin.exchange/) |
| **测试网络** | Avalanche Fuji C-Chain (`43113`) | [Snowtrace](https://testnet.snowtrace.io/) |
| **Factory V2** | `0xE4A575550C2b460d2307b82dCd7aFe84AD1484dd` | [Snowtrace Factory](https://testnet.snowtrace.io/address/0xE4A575550C2b460d2307b82dCd7aFe84AD1484dd) |
| **Router V2** | `0x2D99ABD9008Dc933ff5c0CD271B88309593aB921` | [Snowtrace Router](https://testnet.snowtrace.io/address/0x2D99ABD9008Dc933ff5c0CD271B88309593aB921) |
| **Token B (WAVAX)** | `0xd00ae08403B9bbb9124bB305C09058E32C39A48c` | [Snowtrace WAVAX](https://testnet.snowtrace.io/address/0xd00ae08403B9bbb9124bB305C09058E32C39A48c) |

---

## 二、Token A 与 Token B 信息

| Token | 代币名称 (Symbol) | 合约地址 | Decimals | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| **Token A** | Fake USD (`USDF`) | `0xe9d75eee6b658d3f205cef60f45f733fd58d3e36` | 18 | 自主实现并部署的 FakeUSD Oracle 业务代币合约 |
| **Token B** | Wrapped AVAX (`WAVAX`) | `0xd00ae08403B9bbb9124bB305C09058E32C39A48c` | 18 | 测试网原生包装资产 |

> **精度说明**：USDF 与 WAVAX 均为标准的 18 位精度（`decimals = 18`），两者在价格计算与兑换过程中量级一致，避免了跨精度运算造成的舍入误差。

---

## 三、交易对与流动性注入

| 项目 | 内容 |
| :--- | :--- |
| **交易对** | `USDF / WAVAX Pair` |
| **交易对合约地址** | `0xFe0ca5E4B76eb07218baeD073053d96528047fF2` |
| **合约部署 Tx** | `0xe11753e89d4e3fee47a8e0cc1832e46c8c7dc912cb91496f55c07037d7d8d6c4` |
| **USDF 授权 Tx** | `0x075c1b7e49129c6c04760070c63216ba6049973e5284740a7b44208386ffbf6a` |
| **添加流动性 Tx** | `0x038dbe024c7d3be3ac8f1591cc93c67714e7732cd3545adf36f47614ebe53509` |
| **区块高度** | Block `58282468` |
| **初始流动性注入量** | **20,000 USDF + 0.05 AVAX** |
| **隐含初始池汇率** | 1 AVAX = 400,000 USDF（即 1 USDF ≈ 0.0000025 AVAX） |
| **Snowtrace 交易对链接** | [Snowtrace Pair 0xFe0c...7fF2](https://testnet.snowtrace.io/address/0xFe0ca5E4B76eb07218baeD073053d96528047fF2) |

---

## 四、流动性添加、价格获取与真实业务购买验证截图

运行自动化配置与验证脚本，脚本自动完成 USDF 授权、向 Pangolin 添加流动性生成全新交易对 `0xFe0ca5E4B76eb07218baeD073053d96528047fF2`、链上读取储备量、实时获取 DEX 预言机报价，并成功调用 `buyTokensWithAVAX()` 购买 1813.22 USDF 代币：

![流动性添加、DEX价格获取与真实购买验证](./task3-EzraSheep-verify.png)

---

## 五、获取 Swap / Oracle 价格的核心代码

合约价格获取采用**完全链上动态查询机制**，杜绝任何硬编码的常量价格：

### 方式一：Router Quoter — 获取含 AMM 手续费与滑点的真实 Swap 报价

```solidity
/**
 * @notice 核心 Oracle 函数 1：通过 DEX Router 动态获取指定数量 AVAX 能换取的代币数量
 * @param avaxAmount 传入的 AVAX 数量 (wei)
 * @return tokenAmount 经 DEX 恒定乘积公式计算出的代币数量 (包含池深度与手续费计算)
 */
function getTokenPriceFromRouter(uint256 avaxAmount) public view returns (uint256 tokenAmount) {
    require(dexRouter != address(0), "DEX router not set");
    require(avaxAmount > 0, "AVAX amount must be > 0");

    address wavax = IPangolinRouter(dexRouter).WAVAX();
    address[] memory path = new address[](2);
    path[0] = wavax;
    path[1] = getEffectivePriceToken();

    // 沿 [WAVAX -> USDF] 路径调用 DEX Router 的 getAmountsOut 获得链上实时报价
    uint256[] memory amounts = IPangolinRouter(dexRouter).getAmountsOut(avaxAmount, path);
    return amounts[1];
}

/**
 * @notice 核心 Oracle 函数 2：通过 DEX Router 动态获取指定代币数量对应的 AVAX 数量
 * @param tokenAmount 代币数量 (USDF)
 * @return avaxAmount 对应的 AVAX 数量 (wei)
 */
function getAvaxPriceForToken(uint256 tokenAmount) public view returns (uint256 avaxAmount) {
    require(dexRouter != address(0), "DEX router not set");
    require(tokenAmount > 0, "Token amount must be > 0");

    address wavax = IPangolinRouter(dexRouter).WAVAX();
    address[] memory path = new address[](2);
    path[0] = getEffectivePriceToken();
    path[1] = wavax;

    // 沿 [USDF -> WAVAX] 路径获取反向兑换报价
    uint256[] memory amounts = IPangolinRouter(dexRouter).getAmountsOut(tokenAmount, path);
    return amounts[1];
}
```

### 方式二：Pair Oracle — 读取底层交易对储备量 (Reserves)

```solidity
/**
 * @notice 从 DEX Pair 直接读取底层流动性储备量
 * @return reserveToken 目标代币储备量
 * @return reserveWAVAX WAVAX 储备量
 */
function getReserves() public view returns (uint256 reserveToken, uint256 reserveWAVAX) {
    address pair = getPairAddress();
    require(pair != address(0), "DEX Pair does not exist");
    (uint112 r0, uint112 r1, ) = IPangolinPair(pair).getReserves();
    address token0 = IPangolinPair(pair).token0();
    if (token0 == getEffectivePriceToken()) {
        reserveToken = uint256(r0);
        reserveWAVAX = uint256(r1);
    } else {
        reserveToken = uint256(r1);
        reserveWAVAX = uint256(r0);
    }
}
```

---

## 六、使用价格的合约实际业务逻辑核心代码

合约将获取到的 DEX 动态价格直接应用在**代币购买（铸造）**与**代币出售（赎回）**两大真实业务场景中：

```solidity
/**
 * @notice 核心业务逻辑 1：使用 AVAX 按照 DEX 实时 Oracle 价格购买/铸造代币
 * 业务实现：用户支付 AVAX，合约调用 DEX Oracle 获取实时汇率计算出应得代币数量并铸造给调用者
 */
function buyTokensWithAVAX() public payable returns (uint256 tokensBought) {
    require(msg.value > 0, "Must send AVAX to buy tokens");

    // 🌟 核心：基于 DEX Oracle 动态计算代币购买数量（完全由 DEX 实时汇率决定，非写死固定价格）
    tokensBought = getTokenPriceFromRouter(msg.value);
    require(tokensBought > 0, "Insufficient output from DEX");

    // 为购买者铸造对应数量的代币
    _mint(msg.sender, tokensBought);

    emit TokensPurchased(msg.sender, msg.value, tokensBought);
}

/**
 * @notice 核心业务逻辑 2：用户按 DEX 实时 Oracle 价格出售代币，换回合约金库中的 AVAX
 */
function sellTokensForAVAX(uint256 tokenAmount) external returns (uint256 avaxRefund) {
    require(tokenAmount > 0, "Token amount must be > 0");
    require(balanceOf(msg.sender) >= tokenAmount, "Insufficient token balance");

    // 🌟 核心：基于 DEX Oracle 动态计算应返还的 AVAX 数量
    avaxRefund = getAvaxPriceForToken(tokenAmount);
    require(address(this).balance >= avaxRefund, "Contract has insufficient AVAX balance");

    // 销毁用户的代币并将 AVAX 发送给用户
    _burn(msg.sender, tokenAmount);
    (bool success, ) = msg.sender.call{value: avaxRefund}("");
    require(success, "AVAX transfer failed");

    emit TokensSold(msg.sender, tokenAmount, avaxRefund);
}
```

### 关键设计点分析
1. **完全动态无硬编码**：`buyTokensWithAVAX()` 与 `sellTokensForAVAX()` 均不接受固定价格参数，全部实时调用 `getTokenPriceFromRouter(msg.value)` 与 `getAvaxPriceForToken(tokenAmount)` 从 DEX 实时计算。
2. **链上可溯源留痕**：购买与出售分别触发 `TokensPurchased` 与 `TokensSold` 事件，记录用户、支付 AVAX、成交代币数量，形成可追溯的链上证据。
3. **价格自适应**：流动性池内的储备量随着交易动态变动，后续代币铸造和回购价格自动按照 AMM 恒定乘积公式自适应浮动。

---

## 七、部署后的合约地址列表与区块浏览器链接

| 项目 | 地址 / 哈希 | 区块浏览器 (Snowtrace Fuji) |
| :--- | :--- | :--- |
| **FakeUSD (USDF)** | `0xe9d75eee6b658d3f205cef60f45f733fd58d3e36` | [Snowtrace FakeUSD](https://testnet.snowtrace.io/address/0xe9d75eee6b658d3f205cef60f45f733fd58d3e36) |
| **USDF / WAVAX Pair** | `0xFe0ca5E4B76eb07218baeD073053d96528047fF2` | [Snowtrace Pair](https://testnet.snowtrace.io/address/0xFe0ca5E4B76eb07218baeD073053d96528047fF2) |
| **Deployer 账户** | `0x064F885e29223118E339cD5f6313CD8288B28454` | [Snowtrace Deployer](https://testnet.snowtrace.io/address/0x064F885e29223118E339cD5f6313CD8288B28454) |
| **Pangolin Router** | `0x2D99ABD9008Dc933ff5c0CD271B88309593aB921` | [Snowtrace Router](https://testnet.snowtrace.io/address/0x2D99ABD9008Dc933ff5c0CD271B88309593aB921) |
| **Pangolin Factory** | `0xE4A575550C2b460d2307b82dCd7aFe84AD1484dd` | [Snowtrace Factory](https://testnet.snowtrace.io/address/0xE4A575550C2b460d2307b82dCd7aFe84AD1484dd) |
| **WAVAX 合约** | `0xd00ae08403B9bbb9124bB305C09058E32C39A48c` | [Snowtrace WAVAX](https://testnet.snowtrace.io/address/0xd00ae08403B9bbb9124bB305C09058E32C39A48c) |
| **合约部署 Tx** | `0xe11753e89d4e3fee47a8e0cc1832e46c8c7dc912cb91496f55c07037d7d8d6c4` | [Snowtrace Deploy Tx](https://testnet.snowtrace.io/tx/0xe11753e89d4e3fee47a8e0cc1832e46c8c7dc912cb91496f55c07037d7d8d6c4) |
| **USDF 授权 Tx** | `0x075c1b7e49129c6c04760070c63216ba6049973e5284740a7b44208386ffbf6a` | [Snowtrace Approve Tx](https://testnet.snowtrace.io/tx/0x075c1b7e49129c6c04760070c63216ba6049973e5284740a7b44208386ffbf6a) |
| **添加流动性 Tx** | `0x038dbe024c7d3be3ac8f1591cc93c67714e7732cd3545adf36f47614ebe53509` | [Snowtrace Liquidity Tx](https://testnet.snowtrace.io/tx/0x038dbe024c7d3be3ac8f1591cc93c67714e7732cd3545adf36f47614ebe53509) |
| **buy 业务购买 Tx** | `0x4f76bc1c6410ff2e9a7f22db6ececeebf9aa97657da2165fcfc7034d65761ce6` | [Snowtrace Buy Tx](https://testnet.snowtrace.io/tx/0x4f76bc1c6410ff2e9a7f22db6ececeebf9aa97657da2165fcfc7034d65761ce6) |

---

## 八、成功读取或使用价格的链上数据验证

### 1. 流动性池储备量查询验证
添加流动性后，底层交易对储备量为：
- **USDF Reserve**: `20,000.0 USDF`
- **WAVAX Reserve**: `0.05 AVAX`

### 2. DEX Oracle 实时询价验证
- **AVAX 购买 USDF 报价**：
  `getTokenPriceFromRouter(0.01 AVAX)` => **`3324.995831248957812239 USDF`**
  > **AMM 公式验证**：根据 Uniswap V2 扣除 0.3% 手续费的恒定乘积公式：  
  > $\Delta y = \frac{y \times \Delta x \times 0.997}{x + \Delta x \times 0.997} = \frac{20000 \times 0.01 \times 0.997}{0.05 + 0.01 \times 0.997} = \frac{199.4}{0.05997} \approx 3324.9958$ USDF。  
  > 链上合约返回值与数学计算 **100% 精确吻合**，证实价格完全来自于 DEX 流动性池！

- **USDF 换回 AVAX 报价**：
  `getAvaxPriceForToken(1,000 USDF)` => **`0.002374148687907796 AVAX`**

### 3. 合约核心业务 `buyTokensWithAVAX()` 链上执行验证
- **调用交易**：`0x4f76bc1c6410ff2e9a7f22db6ececeebf9aa97657da2165fcfc7034d65761ce6`（Block `58282471`）
- **支付金额**：`0.005 AVAX`
- **实际获得代币数量**：**`1813.221787760298263162 USDF`**
  > $\Delta y = \frac{20000 \times 0.005 \times 0.997}{0.05 + 0.005 \times 0.997} = \frac{99.7}{0.054985} \approx 1813.221787...$ USDF。  
  > 证明合约成功在链上调用 DEX Oracle，动态计算出买家应得代币数量并精准铸造到买家账户中！

---

## 九、实现过程简要说明

1. **选择 DEX 并实现接口**：在 Avalanche Fuji 上选择主流 Uniswap V2 架构的 Pangolin DEX，编写包含 Factory、Pair 与 Router 的接口文件 `IPangolin.sol`。
2. **专属合约与可配置代币**：实现独立的 `FakeUSD.sol` 合约，代币名称与符号支持动态配置，默认为 Fake USD (`USDF`)；部署者为 `EzraSheep` (`0x064F885e29223118E339cD5f6313CD8288B28454`)。集成 `dexRouter` 配置，开发 `getTokenPriceFromRouter` 与 `getAvaxPriceForToken` 预言机函数，重构 `buyTokensWithAVAX` 与 `sellTokensForAVAX`，实现 100% 依赖 DEX 动态报价的代币购买与回购业务。
3. **本地 Mock 与单元测试**：编写 `MockPangolinRouter.sol` 模拟各种 DEX 汇率与流动性深度，本地 6 项单元测试全部通过。
4. **测试网部署与流动性注入**：使用 Hardhat 将 `FakeUSD` 合约部署至 Avalanche Fuji 测试网（`0xe9d75eee6b658d3f205cef60f45f733fd58d3e36`）；编写自动化部署运维脚本 `setupDexPairAndVerify.ts`，自动向 Pangolin Router 授权并注入 20,000 USDF + 0.05 AVAX 流动性，成功生成交易对 `0xFe0ca5E4B76eb07218baeD073053d96528047fF2`。
5. **端到端链上验证**：调用合约的预言机函数实时询价，并成功发送 0.005 AVAX 调用 `buyTokensWithAVAX()` 购买并铸造了 1813.22 USDF，获取了完整的链上交易哈希与终端执行记录留存。

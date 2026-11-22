---
title: 热钱包的 Gas 记账
date: 2026-08-16 20:05:00
tags:
  - Gas
  - go-ethereum
  - 对账
categories:
  - Web3
---

# 热钱包的 Gas 记账

链上链下对账里，gas 是最容易对不上的一项。金额不大，但笔数多，每笔差一点，月底汇总就是一笔说不清的钱。

这篇把 gas 记账涉及几个问题拆开讲：预估和实际的关系、记到谁头上、重发交易怎么处理、L2 的差异。

## 预估和实际不是一回事

用 go-ethereum 发交易，预估走 `EstimateGas`：

```go
gasLimit, err := client.EstimateGas(ctx, ethereum.CallMsg{
    From:  hotWallet,
    To:    &to,
    Value: value,
    Data:  data,
})
```

预估成本是 `gasLimit × gasPrice`，用来在提现创建时展示给用户。但实际扣费来自回执，EIP-1559 之后是：

```text
实际费用 = gasUsed × (baseFee + priorityFee)
```

`gasLimit` 是上限，`gasUsed` 是实际消耗。ERC20 转账通常六万多，但合约钱包、多签、或目标合约有特殊逻辑时会更高。两者天然不相等，账上要分开记：

| 字段 | 含义 |
|---|---|
| `gas_estimated` | 创建时预估，给用户看的 |
| `gas_actual` | 回执确认后从链上读，记平台成本 |

回执里的关键字段是 `EffectiveGasPrice`：

```go
receipt, err := client.TransactionReceipt(ctx, txHash)
actualFee := new(big.Int).Mul(
    new(big.Int).SetUint64(receipt.GasUsed),
    receipt.EffectiveGasPrice,
)
```

不要自己拿交易里的 `GasPrice` 或 `MaxFeePerGas` 去算——baseFee 由区块决定，只有回执里的 `EffectiveGasPrice` 是真实成交价。

## 记到哪个科目

平台有两类 gas 支出，性质不同，要分开记：

- **用户提现**：gas 由平台承担，用户按提现金额扣余额。gas 计入平台成本，不摊给用户。
- **链上归集**：把热地址的零散资产归集到冷钱包，gas 同样是平台成本，但属于运营支出。

流水里加 `gas_category` 区分：

```sql
INSERT INTO gas_ledger (tx_hash, chain_id, category, gas_used,
                        effective_gas_price, amount_wei, biz_no)
VALUES (?, ?, 'WITHDRAW'/'SWEEP', ?, ?, ?, ?)
```

`biz_no` 关联到提现单或归集任务。这样单笔提现的真实成本能反查，也能按链汇总出平台每天烧了多少 gas。

## 重发交易只按回执记账

交易 Pending 超时后走 RBF 提价，用同一个 nonce 重新广播。链上最终只有一笔成功交易，被替换掉的那笔不会上链，也就不产生 gas。

如果按「广播过就记账」，就会多记一笔。所以规则很简单：**只在拿到回执时记账**。广播阶段不记，被替换的交易没有回执，自然不会进账。

失败交易有回执，也消耗 gas，要记，但单独标记：

```go
if receipt.Status == types.ReceiptStatusFailed {
    // revert 消耗 gas，记平台损失，不关联到用户提现的成功金额
    recordGas(ctx, receipt, "FAILED_TX", orderNo)
}
```

## 对账时的差异来源

把 gas 流水和链上实际支出对起来，差异通常出在这几个地方：

1. **回执没拿到**。RPC 返回 not found，记账跳过。补救办法是交易超过一定时间没记账就重新拉回执，并设重试上限与告警。
2. **内部交易**。合约调用触发的内部转账会消耗 gas，但不出现在 Transfer 日志里。只看日志会漏，要看回执的 `GasUsed`。
3. **L2 的 L1 data fee**。Arbitrum、Optimism 上除了 L2 执行费还有 L1 数据费，回执结构不一样，要单独处理，不能套 L1 的公式。
4. **gas 退款**。某些情况下合约退 gas，实际扣费小于 `gasUsed × effectiveGasPrice`，以回执为准，不要二次推算。

## 一个简单的校验

每天按链汇总 gas 流水，和地址的实际余额变化对一次：

```text
Σ gas_ledger.amount_wei  ≈  地址余额减少额 - Σ 转出金额
```

两边有细微差异正常（还有其他类型交易），但差异突然放大就能发现问题。这个校验能覆盖大部分「回执漏记」「节点返回旧数据」的情况。

gas 单笔小、笔数多，把它当成独立账务科目来记，成本分摊和差异排查都会省事很多。

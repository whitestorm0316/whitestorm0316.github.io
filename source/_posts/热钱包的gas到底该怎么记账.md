---
title: 热钱包的 gas 到底该怎么记账
date: 2026-08-16 20:05:00
tags:
  - Gas
  - go-ethereum
  - 对账
categories:
  - Web3
---

# 热钱包的 gas 到底该怎么记账

链上链下对账里最难对上的一项是 gas。不是因为算不出来，是因为它同时涉及三个环节：发交易前预估、发交易后实际扣费、以及这笔钱该记到谁的头上。

## 预估和实际是两回事

用 go-ethereum 发交易，预估走 `EstimateGas`：

```go
gasLimit, err := client.EstimateGas(ctx, ethereum.CallMsg{
    From:  hotWallet,
    To:    &to,
    Value: value,
    Data:  data,
})
```

然后乘以 gas price 得到预估成本。这个值在提现创建时展现给用户，作为手续费。问题是实际扣费不是这么算的。EIP-1559 之后：

```text
实际费用 = gasUsed × (baseFee + priorityFee)
```

`gasLimit` 是上限，`gasUsed` 是实际消耗。ERC20 转账通常 65k 左右，但遇到合约钱包、多签、或者目标合约有特殊逻辑，会不一样。我们上线初期按固定 gasLimit=100000 估算，结果有些交易实际只用了 5 万，多预估的部分没有退还——因为交易本身没有多扣，是我们账上多记了。

后来改成两笔记录：

| 字段 | 含义 |
|---|---|
| `gas_estimated` | 提现创建时预估，给用户看的 |
| `gas_actual` | 回执确认后从 receipt 读，记进平台成本 |

```go
receipt, err := client.TransactionReceipt(ctx, txHash)
actualFee := new(big.Int).Mul(
    new(big.Int).SetUint64(receipt.GasUsed),
    receipt.EffectiveGasPrice,
)
```

`EffectiveGasPrice` 是回执里的字段，不要自己用交易里的 `GasPrice` 或 `MaxFeePerGas` 去算——baseFee 是区块决定的，只有回执里的是真实成交价。

## 记到谁的头上

平台有两个角色：用户提现、平台归集。两者都产生 gas，但性质不同。

- 用户提现：**平台承担 gas**。用户拿到的是提现金额，扣的是余额里的那部分，gas 单独计入平台成本科目。
- 归集交易：把各热地址的零散资产归集到冷钱包，gas 也是平台成本。

这两类要分开记，否则对账时会把归集交易的 gas 摊到用户提现上，金额对不上还找不到原因。我们的流水里加了一个 `gas_category` 字段：

```sql
INSERT INTO gas_ledger (tx_hash, chain_id, category, gas_used,
                        effective_gas_price, amount_wei, biz_no)
VALUES (?, ?, 'WITHDRAW'/'SWEEP', ?, ?, ?, ?)
```

`biz_no` 关联到提现单号或归集任务号。这样单笔提现的真实成本可以反查，也能按天汇总出平台在这条链上烧了多少钱。

## RBF 之后会多出一笔 gas

失败重发的场景更麻烦。交易 Pending 超时后走 RBF 提价，用同一个 nonce 重新广播。链上最终只有一笔成功交易，但被替换掉的那笔交易在 mempool 里已经消失，**这笔 gas 不会扣**。

如果按照「广播过就记账」的逻辑，就会多记一笔。我们最早就是这么干的，月底一算，gas 成本比链上实际高了一截。

正确做法：**只按回执记账**。广播时不记，只有拿到 `TransactionReceipt` 且状态为成功时才写 gas 流水。被替换的交易没有回执，自然不会进账。

但这里有个边界情况：交易失败（revert）也有回执，也消耗 gas。这类 gas 要记，但要单独标记：

```go
if receipt.Status == types.ReceiptStatusFailed {
    // revert 交易消耗 gas，记平台损失，不关联到用户提现的成功金额
    recordGas(ctx, receipt, "FAILED_TX", orderNo)
}
```

## 对账差异的常见来源

把链上 gas 和实际账对起来，剩下的差异通常来自这几个地方：

1. **回执没拿到**：RPC 节点不稳定，`TransactionReceipt` 返回 not found，记账任务跳过。后来加了个补救——交易超过 1 小时没记账就重新拉回执，节点返回 not found 超过 3 次就告警。
2. **内部交易**：合约调用触发的内部转账也会消耗 gas，但不在 `logs` 里。只解析 Transfer 事件会漏，必须看回执的 `GasUsed`。
3. **L2 的 L1 data fee**：在 Arbitrum、Optimism 上，除了 L2 执行费还有 L1 数据费，回执结构不一样。跨链支持要单独处理这部分，不能把 L2 的公式套上去。
4. **退款**：某些情况下合约会退 gas，实际扣费比 `gasUsed × effectiveGasPrice` 少。以回执字段为准，不要二次推算。

## 一个很朴素的校验

每天跑一个任务，把当天所有 gas 流水按链汇总，和从节点拉到的地址交易总支出比一次：

```text
Σ gas_ledger.amount_wei  ≈  地址余额变化 - Σ 转出金额
```

等式两边有细微差异是正常的（还有其他类型的交易），但差异突然放大就能发现问题。这个校验帮我们抓到过一次 RPC 节点返回了旧回执、导致 gas 少记的情况。

gas 这东西单笔金额小，架不住量大。一个月几万笔提现，每笔差一点点，到月底就是一笔说不清的钱。早点把它当成独立账务科目来记，后面省心很多。

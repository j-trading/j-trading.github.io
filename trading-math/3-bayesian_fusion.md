# 从直觉到量化——贝叶斯信号融合，如何融合多个量化指标?

在量化交易中，最让人纠结的不是“没有信号”，而是“信号太多了，不知道该听谁的”。

各类交易软件中，都默认可以看各种技术指标：均线、RSI、布林通道、MACD，同时也可以通过AI生成各种情绪指数，这些指标直接要如何融合成最终的分数？

一个直观的方法是“少数服从多数”——三个看多就做多。但这里有个致命的问题，**不同行情下，不同信号的“话语权”是不一样的**，如：
* 在震荡行情里，做均线回归策略，可以大赚；但是使用趋势跟踪指标，那只能反复割肉；
* 在趋势行情里，与之相反。
(所以一般量化会有一步，判断`market-regime`, 即当前处于震荡行情、还是趋势行情，这个后面会提到)

**贝叶斯信号融合**——就是为了解决这个问题而生的。它的核心思想是：**让每个信号带着自己的“历史信用”来投票**，也就是记录不同信号在历史中的表现，然后再融合。

## 1. 盘感挑战：三个量化信号，如何取舍？

假设当前是**震荡行情**，你收到了三个信号：
> 震荡行情/趋势行情如何判断 ? 可以通过ADX指标判断得到，不保证对，但是只要能比50%高就有意义。

- **均线金叉**：看多。历史上它在趋势行情里准确率 70%，但现在是震荡市，它的震荡市准确率只有 45%。
- **RSI 超卖反弹**：看多。震荡市准确率 65%。
- **布林带下轨**：看多。震荡市准确率 55%。

你的大脑在做什么？你可能在“感觉”：“RSI 在震荡市比较靠谱，均线金叉在震荡市不太行，但三比零看多，应该还是做多吧？”

这种感觉其实是一种**盘感层面的贝叶斯更新**。你给了 RSI 更大的权重，给均线金叉打了个折扣。但你的大脑没办法精确计算出：综合来看，做多的概率到底是多少？85% 还是 55%？这决定了你是该重仓还是轻仓试探。

贝叶斯融合，就是把这种盘感变成精确的数字。这也是这个专栏的目标，摒弃直觉，拥抱数学。

## 2. 前置知识
### 2.1 贝叶斯定理
中学都学过的[贝叶斯定理]( https://zh.wikipedia.org/wiki/%E8%B4%9D%E5%8F%B6%E6%96%AF%E5%AE%9A%E7%90%86)：
$$P(A|B) = \frac{P(B|A)\cdot P(A)}{P(B)}$$

### 2.2 条件独立/无条件独立
1. **无条件独立**: 即不同事件A和B发生的概率，*不依赖任何条件*相互独立
        $$P(A, B) = P(A) \cdot P(B)$$

> 举例说明：
> 从1..N中，随机取一个数K (N>>k); 
> * P(A) 表示K能被2整除的概率，可知$P(A) =\frac{1}{2}$
> * P(B) 表示K能被3整除的概率，可知$P(B) = \frac{1}{3}$
> 
> 由于$P(A, B) = P(A) \cdot P(B) = \frac{1}{6} $, 故A和B是**无条件独立**.
> 即A和B整除的数字互质，就是无条件独立。

2. **条件独立**: 只有在C这个事件发生后，事件A和事件B才是独立的。
$$P(A, B|C) = P(A|C) \cdot P(B|C)$$

> 举例说明：
> 从1..N中，随机取一个数K (N>>k); 
> * P(A) 表示K能被4整除的概率，可知$P(A) =\frac{1}{4}$
> * P(B) 表示K能被6整除的概率，可知$P(B) = \frac{1}{6}$
> * P(C) 表示K能被2整除的概率，可知$P(C) = \frac{1}{2}$
> 
> 由于gcd(4,6)=2, 进一步得到: $P(A, B|C) = P(A|C) \cdot P(B|C) = \frac{1}{6} $ 
> 这个就是事件A和B是在事件C发生后独立，即**条件独立**。

3. 有n个信号，假设各信号在**给定市场方向下**条件独立, 则有:

$$P(s_1, s_2, ..., s_n \mid \text{up}) = \prod_{i=1}^{n} P(s_i \mid \text{up})$$

> 为什么要做这个假设?
> (1) 首先这个假设经常会不成立，上涨时，各个信号经常是联动的, 但是没关系。
> (2) 做这个假设，可以**把复杂的联合分布($2^n$个)降低到$n$个分布**， 这样我们只需要记录每个信号单独的上涨概率即可，记录n次。
> (3) 有了这个假设, 就可以结合贝叶斯定理, 计算终极目标$P(\text{up} \mid s_1, s_2, ..., s_n )$了！

## 3. 贝叶斯信号融合

上述公式扩展到量化交易，即在得到$n$个交易信号后，判断上涨概率：
$$P(up \mid s_1, s_2, ..., s_n) =\frac{1}{1 + \frac{1-p}{p} \cdot \prod \frac{P(s_i \mid \text{down})}{P(s_i \mid \text{up})}} $$
> 公式推导:$$
    \begin{aligned}
    (求上涨概率)&\quad P(\text{up} \mid s_1,...,s_n) \\
    (贝叶斯公式)&= \frac{P(s_1,...,s_n \mid \text{up})P(\text{up})}{P(s_1,...,s_n)} \\
    （全概率公式）&= \frac{P(s_1,...,s_n \mid \text{up})P(\text{up})}{P(s_1,...,s_n \mid \text{up})P(\text{up}) + P(s_1,...,s_n \mid \text{down})P(\text{down})} \\
    (令p=P(up)+条件独立)&=\frac{\left[\prod P(s_i \mid \text{up})\right] p}{\left[\prod P(s_i \mid \text{up})\right] p + \left[\prod P(s_i \mid \text{down})\right] (1-p)} \\
    &= \frac{1}{1 + \frac{1-p}{p} \cdot \prod \frac{P(s_i \mid \text{down})}{P(s_i \mid \text{up})}} \\
    \end{aligned}
    $$

公式理解:
- $p=P(up)$ : 上涨概率, 初始可以默认为50%, 或者根据历史k线数据赋予一个值;
- $P(s_i \mid \text{down})$和$P(s_i \mid \text{up})$: 下跌/上涨时，出现信号 $s_i$的概率，同样基于历史数据生成;

这样基于贝叶斯的signal融合就很直观了: 
* 记录上涨概率$p$、在不同行情下(上涨、下跌), 不同信号出现的概率。
* 另外, 可以拆分不同股票/币单独处理，拆分不同market-regime也单独处理。

## 4. python实现

代码如下：
```python

class BayesianSignalFuser:
    """Fuses multiple signals via Bayesian updating.

    P(up | s1, s2, ..., sn) ∝ P(s1|up) * P(s2|up) * ... * P(sn|up) * P(up)

    P(si|up) comes from the conditional win rate table.
    """

    def __init__(self, win_rate_provider, default_prior: float = 0.5) -> None:
        self._win_rate_provider = win_rate_provider
        self._default_prior = default_prior

    def fuse(self, signals: list[Signal], prior: float | None = None) -> FusedSignal:
        if prior is None:
            prior = self._default_prior

        now = datetime.now(timezone.utc)

        # Filter out neutral signals
        actionable = [s for s in signals if s.direction != Direction.NEUTRAL]
        if not actionable:
            return FusedSignal(
                direction=Direction.NEUTRAL,
                probability=prior,
                contributing_signals=(),
                market_regime=MarketRegime.RANGING,
                timestamp=now,
            )

        # Use the market regime from the last signal (all should agree for same bar)
        regime = actionable[-1].market_regime

        # Bayesian update: P(up|signals) ∝ product of P(si|up) * P(up)
        # P(si|up) for a "long" signal = conditional win rate for that signal type in this regime
        # P(si|down) for a "long" signal = 1 - conditional win rate
        # For a "short" signal, we flip: P(si|up) = 1 - win_rate, P(si|down) = win_rate

        log_likelihood_up = 0.0
        log_likelihood_down = 0.0

        for s in actionable:
            win_rate, total, wins = self._win_rate_provider.get_win_rate(s.signal_type, regime)
            _logger.debug("Fuse Win rate: %.2f%% (%d/%d)", win_rate * 100, wins, total)
            # Clamp to avoid log(0)
            p_up = max(1e-6, min(1 - 1e-6, win_rate))
            p_down = 1 - p_up

            if s.direction == Direction.LONG:
                log_likelihood_up += np.log(p_up)
                log_likelihood_down += np.log(p_down)
            else:  # SHORT
                log_likelihood_up += np.log(p_down)
                log_likelihood_down += np.log(p_up)

        # Posterior
        p_prior = max(1e-6, min(1 - 1e-6, prior))
        log_posterior_up = log_likelihood_up + np.log(p_prior)
        log_posterior_down = log_likelihood_down + np.log(1 - p_prior)

        # Normalize using log-sum-exp trick
        max_log = max(log_posterior_up, log_posterior_down)
        posterior_up = np.exp(log_posterior_up - max_log)
        posterior_down = np.exp(log_posterior_down - max_log)
        probability = float(posterior_up / (posterior_up + posterior_down))

        direction = Direction.LONG if probability > 0.5 else Direction.SHORT

        return FusedSignal(
            direction=direction,
            probability=probability,
            contributing_signals=tuple(actionable),
            market_regime=regime,
            timestamp=now,
        )
```


## 4. 实盘局限

市场难以预测，所以任何模型都会有缺陷，当前策略局限如下：（除非你有足够的资金、底牌可以操控市场）

**条件独立性的假设。** 贝叶斯融合假设各信号在给定行情方向后是条件独立的：$P(s_1, s_2 \mid up) = P(s_1 \mid up) \times P(s_2 \mid up)$。但现实中，均线金叉和 MACD 金叉高度相关——它们本质上是同一个信息的不同表达。如果你把 5 个高度相关的趋势指标一起扔进融合器，它会**严重高估**后验概率，给你一个虚假的 95% 置信度。解决方案：要么用互信息做信号筛选，要么引入协方差矩阵做多元高斯近似。

**胜率表是历史的后视镜。** 条件胜率来自历史回测，但市场状态会“漂移”。2018 年的震荡市和 2024 年的震荡市可能完全不是一回事。我的经验是：胜率表必须做**滚动窗口更新**（比如只看最近 500 笔交易），并且人为压低极端值（>90% 的胜率直接 clip 到 90%）。

**结语**

交易的本质是在不确定中做决策。贝叶斯融合不是帮你“消灭不确定性”，而是**精确量化不确定性**——当它给出 52% 的概率时，你知道应该轻仓试探；当它给出 85% 时，你才敢放心加码。


> 关于信号融合：如果你只有两三个高度相关的指标（比如均线+MACD+布林带），贝叶斯融合的提升有限，因为冗余信号只会放大同一个偏见。真正的威力在于跨体系信号融合——趋势指标 + 波动率指标 + 链上数据 + 资金流，这时候贝叶斯更新的“独立证据累积”效应才能真正发挥。

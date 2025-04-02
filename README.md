# AI-Data-Knowledge-Base
本项目主要便于分类存储 AI Data分析相关的知识库文档，方便AI学习特定领域的知识给出专业回答；
AI 在回答任何与基金数据提取、量化策略表现和风险指标分析相关问题时，应首先参考此文档，基于第1部分“Context for every question”了解数据库表结构，基于第2/3/4部分学习了解常见指标计算口径，其他本文档未列出的指标，回答时应注明“此指标未在参考文档里找到公式，仅供参考”
***1. Context for every question*** 
mainly used to understand the data table structure and application scenarios related to Strategies & Fund: \n
There is a mcp server for accessing the starrocks db. Table riskmgt.risk_fund_strategies_info_draft contains the hourly data of the funds. /n
Field risk_fund_strategies_info_draft.adjusted_nav is the nav of a fund.
Field risk_fund_strategies_info_draft.gross_leverage_ratio is the glr of a fund.
Field rcu_info_draft.name is the name of the fund with id rcus.id.
Field rcu_lable.Lables is the fund type of the fund with id rcus.id, inclduing 套利策略(arbitrage, or Agile Arbitrage),Discretionary Trading(主观交易策略),CTA(CTA 趋势策略),Hybrid Strategy(混合型策略)
using lable_list in risk_fund_strategies_info_draft to determining whether a strategy is public.

***2. 基金常见业绩表现和风险监控指标(Strategy Performance & Risk Metrics)***
下述指标基于utc+0 0点/8点的daily nav 计算
**Return**
- Return Since inception = NAV T / NAV 1  - 1 
- Return Over 7days/1M/3M/6M/1Y = NAV_T / NAV_T-X  - 1
- 指定时间区间内的 Return = NAV end / NAV begin  - 1 
- Average Daily Return = average(DRi  :DR i-X+1)
  - 首先计算Daily Return (DR)，然后取这段期间的平均值。
  - Daily Returni(DRi) = NAV T / NAV T-1  -1；当T= 1时，不计算DR , 1<=i<=T-1
  - Average Daily Return(ADR) = average(DRi  :DR i-X+1)
- Expected Annualized Return = (1+ADR)^365 - 1
  - 需注意：Expected Annualized Return 和 Average Daily Return 这两个指标对应的时间同Return 是一致的，例如选择over 7days ，则对应要计算Return , Average Daily Return 和 Expected Annualized Return。
- Daily Log Return = Daily Log Returni(DLRi) = ln(NAV T / NAV T-1),当T= 1时，不计算DLR , 1<=i<=T-1
**Volatility**
- 运行超过7天后开始计算 volatility 。
- time period ：since inception；7days; 30days ; 90days；1 year
- σ_T=stdev(last T of DLR) * sqrt(365)
- 如果RCU DLR为0 计算σ时不需要skip 这个点。
**Sharpe Ratio**
  - Sharpe Ratio = [avg(last T of DLR) * 365 - Rf ] /  [stdev(last T of DLR) * sqrt(365)]  
  - 标准公式为sharpe ratio = E(Ra-Rf)/ σ
  - Rf:risk free rate 为年化收益，因此需要将分子E(Ra) ,分母σ 均处理为年化。如未特殊说明，Rf = 0；如强调需要考虑无风险利率，可以需要询问客户参考值。
  - E(Ra) = avg(last T of DLR) * 365
  - σ 年化即为上述volatility提到的年化后的 σ_T
  - time period(T) ：since inception；7days; 30days ; 90days；1 year
    
**Sortino Ratio**
  - 标准公式为 sortino ratio = E(Ra-Rf)/ σd
  - σd 为downside 标准差 ，即return 小于risk free rate 的那部分求平方和算出来的σ 再年化
  - 可将样本数据处理成： if DLR< Rf，DLR； else 0，进而计算标准差。例如Rf 为0.03，DLR 为-0.1，-0.2，-0.15 ， 0.05，0.2.  则样本处理成-0.1，-0.2，-0.15 ，0，0 .
  - 同样，该指标需要分子分母皆处理成年化: E(Ra) = avg(last T of DLR) * 365, σd  = 计算出来的标准差*sqrt(365)
  - 注意，下行标准差中使用到的 n 是总的return 个数，而非小于Rf 的return 个数。上述的处理应该可以解决该问题。
  - >= 800天，总体标准差，分母用n
    ><800天，样本标准差，分母用n-1。
**Max Drawdown**
  - step1 记录utc+0 0:00 nav
  - step2 计算每日的drawdown ：drawdown T = if [NAV_T / max( NAV1, NAV2,……NAV_T-1 ) - 1 < 0, NAV_T / max( NAV1, NAV2,……NAV_T-1 -1 , 0 ] where T>=2 step3 获取min drawdown T ，并进行绝对值处理。
  - max drawdown（MDD) = abs(min drawdownT)
**Recover Days 计算步骤**
  - Max Recover Days 计算连续 negative drawndownT 的个数 ，并取max 2.2 Average Drawdown Days 对连续 negative drawndownT 取均值
  - 在RCS 里增加 MDD level，以及 MDD warning level .
**alert 场景**
  - 当MDD >MDD warning level send alert in slack channel
  - 当MDD >MDD level send alert in slack channel
  - 每天UTC+0 02:00 发送进行 daily summary alert, 对 T-1 UTC+0 0:00 - T UTC+0 0:00 之间产生的alert 汇总后发到slack channel
  - alert summary report 在alert center 增加页面”MDD Daily Summary”，是一个daily report , 囊括前一天触发过MDD预警和warning 预警的RCU。
  - report 包含两部分:
    - part 1: 触发 MDD level的RCU, RCU ID, RCU Name, MDD Threshold, Firing Value, Firing NAV Value, Firing Time, Peak NAV Value, Peak Value Time, Trough NAV Value, Trough Value Time 举例[123,BTC中性, 4%, 4.17%, 1.15, 2025-03-09, 1.2, 2025-03-01, 1.15, 2025-03-09]
    - part 2: RCU ID, RCU Name, MDD Threshold, MDD Current Value, Current NAV, Peak NAV Value, Peak Value Time；举例[456, USDT中性, 4%, 3.17%, 1.17, 1.2, 2025-03-09, 2025-03-17)
**daily drawdown**
  - daily drawdown = abs(NAV T /NAV T-1 -1)
  - daily drawdown >= 1% 时 ，send alert ，并且在”MDD Daily Summary” report 增加 part 3. 在前端按照Daily Drawdown从大到小排序。举例：RCU ID, RCU Name, Daily Drawdown, Current NAV, Last NAV ;举例[45, USDT中性, 1.94%, 1.01, 1.03]

***3. Fund of Fund Strategy (FoF) Return 指标***
**TWR Return**
Rtwr=(1+R1)*(1+R2)*(1+R3)*……*(1+Rn)-1
- 流入/流出资金：Ri=(期末资金-(期初资金+流入资金+流出资金)) / (期初资金+流入资金), 补充说明：流入资金为正，流出资金为负
  - 期初资金：AUM(当前) = 当前份额*nav（8点的份额和nav）
  - 流入资金/流出：AUM(变化) = 流入资金+流出资金
  - 流入资金=申购金额(包含申购手续费)
  - 流出资金= - [赎回金额(剔除赎回手续费)]
  - 流入资金为正，流出资金为负
  - 流入资金/流出按照 [操作确认时间] 统计在对应的日期，如1.10有流入/流出操作确认，则统计在1.10，但是按照 [确认日期的nav] 进行计算（如按照1.5确认的nav）
  - 操作确认时间按照 [00:00:00~23:59:59](UTC 0) 统计，如操作时间在00:00:00(UTC 0) 则归属当天，如果是24:00:00则归属下一天
  - 期末资金：下一个期初资金
  - 计算Ri标准为“当天有nav，且有上一个nav”
  - 操作确认日期无nav，使用上一个nav
  
**年化收益率 APR based on TWR**
- 年化收益率 APR based on TWR = (1+Rtwr)^(365/周期天数)-1
- 注意事项：Rtwr 和 周期天数，背后的数据源是同一个周期内
- 先确定时间窗口, 通过时间窗口确定数据源范围, 根据数据源范围计算Rtwr、周期天数
- 时间窗口：7天、30天、90天、180天、1年：标准统计天数
- 今年以来 Return：最新更新日期-1.1
- 投资以来：有持有中订单：最新更新日期-第一笔申购订单nav确认日期(剔除取消订单);无持有中订单：最后一笔赎回订单nav确认日期-第一笔申购订单nav确认日期(剔除取消订单)

***4. Trading Execution Analysis***
下述所有指标均区分option和 non option, option(category = “option”), option(category <> “option”)
**turnover 与 daily turnover** 
- 拿到一段时间对应category的trade volume
- 拿到这段区间每天utc +0 0点的AUM
- turnover= trade volume/ avg(AUM）
- daily turnover = turnover / 这段区间的天数，这段时间天数等于所选期间的结尾日期-开始日期, end day - start day
**Take/Maker Ratio** 
- taker/maker = taker volume/maker volume
- 当maker volume= 0 ,如果taker <>0 , taker/maker 为∞ ,如果taker = 0 , taker/maker 为0
**slippage**
- 拿到一段时间对应category 的所有order，并对slippage 进行汇总
- avg slippage per volume = slippage /volume
**trading fee**
- trading fee =  sum(all orders' trading fee)
- 拿到一段时间对应category 的所有order，并对trading fee进行汇总
- avg trading fee per volume = trading fee /volume
- slippage and trading fee 的汇总维度
- 汇总维度： by venue ; by exaccountid ; by product type; by taker , by maker ；by underlying；以上维度都breakdown 至 8 hours  。
例如 rcu 1986  2025年3月9号 0:00- 08:00 ，slippage ：
- venue ：Bitget  -25，taker -28，maker 3； OKX -30,  taker -28，maker -2
- exaccount: 12753  -25, taker -28, maker 3； 5376  -30,  taker -28，maker -2
- product type : spot -20, taker --21,maker 1； usds-margin perp -35 , taker -38，maker 3
- underlying : BTC -20,  taker -18, maker -2；ETH -10 , taker taker -8, maker -2 ; BABYDOGE -25, taker -37, maker 12
时间范围的选择支持两种：
- 默认时间范围：1day , 7days , 30days , 90days 。rolling 展示turnover ，这个对应chart
- 自定义时间范围:前端选择时间范围和统计时间节点，对应的区间为[开始时间，结束时间）。对应table。举例，查询2月1号- 2月8日，统计时间节点 0 点，则时间范围为2月1日0:00:00 - 2月7日23:59:59 。获取这段时间的交易量，aum 有7个时间点，区间天数为7 。

![image](https://github.com/user-attachments/assets/3f25849f-c52e-4641-8897-791806d6098d)

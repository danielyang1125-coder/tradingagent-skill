---
name: tradingagents-cn-skill
version: 2.0.0
description: >
  分析股票并生成 JSON 分析报告。当用户提到以下任何一种意图时触发：
  分析股票、股票分析、股票报告、分析一下、
  看看这只股票、帮我分析、技术分析、基本面分析、买卖建议、
  风险评估、多空分析、投资建议。
  支持输入：股票截图、股票代码、股票名称、行情文字描述。
  输出：多智能体深度分析 JSON 报告（技术面+基本面+新闻+多空辩论+风控评估+五级评级）。
metadata:
  openclaw:
    emoji: "📊"
    requires:
      bins: ["python3"]
---

# TradingAgents-CN Skill

多智能体股票分析框架。Agent **在主进程中串行**完成 12 步分析，参照 TradingAgents 论文架构：
三位分析师（各自独立调用）→ 多空辩论（2轮）→ 研究管理者裁决 → 交易员 → 风控三方辩论 → 投资组合经理最终决策 → JSON 报告。

## 全局规则

### ⚠️ 禁止 SubAgent（最高优先级）

**所有步骤必须在主 Agent 进程中串行执行，禁止使用 subagent、子任务、并行 agent 或任何形式的任务分派。**

原因：本 Skill 的每一步都依赖前面步骤的完整输出（stock_data → 分析师报告 → 辩论 → 决策）。SubAgent 不共享上下文，会导致数据丢失（如新闻为空、技术指标缺失）。

**具体要求：**
- 不要使用 `dispatching-parallel-agents` 或类似机制
- 不要把任何步骤委托给其他 agent 或 worker
- 所有 LLM 调用、搜索工具调用、脚本执行都在同一个 agent 会话中完成

### 执行模式

**全程自动执行，不询问用户。** 遇到错误自动重试（最多 3 次），某步失败使用默认值继续，只在最终输出 JSON 和摘要。

### 验证策略

**每步 LLM 调用后必须 validate。** 验证通过后将 JSON 保存在 Agent 上下文变量中，不写磁盘。

**每步 validate 命令格式：**
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step <步骤名> --stock-code <code>
```

辩论 Round 2 需要加 `--round 2`：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step bull_debate --stock-code <code> --round 2
```

验证失败（exit 1）→ 将 stderr 中的 hint 追加到 prompt 重试，最多 3 次。全部失败 → 用默认值：
```bash
python3 {baseDir}/scripts/validate_step.py --step <步骤名> --default
```

### 语言要求

所有 LLM 调用的 system_prompt 和 user_message 使用**中文**。所有分析内容使用**中文**输出。

---

## 工作流程

```
Step 1: 获取原始文本 + 结构化提取 → stock_data JSON
Step 2: 搜索工具获取新闻 → news_data
───── 阶段一：三位分析师报告（3次独立LLM调用）─────
Step 3: 技术/市场分析师 → tech_analyst
Step 4: 基本面分析师 → fundamentals_analyst
Step 5: 新闻分析师 → news_analyst
───── 阶段二：多空辩论（2轮）─────
Step 6: 看多研究员 Round 1 + 看空研究员 Round 1
Step 7: 看多研究员 Round 2 + 看空研究员 Round 2
───── 阶段三：研究管理者裁决 + 交易员 ─────
Step 8: 研究管理者裁决 LLM → manager_decision
Step 9: 交易员计划 LLM → trading_plan
───── 阶段四：风控三方辩论 ─────
Step 10: 激进 + 保守 + 中立三方风控辩论
───── 阶段五：最终决策 ─────
Step 11: 投资组合经理（最终决策）LLM → final_decision（验证 rating）
───── 阶段六：报告生成 ─────
Step 12: 组装并输出 JSON 报告
```

---

## Step 1: 获取原始文本 + 结构化提取

根据用户输入类型：

- **截图/图片**：识别图片中的所有可见数据（股票代码、价格、涨跌幅、均线、MACD、KDJ 等技术指标数值），优先提取图表上标注的具体数字
- **文字描述**：直接使用
- **股票代码/名称**：后续通过搜索工具补充数据

**LLM 调用：结构化数据提取**
- system_prompt:
  ```
  你是股票数据提取专家。从用户提供的文本（可能来自截图识别、交易软件、财报等）中，
  提取结构化的股票数据。只提取文本中明确存在的信息，缺失的字段设为 null。
  不要虚构或推测任何数据。以纯 JSON 格式返回。

  特别注意：如果文本中包含技术指标的具体数值（如 MA5:98.32、DIF:2.45、K:75.3），
  必须精确提取这些数值，不要丢弃。
  ```
- user_message:
  ```
  请从以下文本中提取股票数据，以纯 JSON 格式返回：

  {原始文本}

  要求返回的 JSON 格式：
  {
    "stock_code": "股票代码（如 PDD、600519、00700.HK）",
    "stock_name": "股票名称",
    "current_price": 数字或null,
    "change_pct": "涨跌幅字符串或null",
    "volume": "成交量或null",
    "turnover": "成交额或null",
    "technical_indicators": {
      "MA5": 数字或null, "MA10": 数字或null, "MA20": 数字或null,
      "MA30": 数字或null, "MA60": 数字或null, "MA120": 数字或null,
      "RSI": 数字或null,
      "MACD": "MACD综合描述或null",
      "MACD_detail": {"DIF": 数字或null, "DEA": 数字或null, "MACD_histogram": 数字或null},
      "KDJ": "KDJ综合描述或null",
      "KDJ_detail": {"K": 数字或null, "D": 数字或null, "J": 数字或null},
      "BOLL_upper": 数字或null, "BOLL_mid": 数字或null, "BOLL_lower": 数字或null,
      "volume_trend": "放量/缩量/平量 或null"
    },
    "fundamentals": {
      "PE": 数字或null, "PB": 数字或null, "ROE": "百分比或null",
      "market_cap": "市值或null", "revenue": "营收或null", "net_profit": "净利润或null"
    },
    "k_line_pattern": "K线形态描述或null",
    "chart_observation": "图表整体观察（均线排列、MACD柱体变化、量价配合等定性描述）或null",
    "other_info": "其他信息或null"
  }
  ```

**A股补充：用 Python 直接获取实时行情和 K 线数据**

当用户输入的是 A 股代码/名称时，优先通过以下 API 获取真实数据，再填入 stock_data：

```python
import urllib.request, json

# 1. 新浪财经实时行情（GBK 编码）
# 深市(000/002/003/300)用 sz 前缀，沪市(600/601/603/688)用 sh 前缀
url = f"https://hq.sinajs.cn/list=sz{stock_code}"
req = urllib.request.Request(url, headers={"Referer": "https://finance.sina.com.cn/", "User-Agent": "Mozilla/5.0"})
with urllib.request.urlopen(req, timeout=15) as r:
    line = r.read().decode("gbk")
# 解析：fields = line.split('"')[1].split(',')
# fields[3]=最高 fields[4]=最低 fields[5]=现价 fields[6]=涨跌额 fields[7]=成交量 fields[8]=成交额

# 2. 新浪财经日K线+均线（GBK 编码，含 MA5/10/20/30/60）
url = f"https://money.finance.sina.com.cn/quotes_service/api/json_v2.php/CN_MarketData.getKLineData?symbol=sz{stock_code}&scale=240&ma=5,10,20,30,60&datalen=60"
req = urllib.request.Request(url, headers={"Referer": "https://finance.sina.com.cn/", "User-Agent": "Mozilla/5.0"})
with urllib.request.urlopen(req, timeout=15) as r:
    klines = json.loads(r.read().decode("gbk"))
# 每条格式：{"day":"2026-04-21","open":"1.91","high":"1.92","low":"1.86","close":"1.87",
#            "volume":"210949622","ma_price5":1.888,"ma_price10":1.888,"ma_price20":1.894,"ma_price30":1.981}
```

从 K 线数据中计算 MACD/KDJ（纯 Python，无需第三方库）：
```python
def ema(prices, n):
    k = 2/(n+1); e = [prices[0]]
    for p in prices[1:]: e.append(p*k + e[-1]*(1-k))
    return e

closes = [float(k["close"]) for k in klines]
ema12 = ema(closes, 12); ema26 = ema(closes, 26)
dif = [ema12[i]-ema26[i] for i in range(len(closes))]
dea = ema(dif, 9)
macd = [(dif[i]-dea[i])*2 for i in range(len(closes))]

def kdj(data, n=9):
    k, d = 50, 50; ks, ds, js = [], [], []
    for i in range(len(data)):
        low_n = min(float(x["low"]) for x in data[max(0,i-n+1):i+1])
        high_n = max(float(x["high"]) for x in data[max(0,i-n+1):i+1])
        c = float(data[i]["close"])
        rsv = (c-low_n)/(high_n-low_n)*100 if high_n!=low_n else 50
        k = k*2/3+rsv/3; d = d*2/3+k/3; j = 3*k-2*d
        ks.append(k); ds.append(d); js.append(j)
    return ks, ds, js
```

---

## Step 2: 获取新闻数据

使用系统可用的搜索工具（如 web_search 或已安装的搜索类 MCP 工具）搜索该股票的最新新闻，**至少搜索 3 个查询**：
1. `{股票名称} {股票代码} 最新消息 2026`
2. `{股票代码} stock news latest`
3. `{股票名称} 研报 评级`

从搜索结果中提取每条新闻的**标题和摘要（snippet）**，不需要逐条 web_fetch。

将搜索结果整理为 JSON 数组，**保存为变量 `news_data`**：
```json
{
  "news_list": [
    {"title": "新闻标题", "date": "日期", "source": "来源", "summary": "至少50字的摘要", "sentiment": "偏多/中性/偏空", "url": "原文链接"},
    ...
  ]
}
```

**关键约束**：
- `news_list` 数组**至少包含 3 条新闻**，不足则追加搜索
- `summary` 字段不得为空，最少 50 字
- 如果实在搜不到，设置 `news_list` 为空数组但 `sentiment` 设为 "暂无数据"

**A股补充：web_search 对中文新闻覆盖差时，改用东方财富公告 API**

```python
import urllib.request, json

url = f"https://np-anotice-stock.eastmoney.com/api/security/ann?sr=-1&page_size=10&page_index=1&ann_type=A&client_source=web&stock_list={stock_code}"
req = urllib.request.Request(url, headers={"Referer": "https://quote.eastmoney.com/", "User-Agent": "Mozilla/5.0"})
with urllib.request.urlopen(req, timeout=15) as r:
    d = json.loads(r.read())
    items = d.get("data", {}).get("list", [])
    # 每条：{"notice_date": "2026-04-21 ...", "title": "公司名:关于..."}
```

将公告标题整理为 `news_list` 条目，`summary` 字段根据标题内容手动推断（50字以上），`sentiment` 根据公告性质判断（司法拍卖/诉讼→偏空，收购/重组→偏多，常规公告→中性）。

---

## Step 3: 技术/市场分析师

**LLM 调用：**
- system_prompt: 读取 `references/tech_prompt.md`
- user_message:
  ```
  请对以下股票进行技术/市场分析：

  股票数据：
  {stock_data JSON（Step 1 结果）}

  请从技术面角度深度分析，以纯 JSON 格式返回。
  ```

**验证并保存**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step tech --stock-code {股票代码}
```

---

## Step 4: 基本面分析师

**LLM 调用：**
- system_prompt: 读取 `references/fundamentals_prompt.md`
- user_message:
  ```
  请对以下股票进行基本面分析：

  股票数据：
  {stock_data JSON（Step 1 结果）}

  请从基本面角度深度分析，以纯 JSON 格式返回。
  ```

**验证并保存**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step fundamentals --stock-code {股票代码}
```

---

## Step 5: 新闻分析师

**LLM 调用：**
- system_prompt: 读取 `references/news_prompt.md`
- user_message:
  ```
  请对以下股票的近期新闻进行分析：

  股票数据：
  {stock_data JSON（Step 1 结果）}

  近期新闻：
  {news_data JSON（Step 2 结果）}

  请从新闻面角度深度分析，以纯 JSON 格式返回。
  ```

**验证并保存**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step news --stock-code {股票代码}
```

**如果某个分析师输出异常**（空、非 JSON、缺字段），validate 会 exit 1 并给 hint，重试或使用默认值：
```bash
python3 {baseDir}/scripts/validate_step.py --step tech --default
python3 {baseDir}/scripts/validate_step.py --step fundamentals --default
python3 {baseDir}/scripts/validate_step.py --step news --default
```

---

## Step 6: 多空辩论 Round 1

**一次消息内连续完成两个 LLM 调用：**

### 6A: 看多研究员 Round 1

- system_prompt: 读取 `references/bull_prompt.md`
- user_message:
  ```
  请基于以下三份分析师报告，作为看多研究员发表你的第一轮论证：

  技术/市场分析报告：
  {tech_analyst 的 report 字段}

  基本面分析报告：
  {fundamentals_analyst 的 report 字段}

  新闻分析报告：
  {news_analyst 的 report 字段}

  这是第一轮辩论，尚无看空方论点。请基于以上分析报告构建你的看多论证。
  ```

**验证并保存 bull_r1**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step bull_debate --stock-code {股票代码} --round 1
```

### 6B: 看空研究员 Round 1

- system_prompt: 读取 `references/bear_prompt.md`
- user_message:
  ```
  请基于以下三份分析师报告，作为看空研究员发表你的第一轮论证：

  技术/市场分析报告：
  {tech_analyst 的 report 字段}

  基本面分析报告：
  {fundamentals_analyst 的 report 字段}

  新闻分析报告：
  {news_analyst 的 report 字段}

  看多研究员的论点：
  {bull_r1 的 debate_text}

  这是第一轮辩论。请基于分析报告构建你的看空论证，并回应看多方的观点。
  ```

**验证并保存 bear_r1**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step bear_debate --stock-code {股票代码} --round 1
```

---

## Step 7: 多空辩论 Round 2

### 7A: 看多研究员 Round 2

- system_prompt: 读取 `references/bull_prompt.md`
- user_message:
  ```
  这是第二轮辩论。请回应看空研究员的论点：

  辩论历史：
  看多 Round 1：{bull_r1 的 debate_text}
  看空 Round 1：{bear_r1 的 debate_text}

  请针对看空方的每个具体论点进行回应和反驳。
  ```

**验证并保存 bull_r2**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step bull_debate --stock-code {股票代码} --round 2
```

### 7B: 看空研究员 Round 2

- system_prompt: 读取 `references/bear_prompt.md`
- user_message:
  ```
  这是第二轮辩论。请回应看多研究员的论点：

  辩论历史：
  看多 Round 1：{bull_r1 的 debate_text}
  看空 Round 1：{bear_r1 的 debate_text}
  看多 Round 2：{bull_r2 的 debate_text}

  请针对看多方的每个具体论点进行回应和反驳。
  ```

**验证并保存 bear_r2**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step bear_debate --stock-code {股票代码} --round 2
```

---

## Step 8: 研究管理者裁决

**LLM 调用：**
- system_prompt: 读取 `references/manager_prompt.md`
- user_message:
  ```
  请评估以下多空辩论，做出你的裁决：

  完整辩论记录：
  === 看多研究员 Round 1 ===
  {bull_r1 的 debate_text}

  === 看空研究员 Round 1 ===
  {bear_r1 的 debate_text}

  === 看多研究员 Round 2 ===
  {bull_r2 的 debate_text}

  === 看空研究员 Round 2 ===
  {bear_r2 的 debate_text}

  请做出明确的决策（买入/卖出/持有），并制定详细的投资计划。以纯 JSON 格式返回。
  ```

**验证并保存**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step manager --stock-code {股票代码}
```

---

## Step 9: 交易员计划

**LLM 调用：**
- system_prompt: 读取 `references/trader_prompt.md`
- user_message:
  ```
  基于分析师团队的综合分析，以下是为 {股票名称}（当前价格：{current_price}）制定的投资计划：

  研究管理者的投资计划：
  {manager_decision JSON}

  请基于此计划，制定具体的交易方案。以纯 JSON 格式返回。
  所有价格必须是具体数字。
  ```

**验证并保存**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step trader --stock-code {股票代码}
```

---

## Step 10: 风控三方辩论

**连续完成三个 LLM 调用：**

### 10A: 激进型风控分析师

- system_prompt: 读取 `references/risk_aggressive_prompt.md`
- user_message:
  ```
  交易员的交易计划：
  {trading_plan JSON}

  参考数据：
  技术分析摘要：{tech_analyst 的 key_points}
  基本面摘要：{fundamentals_analyst 的 key_points}
  新闻摘要：{news_analyst 的 key_points}

  请从激进型风险分析师的角度评估此交易计划。
  ```

**验证并保存**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step risk_aggressive --stock-code {股票代码}
```

### 10B: 保守型风控分析师

- system_prompt: 读取 `references/risk_conservative_prompt.md`
- user_message:
  ```
  交易员的交易计划：
  {trading_plan JSON}

  参考数据（同上摘要）。

  激进型分析师的观点：
  {risk_aggressive 的 debate_text}

  请从保守型风险分析师的角度评估此交易计划，并回应激进派的观点。
  ```

**验证并保存**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step risk_conservative --stock-code {股票代码}
```

### 10C: 中立型风控分析师

- system_prompt: 读取 `references/risk_neutral_prompt.md`
- user_message:
  ```
  交易员的交易计划：
  {trading_plan JSON}

  参考数据（同上摘要）。

  激进型分析师的观点：
  {risk_aggressive 的 debate_text}

  保守型分析师的观点：
  {risk_conservative 的 debate_text}

  请从中立型风险分析师的角度评估此交易计划，平衡激进派和保守派的观点。
  ```

**验证并保存**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step risk_neutral --stock-code {股票代码}
```

---

## Step 11: 投资组合经理（最终决策）

**LLM 调用：**
- system_prompt: 读取 `references/risk_manager_prompt.md`
- user_message:
  ```
  请综合以下信息，做出最终投资决策：

  标的：{股票代码} {股票名称}，当前价格：{current_price}

  研究管理者的投资计划：
  {manager_decision JSON}

  交易员的交易提案：
  {trading_plan JSON}

  === 风控分析师辩论记录 ===

  激进型分析师：
  {risk_aggressive 的 debate_text}

  保守型分析师：
  {risk_conservative 的 debate_text}

  中立型分析师：
  {risk_neutral 的 debate_text}

  请使用五级评级（买入/增持/持有/减持/卖出），以纯 JSON 格式返回最终决策。
  ```

**此步必须验证**：
```bash
echo '<LLM输出>' | python3 {baseDir}/scripts/validate_step.py --step portfolio_manager --stock-code {股票代码} --attempt 1
```

- `exit 0` → 通过，继续
- `exit 1` → 将 hint 追加到 prompt 重新调用 LLM，最多重试 **3 次**
- 超过 3 次 → 使用默认值：
  ```bash
  python3 {baseDir}/scripts/validate_step.py --step portfolio_manager --default --stock-code {股票代码}
  ```

---

## Step 12: 输出 JSON 报告

将所有步骤的上下文变量组装成完整 JSON，直接输出到对话中：

```json
{
  "stock_code": "{股票代码}",
  "stock_name": "{股票名称}",
  "current_price": {当前价格},
  "news_data": {news_data},
  "parallel_analysis": {
    "tech_analyst": {tech_analyst},
    "fundamentals_analyst": {fundamentals_analyst},
    "news_analyst": {news_analyst}
  },
  "investment_debate": {
    "bull_r1": {bull_r1},
    "bear_r1": {bear_r1},
    "bull_r2": {bull_r2},
    "bear_r2": {bear_r2}
  },
  "manager_decision": {manager_decision},
  "trading_plan": {trading_plan},
  "risk_debate": {
    "aggressive": {risk_aggressive},
    "conservative": {risk_conservative},
    "neutral": {risk_neutral}
  },
  "final_decision": {final_decision}
}
```

同时附上简要的分析摘要：
- 评级（五级：买入/增持/持有/减持/卖出）
- 关键价格：买入价、目标价、止损价
- 风险等级和投资期限
- 一句话投资论文

---

## 调试方法

### 单步验证工具
```bash
# 测试某个 LLM 输出是否通过验证
echo '{"rating":"买入","executive_summary":"摘要","investment_thesis":"论文","risk_level":"中"}' | python3 {baseDir}/scripts/validate_step.py --step portfolio_manager

# 获取某步骤的默认值
python3 {baseDir}/scripts/validate_step.py --step tech --default
```

### 日志查看
```bash
cat {baseDir}/scripts/logs/{股票代码}_*.log
```

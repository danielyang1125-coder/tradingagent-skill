# A股数据获取实战手册

> 本文档提炼自多次实战数据采集的成败经验，供其他技能（行业轮动、财务分析、量化筛选等）快速参考。
> 最后更新：2026-05-10，AKShare 1.18.60，Tushare free token

---

## 一、启动检查清单（30秒内完成）

每次分析启动时，按顺序执行：

```bash
# 1. 找 Python（Windows 下不要用 WindowsApps 存根）
dir /s /b "%LOCALAPPDATA%\Programs\Python\Python3*\python.exe"

# 2. 验证 AKShare
python -c "import akshare; print(akshare.__version__)"

# 3. 验证 Tushare（如有token）
python -c "import tushare; print('OK')"
```

---

## 二、数据源分层策略

```
L1: AKShare — 免费，无密钥，主力数据源
L2: Tushare — 免费注册，补AKShare的3个缺口
L3: 东方财富裸API — 仅当L1/L2均失败时使用
```

**分工原则**：能用AKShare的用AKShare，只有以下3个场景切换到Tushare：
- CPI/PPI近期数据（需积分≥2000）
- 北向资金净买卖数据
- 个股申万行业分类

---

## 三、已验证函数速查

### 3.1 ✅ 稳定可靠（直接用）

| 数据需求 | 函数 | 备注 |
|---------|------|------|
| 上证/科创/创业板日线 | `ak.stock_zh_index_daily(symbol='sh000001')` | date是datetime.date，过滤前 `.apply(lambda v: v.strftime("%Y-%m-%d"))` |
| SW申万行业日线 | `ak.index_hist_sw(symbol='801080')` | 列序: [代码,日期,收盘,开盘,最高,最低,成交量,成交额] |
| LPR | `ak.macro_china_lpr()` | 最新值在iloc[-1]，不是iloc[0] |
| PMI制造业 | `ak.index_pmi_man_cx()` | 最新值在iloc[-1] |
| PMI非制造业 | `ak.index_pmi_ser_cx()` | 最新值在iloc[-1] |
| M2/货币供应 | `ak.macro_china_money_supply()` | ⚠️ 最新值在iloc[0]，不是在iloc[-1] |
| GDP | `ak.macro_china_gdp()` | ⚠️ 最新值在iloc[0]，不是在iloc[-1] |

### 3.2 ⚠️ 需Tushare补缺

| 数据需求 | AKShare | 为什么不行 | Tushare替代 |
|---------|---------|----------|------------|
| CPI | `macro_china_cpi_yearly()` | 2025-09起`今值`列全NaN | `pro.cn_cpi()`（需积分≥2000） |
| PPI | `macro_china_ppi_yearly()` | 同上 | `pro.cn_ppi()`（需积分≥2000） |
| 北向资金 | `stock_hsgt_hist_em()` | `当日成交净买额`列全NaN | `pro.moneyflow_hsgt()` |

### 3.3 ❌ 不可用

| 函数 | 原因 |
|------|------|
| `macro_china_pmi()` | 返回2008年旧数据，用 `index_pmi_man_cx()` 替代 |
| `sw_index_daily()` | 此函数不存在，用 `index_hist_sw()` 替代 |
| `stock_board_industry_spot_em()` | 依赖东方财富push2 API，网络可能不通 |

---

## 四、三大陷阱与解决方案

### 陷阱1：AKShare 各函数行排序不一致

**这是最容易出错的地方。** 不同函数返回的DataFrame，最新数据行的位置不同：

| 函数 | 最新值位置 | 获取方式 |
|------|---------|---------|
| `macro_china_money_supply()` | 第0行 | `df.iloc[0]` |
| `macro_china_gdp()` | 第0行 | `df.iloc[0]` |
| `macro_china_lpr()` | 最后一行 | `df.iloc[-1]` |
| `index_pmi_man_cx()` | 最后一行 | `df.iloc[-1]` |
| `index_pmi_ser_cx()` | 最后一行 | `df.iloc[-1]` |
| `macro_china_cpi_yearly()` | 最后一行 | `df.iloc[-1]` |
| `macro_china_ppi_yearly()` | 最后一行 | `df.iloc[-1]` |
| `stock_zh_index_daily()` | 最后一行 | `df.iloc[-1]` |
| `index_hist_sw()` | 最后一行 | `df.iloc[-1]` |

> **规则**：历史时间序列函数（日线/PMI/LPR）→ iloc[-1]；汇总统计函数（M2/GDP）→ iloc[0]。
> **保底**：永远用 `df.tail(3)` 验证，不凭记忆假设。

### 陷阱2：SW行业指数列结构与普通指数不同

```python
# ❌ 错误（把SW指数当普通指数处理）
date_col = df.columns[0]  # SW的col[0]是"代码"，不是日期！
close_val = row.iloc[4]   # SW的col[4]是"最高"，不是"收盘"！

# ✅ 正确
date_str = row.iloc[1]    # col[1] = 日期
close_val = row.iloc[2]   # col[2] = 收盘
```

SW指数列序：`[代码, 日期, 收盘, 开盘, 最高, 最低, 成交量, 成交额]`

### 陷阱3：Tushare北向资金列名含义

```python
# ❌ 错误
ggt_ss  # 这是"港股通(沪)"，南向资金，不是北向！
ggt_sz  # 这是"港股通(深)"，南向资金，不是北向！

# ✅ 正确
hgt          # 沪股通（北向净买入），单位：万元
sgt          # 深股通（北向净买入），单位：万元
north_money  # 北向合计 = hgt + sgt，单位：万元
south_money  # 南向合计 = ggt_ss + ggt_sz，单位：万元

# 转换为亿元
net_buy_yi = round(north_money_wan / 10000, 2)
```

---

## 五、数据验证规则

每个数据点获取后必须验证，不通过则标记异常：

| 数据 | 合理范围 | 验证方式 |
|------|---------|---------|
| PMI | 45–55 | `assert 45 < val < 55` |
| CPI | -2% ~ +6% | 同比不应超过±10% |
| GDP | 2%–8% | 中国近十年在3%–8%区间 |
| 指数收益率（月） | -30% ~ +30% | 单月超过±20%标记为异常 |
| 北向净买入（日） | -200亿 ~ +300亿 | 单日超过±500亿标记异常 |
| M2同比 | 5%–15% | 中国M2很少低于5%或高于20% |
| SW行业收益率（月） | -40% ~ +40% | 单行业月收益超过±30%需复核 |

---

## 六、北向资金完整示例（Tushare）

```python
import tushare as ts
ts.set_token("your_token")
pro = ts.pro_api()

nb = pro.moneyflow_hsgt(start_date='20260401', end_date='20260510')
nb = nb.sort_values('trade_date')

# 方法1: 直接用 north_money（推荐）
total_yi = nb['north_money'].astype(float).sum() / 10000

# 方法2: 分市场
hgt_yi = nb['hgt'].astype(float).sum() / 10000     # 沪股通
sgt_yi = nb['sgt'].astype(float).sum() / 10000     # 深股通

print(f"北向净买入: {total_yi:.1f}亿 (沪:{hgt_yi:.1f} 深:{sgt_yi:.1f})")
```

---

## 七、Windows Shell 编码规范

```bash
# 标准执行模板
chcp 65001 >nul && python script.py

# 输出到文件（避免终端乱码干扰read_file）
python script.py > _log.txt 2>&1

# Python脚本内部
import sys
sys.stdout.reconfigure(encoding='utf-8')
with open('output.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, ensure_ascii=False)
```

---

## 八、速率限制

| 数据源 | 限制 | 建议间隔 |
|--------|------|---------|
| AKShare 指数数据 | 底层东方财富频率限制 | 批量时 >= 0.1s/次 |
| Tushare 免费账户 | 分钟调用次数限制 | >= 0.5s/次 |
| CPI/PPI/北向 | 低频查询 | 每次分析只调1-2次，无需特别处理 |

---

## 九、完整数据采集流程（推荐）

```
第0层: 环境探测 (30s)
  ├── 找到Python路径
  ├── import akshare, import tushare
  └── 确认版本

第1层: API探测 (60s)
  ├── 对每个待用函数: dir(ak) 中确认存在
  ├── 拉一条数据: df.tail(3) 确认列结构
  └── 验证值在合理范围

第2层: 分层采集
  ├── AKShare: 指数日线、SW行业、PMI、LPR、M2、GDP
  ├── Tushare: 北向资金（必须）、CPI/PPI（如有权限）
  └── 合并到统一JSON

第3层: 结果验证
  ├── 收益率在±30%内
  ├── 宏观值在合理范围
  └── 标记所有NaN/异常值
```

---

## 十、关联技能速查

当其他技能需要A股数据时，引用本文档对应章节：

| 场景 | 参考章节 |
|------|---------|
| 行业轮动分析 | 三、四、五、六 |
| 财务分析（单个公司） | 三、八 |
| 量化因子筛选 | 三、四、八 |
| 宏观研判 | 三（3.1节）、四（陷阱1）、五 |
| 北向资金分析 | 四（陷阱3）、六 |

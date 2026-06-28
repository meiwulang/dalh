---
name: stock-data-fetcher
description: "A股实时行情数据抓取系统。支持大盘指数、全市场涨幅榜、涨停识别（含连板天数反推）、板块涨跌排行、个股快照、龙虎榜、资金流向、竞价数据、北向资金。触发词：数据、行情、涨停、连板、指数、板块、涨幅榜、日K、反推连板、市场情绪、实时数据、A股数据、龙虎榜、资金流向、北向、竞价。"
---

# A股实时行情数据抓取系统

> 多源聚合数据体系（新浪 + 东方财富 + 腾讯 + mootdx通达信 + 百度 + 同花顺 + 悟道A股），用于全市场短线交易数据的实时采集与分析。

---

## 一、数据接口总览

| 序号 | 数据源 | 协议 | 封IP风险 | 鉴权 | 核心用途 |
|------|--------|------|---------|------|---------|
| 1 | **新浪财经** | HTTP | ✅ 低 | 无需 | 大盘指数、个股实时快照（最稳定兜底） |
| 2 | **东方财富** | HTTP | ⚠️ 中 | 无需 | 全市场涨幅榜、行业/概念/地区板块排行、个股明细 |
| 3 | **腾讯财经** ⭐ | HTTP | ✅ 极低 | 无需 | 实时价、PE/PB/市值/换手率/涨跌停价/指数/ETF |
| 4 | **mootdx（通达信）** ⭐ | TCP:7709 | ✅ 极低 | 无需 | K线(多周期)+五档盘口+逐笔成交+46字段实时报价 |
| 5 | **百度股市通** | HTTP | ✅ 极低 | 无需 | 日K线自带MA5/MA10/MA20均价 |
| 6 | **同花顺** | HTTP | ✅ 极低 | 无需 | 强势股归类、题材归因、北向资金流向 |
| 7 | **涨停池（东财）** ⭐ | HTTP | ⚠️ 中 | 无需 | 涨停股所属行业/概念/5日/10日/20日涨幅/涨停原因 |
| 8 | **昨日涨停BK0815** | HTTP | ⚠️ 中 | 无需 | 连板成功率、涨停溢价、晋级率 |
| 9 | **悟道A股** 🔑 | HTTP | ✅ 低 | 需API Key | **仅涨停梯队ladder可用**（数据最丰富，含涨停原因/连板/主营/题材）；龙虎榜等需网页登录JWT不开放 |
| 10 | **akshare 涨停池三件套** ⭐ | HTTP | ✅ 低 | 无需 | 涨停池(含连板数)/炸板池/跌停池 — 替代悟道涨停梯队 |
| 11 | **efinance（东财封装库）** ⭐ | HTTP | ✅ 低 | 无需 | 龙虎榜/逐分钟资金流/板块成分股/板块归属 — 替代悟道龙虎榜+资金流 |

---

### 1.1 新浪指数快照（最稳定·兜底）

```
GET https://hq.sinajs.cn/list={codes}
Header: Referer: https://finance.sina.com.cn
```

**指数编码**：
| 指数 | 编码 |
|------|------|
| 上证指数 | `s_sh000001` |
| 深证成指 | `s_sz399001` |
| 创业板指 | `s_sz399006` |
| 科创50 | `s_sh000688` |
| 沪深300 | `s_sh000300` |
| 中证1000 | `s_sh000852` |
| 北证50 | `s_sz899050` |

**个股编码规则**：
- 沪市：`sh{6位代码}` → 例：`sh600353`
- 深市：`sz{6位代码}` → 例：`sz002141`
- 科创板：`sh{6位代码}` → 例：`sh688598`
- 北交所：`bj{6位代码}`

**批量请求示例**：
```
https://hq.sinajs.cn/list=s_sh000001,s_sz399001,s_sz399006,s_sh000688,sh600353,sz002141,sh688598
```

**返回格式**（ISO-8859-1编码，需转UTF-8）：
```
var hq_str_s_sh000001="上证指数,4100.35,-7.72,-0.19,3628520,85464445";
var hq_str_sz002141="贤丰控股,4.600,4.280,4.710,4.710,4.460,4.710,0.000,52141350,241070243.000,...";
```

**字段含义**（指数）：名称, 现价, 涨跌额, 涨跌幅%, 成交量(手), 成交额(元)

**字段含义**（个股）：名称, 今开, 昨收, 现价, 最高, 最低, 买一价, 卖一价, 成交量(股), 成交额(元), 盘口明细...

---

### 1.2 东方财富全市场涨幅榜（核心）

```
GET https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=500&po=1&np=1&fltt=2&fid=f3&fs=m:0+t:6,m:0+t:80,m:1+t:2,m:1+t:23,m:0+t:81+s:2048&fields=f12,f13,f14,f2,f3,f4,f5,f6,f8,f15,f16,f17,f18,f20,f21,f62
Header: Referer: https://quote.eastmoney.com
```

**参数说明**：
| 参数 | 含义 |
|------|------|
| `pn=1` | 页码 |
| `pz=500` | 每页500条 |
| `po=1` | 排序方式（1=升序，0=降序） |
| `fid=f3` | 按涨跌幅排序 |
| `fs=m:0+t:6,m:0+t:80,m:1+t:2,m:1+t:23,m:0+t:81+s:2048` | 市场筛选（深主板+创业板+沪主板+科创板+北交所） |
| `fields=f12,f13,f14,f2,f3,f4,f5,f6,f8,f15,f16,f17,f18,f20,f21,f62` | 返回字段 |

**字段含义**：
| 字段 | 含义 |
|------|------|
| f12 | 股票代码 |
| f13 | 市场标识（0=深市，1=沪市） |
| f14 | 股票名称 |
| f2 | 现价 |
| f3 | 涨跌幅% |
| f4 | 涨跌额 |
| f5 | 成交量(手) |
| f6 | 成交额(万) |
| f8 | 换手率% |
| f15 | 最高价 |
| f16 | 最低价 |
| f17 | 今开 |
| f18 | 昨收 |
| f20 | 市值 |
| f21 | 流通市值 |
| f62 | 量比 |

---

### 1.3 东方财富个股快照

```
GET https://push2.eastmoney.com/api/qt/stock/get?secid={market}.{code}&fields=f43,f44,f45,f46,f47,f48,f49,f50,f51,f52,f57,f58,f60,f71,f84,f85,f86,f107,f116,f117,f127,f168,f169,f170,f171,f191,f192,f193,f194,f195,f196,f197,f198
```

**secid规则**：
| 市场 | secid格式 |
|------|----------|
| 深市（主板/创业板） | `0.{6位代码}` → 例：`0.002141` |
| 沪市（主板/科创板） | `1.{6位代码}` → 例：`1.600353` |
| 北交所 | 按接口返回的 `f13.code` 组合 |

**关键字段**：
| 字段 | 含义 |
|------|------|
| f43 | 现价 |
| f44 | 最高 |
| f45 | 最低 |
| f46 | 今开 |
| f47 | 涨停价 |
| f48 | 跌停价 |
| f57 | 股票代码 |
| f58 | 股票名称 |
| f60 | 昨收 |
| f71 | 成交量(手) |
| f84 | 总市值 |
| f85 | 流通市值 |
| f86 | 市盈率 |
| f107 | 涨跌幅 |
| f116 | 换手率 |
| f127 | 涨跌额 |
| f168 | 换手率(实) |
| f169 | 涨跌额 |
| f170 | 涨跌幅 |
| f171 | 振幅% |

---

### 1.4 东方财富日K线（反推连板天数）

```
GET https://push2his.eastmoney.com/api/qt/stock/kline/get?secid={market}.{code}&fields1=f1,f2,f3,f4,f5,f6&fields2=f51,f52,f53,f54,f55,f56,f57,f58,f59,f60,f61&klt=101&fqt=1&beg=20260601&end=20260618
```

**参数说明**：
| 参数 | 含义 |
|------|------|
| secid | `{market}.{code}`（同个股快照规则） |
| fields1 | 固定为 f1,f2,f3,f4,f5,f6 |
| fields2 | f51=日期, f52=开盘, f53=收盘, f54=最高, f55=最低, f56=成交量, f57=成交额, f58=振幅, f59=涨跌幅%, f60=涨跌额, f61=换手率% |
| klt=101 | 日K线 |
| fqt=1 | 前复权 |
| beg/end | 起止日期 `YYYYMMDD` |

**返回格式**：
```json
{
  "data": {
    "klines": [
      "2026-06-18,40.55,42.01,42.01,41.50,31050969,1289684694.00,3.82,10.00,3.82,5.46",
      "2026-06-17,35.80,38.19,38.48,35.23,...,...",
      ...
    ]
  }
}
```

每根K线字段：日期, 开盘, 收盘, 最高, 最低, 成交量, 成交额, 振幅, 涨跌幅%, 涨跌额, 换手率%

**反推连板算法**：
```python
lines = kline_data.split(';')
board_days = 0
for i, line in enumerate(lines):
    fields = line.split(',')
    chg_pct = float(fields[8])  # 涨跌幅%
    close = float(fields[2])
    if chg_pct >= 9.8:  # 主板涨停标准
        board_days += 1
    elif chg_pct >= 19.5 and secid.startswith('0'):  # 创业板/科创板
        board_days += 1
    else:
        break  # 遇到不涨停的K线就停止计数
```

---

### 1.5 东方财富板块排行（行业·概念·地区）

同一接口，`fs` 参数区分三种板块类型：

| 板块类型 | `fs` 参数 | 说明 |
|---------|----------|------|
| 🗺️ **地区板块（领涨地区）** | `m:90+t:1` | 各省/直辖市/自治区 |
| 🏭 **行业板块（领涨板块）** | `m:90+t:2` | 申万行业分类 |
| 💡 **概念板块（领涨题材）** | `m:90+t:3` | 热门概念/题材 |

**请求示例**：
```
# 行业板块（涨跌幅降序，取前20）
GET https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=20&po=0&np=1&fltt=2&invt=2&fs=m:90+t:2&fields=f2,f3,f4,f8,f12,f14,f20,f21,f62,f104,f105,f128,f136&fid=f3

# 概念板块
GET https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=20&po=0&np=1&fltt=2&invt=2&fs=m:90+t:3&fields=f2,f3,f4,f8,f12,f14,f20,f21,f62,f104,f105,f128,f136&fid=f3

# 地区板块
GET https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=20&po=0&np=1&fltt=2&invt=2&fs=m:90+t:1&fields=f2,f3,f4,f8,f12,f14,f20,f21,f62,f104,f105,f128,f136&fid=f3
```

**返回字段含义**：
| 字段 | 含义 |
|------|------|
| f12 | 板块代码（如 BK1128） |
| f14 | 板块名称 |
| f2 | 最新价 |
| f3 | 涨跌幅% |
| f4 | 涨跌额 |
| f8 | 换手率% |
| f20 | 总市值 |
| f21 | 流通市值 |
| f62 | 量比 |
| f104 | 上涨家数 |
| f105 | 下跌家数 |
| f128 | 领涨股名称 |
| f136 | 领涨股涨跌幅% |

**获取板块成分股**：拿到板块代码（如 `BK1128`）后，用以下接口获取成分股：
```
GET https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=100&po=0&np=1&fltt=2&fs=b:BK1128&fields=f12,f14,f2,f3,f4,f5,f6,f8,f15,f16,f17,f18,f20,f21&fid=f3
```

**备注**：`fid=f3` 排序可能不生效，建议在应用层按 `f3` 排序。

---

### 1.6 东方财富涨停池接口（备用）

```
GET https://push2ex.eastmoney.com/getTopicZTPool?ut=7eea3edcaed734bea9cbfc24409ed989&d=20260618&pageindex=0&pagesize=500&sort=fbt:asc
```

**注意**：此接口当前返回 `data:null`，不可靠。改用涨幅榜+日K反推法替代。

---

### 1.7 腾讯财经（实时估值·不封IP）⭐

```
GET https://web.ifzq.gtimg.cn/appstock/app/fqkline/get?param={code},{period},{beg},{end},{limit}
```

**核心端点**：

| 用途 | URL |
|------|-----|
| 日K线（含MA均线） | `https://web.ifzq.gtimg.cn/appstock/app/fqkline/get?param=sh600519,day,,,10` |
| 实时行情/估值 | `https://qt.gtimg.cn/q=sh600519` |
| 批量行情 | `https://qt.gtimg.cn/q=sh600519,sz000001,sh600036` |

**返回格式**（批量行情）：
```
v_sh600519="1
名称,代码,现价,涨跌幅%,涨跌额,成交量(手),成交额(万),总市值,流通市值,PE(TTM),换手率%,最高,最低,今开,昨收,涨停价,跌停价,...

v_sz000001="...
```

**字段含义**（按位置索引）：
| 索引 | 含义 |
|------|------|
| 1 | 名称 |
| 2 | 代码 |
| 3 | 现价 |
| 5 | 涨跌幅% |
| 6 | 涨跌额 |
| 7 | 成交量(手) |
| 8 | 成交额(万) |
| 44 | 最高 |
| 45 | 最低 |
| 46 | 今开 |
| 47 | 昨收 |
| 48 | 涨停价 |
| 49 | 跌停价 |
| 56 | 换手率% |
| 59 | 总市值 |
| 60 | 流通市值 |
| 62 | PE(TTM) |

**特点**：
- 不封IP，比东方财富更稳定
- 返回数据包含 PE/PB/市值/换手率（新浪没有的估值数据）
- 日K线端点返回的数据自带 MA5/MA10/MA20 均线（需解析）
- 适合作为东方财富的兜底替代

---

### 1.8 mootdx 通达信接口（最全K线·盘口·不封IP）⭐

```
# 安装
pip install 'mootdx[all]'
```

**核心能力**：

```python
from mootdx.quotes import Quotes

client = Quotes.factory(market='std')  # 标准通达信行情

# 1. K线数据（多周期）
kline = client.bars(symbol='600519', frequency=9, start=0, count=120)
# frequency: 9=日线, 5=周线, 6=月线, 8=60分钟, 3=30分钟, 2=15分钟, 1=5分钟

# 2. 实时行情（46字段）
quote = client.quotes(symbols=['600519', '000001'])
# 返回：代码、名称、今开、昨收、现价、最高、最低、总手、金额、涨跌、涨幅、换手率、市盈率、市净率、总市值、流通市值...

# 3. 五档盘口
ticks = client.transactions(symbol='600519', start=0, count=100)  # 逐笔成交

# 4. 指数K线
index_kline = client.bars(symbol='000001', frequency=9, start=0, count=120)  # 上证指数
```

**特点**：
- 通达信官方数据通道，**极低封IP风险**
- 不需要网页爬取，TCP 协议直连，速度快
- K线支持多周期（1/5/15/30/60分钟，日/周/月）
- 支持五档盘口和逐笔成交（Level-2 级数据）
- **强烈推荐**替代东方财富日K线接口

---

### 1.9 百度股市通（K线自带均线）

```
GET https://gupiao.baidu.com/api/stocks/stockkline?code=sh600519&ktype=day&start_date=20260601&end_date=20260618
```

**特点**：
- 返回直接包含 MA5/MA10/MA20 均价，省去本地计算
- 极低封IP风险
- 适合需要均线数据时补充使用

---

### 1.10 同花顺热点·北向资金（零鉴权）

**北向资金流向**：
```
GET https://data.10jqka.com.cn/funds/ggjj/op=down/globalapi/market/board/north_flow
```

**强势股/题材归类**：
```
GET https://data.10jqka.com.cn/funds/hkgg/field/trade_hf/order/asc/page/1/ajax/1/
```

**特点**：
- 零鉴权，不封IP
- 数据直接可用，无需解析复杂格式
- 北向资金适合盘中监控外资动向


---

### 1.11 东方财富涨停池（含涨停原因·连板·所属行业）⭐

> 通过东方财富全市场榜单 + 涨幅筛选，提取涨停池并关联行业/概念/多周期涨幅。

**请求地址**：同 1.2 全市场涨幅榜，增加以下字段：

| 字段 | 含义 | 用途 |
|------|------|------|
| f100 | 所属行业 | 快速识别涨停股的行业归属 |
| f103 | 备注（概念标签） | **涨停原因/概念归属的关键字段** |
| f109 | **5日涨跌幅%** | 中期强度排名 |
| f152 | **20日涨跌幅%** | 中期趋势强度 |
| f161 | **10日涨跌幅%** | 短期趋势强度 |

**涨停原因获取逻辑**：
```
涨停原因 = 取 f103（备注/概念标签）解析关键词归类
- 如 f103 包含 "AI"、"机器人"、"新能源" 等 → 按对应概念归类
- 结合 f100（所属行业）+ 涨停股所属板块代码交叉验证
```

**简化版涨停池（含连板数据）推荐用 akshare**：
```python
import akshare as ak
df = ak.stock_zt_pool_em(date='20260618')
# 返回：代码, 名称, 涨跌幅, 封单金额, 流通市值, 所属行业, 涨停原因, 开板次数, 连板天数
```

---

### 1.12 全市场多周期涨幅排名（5日/10日/20日涨幅）

在东方财富全市场榜单中排序 `f109`、`f161`、`f152` 即可获取多周期涨幅排名：

```
GET https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=5000&po=0&np=1&fltt=2&fid=f109&fs=m:0+t:6,m:0+t:80,m:1+t:2,m:1+t:23,m:0+t:81+s:2048&fields=f12,f14,f2,f3,f4,f5,f6,f8,f109,f152,f161,f100,f103
```

- `fid=f109&po=0` → 5日涨幅降序（近5日最强）
- `fid=f152&po=0` → 20日涨幅降序（近1月最强）
- `fid=f161&po=0` → 10日涨幅降序

---

### 1.13 涨跌家数细分统计

通过全市场榜单（pz=5000 拉取全量）在应用层按代码前缀细分统计：

| 板块 | 代码前缀 | 涨/跌判定 |
|------|---------|----------|
| 沪市主板 | 60xxxx | f3 > 0 上涨，f3 < 0 下跌 |
| 深市主板 | 00xxxx | 同上 |
| 创业板 | 30xxxx | 同上 |
| 科创板 | 688xxx | 同上 |
| 北交所 | 4xxxxx, 8xxxxx | 同上 |

---

### 1.14 昨日涨停今日表现（连板成功率）

```
GET https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=500&po=0&np=1&fltt=2&fs=b:BK0815&fields=f12,f14,f2,f3,f4,f5,f6,f8
```

**BK0815** = 东方财富"昨日涨停"板块代码。通过该接口获取昨日涨停股今日的表现，用于计算：
- **连板成功率** = 今日继续涨停数 / 昨日涨停总数
- **涨停溢价** = 昨日涨停股今日平均涨幅

---

### 1.15 悟道A股 🔑（仅涨停梯队 API 可用·数据最丰富）

> ⚠️ **2026-06-26 实测真相**：悟道官网（`stock.quicktiny.cn`）注册申请的 `lb_` API Key **仅 `/api/ladder` 一个接口可用**。其龙虎榜等接口走 `/api/admin/*` 路径，需**网页登录后的 JWT**（cookie+Bearer），不开放给 API Key。本节只保留实测可用的 `ladder`，原登记的"26 个 API / 龙虎榜 / 竞价 / 资金流向 / 炸板池"等接口路径均不开放给 API Key 调用——这些能力改由本 skill 1.16（akshare 涨停池三件套）和 1.17（efinance）免费覆盖。

**获取 API Key**：`https://stock.quicktiny.cn/developer`（注册后在开发者后台拿 `lb_` 开头的 Key）

**唯一可用接口：涨停梯队**

```bash
# 涨停梯队（唯一对 API Key 开放的接口，数据极丰富）
curl -s -H "Authorization: Bearer $LB_API_KEY" \
  "https://stock.quicktiny.cn/api/ladder?date=$(date +%Y-%m-%d)"
```

**返回字段**（每只涨停股，2MB 量级）：
| 字段 | 含义 |
|------|------|
| `continue_num` | **连板天数**（直接返回，无需反推） |
| `high_days` | 总板数描述（如 `7天6板`） |
| `limit_up_type` | 封板类型（`一字板`/`换手板`等） |
| `reason_type` | **涨停原因**（如 `氮化铝+可控核聚变+定增受理`） |
| `reason_info` | 涨停原因详细说明（多段公告摘要） |
| `primary_theme` | 主题材 |
| `industry` | 所属行业 |
| `main_business` | 主营业务 |
| `business_scope` | 经营范围 |
| `first_limit_up_time` / `last_limit_up_time` | 首/末次封板时间（Unix 时间戳） |
| `limitAmount` | 封板资金 |
| `tradeAmount` | 成交额 |
| `turnover_rate` / `actual_turnover_rate` | 换手率（名义/真实） |
| `kpl_*` | 开盘啦系列字段（开盘竞价数据） |
| `open_num` | 开板次数 |

**特点**：
- ✅ `ladder` 数据质量**全 skill 最高**——连板天数+涨停原因+主营+题材+封板资金一站式返回，远超 akshare `stock_zt_pool_em`
- ✅ 含 `kpl_*` 开盘啦字段（开盘竞价相关），是其它免费源没有的
- ⚠️ **仅此一个接口**对 API Key 开放；龙虎榜/资金流向/炸板池等需网页登录 JWT，不开放给 API Key
- ⚠️ Key 有调用频次限制（免费额度）
- 🔁 龙虎榜需求 → 用 efinance `get_daily_billboard`（1.17 节，免费）
- 🔁 资金流向需求 → 用 efinance `get_today_bill`（1.17 节，免费，逐分钟更细）
- 🔁 炸板池需求 → 用 akshare `stock_zt_pool_zbgc_em`（1.16 节，免费）

---

### 1.16 akshare 涨停池三件套（免费·替代悟道涨停梯队）⭐

> `akshare` 封装东方财富涨停池专用接口，**连板天数/炸板次数/封板资金直接返回**，无需自推算、无需 API Key。2026-06-26 实测三接口全部可用。

**安装**：`pip install akshare`

```python
import akshare as ak

# 1. 涨停池（含连板数·封板资金·炸板次数·涨停统计·所属行业）
df_zt = ak.stock_zt_pool_em(date='20260625')
# 列：代码, 名称, 涨跌幅, 最新价, 成交额, 流通市值, 总市值, 换手率,
#     封板资金, 首次封板时间, 最后封板时间, 炸板次数, 涨停统计, 连板数, 所属行业
# 例：兴业科技(002674) 连板数=5 涨停统计='5/5' 炸板次数=0 封板资金=4.4亿 所属行业='纺织制造'

# 2. 炸板池（盘中触及涨停但未封住）
df_zb = ak.stock_zt_pool_zbgc_em(date='20260625')
# 列：代码, 名称, 涨跌幅, 最新价, 涨停价, 成交额, 流通市值, 换手率,
#     涨速, 首次封板时间, 炸板次数, 涨停统计, 振幅, 所属行业
# 例：上海港湾(605598) 炸板次数=1 涨停统计='2/1' 振幅=8.51%

# 3. 跌停池（含连续跌停·开板次数·封单资金）
df_dt = ak.stock_zt_pool_dtgc_em(date='20260625')
# 列：代码, 名称, 涨跌幅, 最新价, 成交额, 流通市值, 动态市盈率, 换手率,
#     封单资金, 最后封板时间, 板上成交额, 连续跌停, 开板次数, 所属行业
```

**连板梯队构建（无需自推算）**：
```python
df = ak.stock_zt_pool_em(date=YYYYMMDD)
# 直接按"连板数"列分组即得连板梯队
ladder = df.groupby('连板数').apply(
    lambda g: list(zip(g['名称'], g['代码'], g['封板资金'], g['所属行业']))
).to_dict()
# 输出：{5: [('兴业科技','002674',4.4亿,'纺织制造')], 4: [...], 3: [...], ...}
```

**炸板率计算**：
```python
zt_n = len(ak.stock_zt_pool_em(date=d))      # 封板数
zb_n = len(ak.stock_zt_pool_zbgc_em(date=d)) # 炸板数
break_rate = zb_n / (zt_n + zb_n) * 100       # 炸板率%
```

**特点**：
- ✅ **连板数直接返回**，彻底替代"东财涨幅榜+日K反推"算法
- ✅ **炸板次数/封板资金**直接返回，无需悟道 Key
- ✅ **涨停统计**列格式 `'连板天数/封板成功天数'`，可识别伪连板
- ✅ 含**所属行业**，涨停原因归类一步到位
- ⚠️ 网络不稳时三个接口可能独立成功/失败，**断连时分别重试三个接口**，不要因一个失败放弃全部

---

### 1.17 efinance 龙虎榜·资金流·板块归属（免费·替代悟道龙虎榜+资金流）⭐

> `efinance` 是基于东方财富爬虫的封装库，API 简洁、零鉴权。2026-06-26 实测龙虎榜/逐分钟资金流/板块归属/板块成分股全部可用。**资金流逐分钟返回，比悟道的聚合资金流更细。**

**安装**：`pip install efinance`

```python
import efinance as ef

# 1. 龙虎榜（当日全部上榜股）— 替代悟道 /dragon-tiger
df_lhb = ef.stock.get_daily_billboard('2026-06-25')
# 列：股票代码, 股票名称, 上榜日期, 解读, 收盘价, 涨跌幅, 换手率,
#     龙虎榜净买额, 龙虎榜买入额, 龙虎榜卖出额, 龙虎榜成交额, 市场总成交额,
#     净买额占总成交比, 成交额占总成交比, 流通市值, 上榜原因
# 例：101行 × 16列；含"上榜原因"和"解读"（如"卖一主卖，成功率50.92%"）

# 2. 逐分钟资金流（主力/超大单/大单/中单/小单）— 替代悟道 /money-flow
df_flow = ef.stock.get_today_bill('600519')
# 列：股票名称, 股票代码, 时间, 主力净流入, 小单净流入, 中单净流入,
#     大单净流入, 超大单净流入
# 例：240行（全天每分钟一行），可计算主力全天累计净流入

# 3. 个股所属全部板块（行业+概念）— 替代东财交叉拼接
df_board = ef.stock.get_belong_board('600519')
# 列：股票名称, 股票代码, 板块代码, 板块名称, 板块涨幅
# 例：29行 × 5列；贵州茅台属于食品饮料、白酒、超级品牌等29个板块

# 4. 板块成分股（按板块名称查）— 替代东财手搓成分股接口
df_mem = ef.stock.get_members('白酒')
# 列：指数代码, 指数名称, 股票代码, 股票名称, 股票权重
# 例：17行 × 5列；含权重，可识别板块龙头

# 5. 最新实时快照（单股，37字段 Series）
snap = ef.stock.get_quote_snapshot('600519')  # pandas Series

# 6. IPO 审核队列（题材打新参考）
df_ipo = ef.stock.get_latest_ipo_info()
# 列：发行人全称, 审核状态, 注册地, 证监会行业, 保荐机构, 会计师事务所,
#     更新日期, 受理日期, 拟上市地点
```

**主力资金全天累计净流入**：
```python
df = ef.stock.get_today_bill('600519')
main_net = df['主力净流入'].sum()   # 全天主力净流入（元）
big_net  = df['超大单净流入'].sum() # 超大单累计
```

**龙虎榜净买额排行**：
```python
df = ef.stock.get_daily_billboard(date_str)
top_buy = df.sort_values('龙虎榜净买额', ascending=False).head(10)
for _, r in top_buy.iterrows():
    print(f"{r['股票名称']}({r['股票代码']}) 净买{r['龙虎榜净买额']/1e4:.0f}万 原因:{r['上榜原因']}")
```

**特点**：
- ✅ **龙虎榜含上榜原因+解读**，比悟道字段更丰富（含"成功率"统计）
- ✅ **资金流逐分钟返回**，可做盘中分时资金流分析（悟道只有聚合值）
- ✅ **板块归属一键查全量**，替代东财"涨幅榜交叉拼接板块代码"的笨办法
- ✅ **板块成分股按名称查**，无需先拿板块代码
- ✅ IPO 审核队列是当前 skill 完全缺失的数据（打新题材/次新股预热）
- ⚠️ `get_realtime_quotes`(全市场批量)和`get_quote_history`(K线)底层走 push2.eastmoney.com，断连风险同东财原生接口，**这两块仍用腾讯/mootdx**，efinance 仅用于龙虎榜/资金流/板块

---

## 二、涨停筛选规则

| 市场板块 | 涨停判定标准 |
|---------|------------|
| 沪市主板(sh60xxxx) | 涨幅 >= 9.8% |
| 深市主板(sz00xxxx/sz30xxxx非3开头) | 涨幅 >= 9.8% |
| 创业板(sz30xxxx) | 涨幅 >= 19.5% |
| 科创板(sh688xxx) | 涨幅 >= 19.5% |
| 北交所 | 涨幅 >= 29% |
| ST/*ST | 涨幅 >= 4.8% |

---

## 三、数据采集流程

### 流程1：获取大盘情绪
```
1. 新浪指数快照 → 获取7大指数实时数据
2. 或：腾讯财经批量行情 → 同步获取指数+估值数据
3. 计算：涨跌家数比、涨停/跌停比
4. 输出：市场情绪温度（冰点/低迷/正常/活跃/高潮）
```

### 流程2：获取涨停梯队

**方式A（自推算·免费·无需Key）**：
```
1. 东方财富涨幅榜(pz=500) → 全市场涨幅排序
2. 按涨停规则筛选 → 得到当日涨停股列表（含涨幅信息）
3. 对每只涨停股 → mootdx通达信K线(或东财K线兜底) → 反推连续涨停天数
4. 按涨停天数分组 → 输出连板梯队（6板/5板/4板/3板/2板/首板）
5. 统计：严格涨停总数、炸板数（通过涨幅靠近涨停但未封住识别）
```

**方式B（悟道API·需Key·推荐）**：
```
1. 悟道 /ladder → 直接获取连板梯队（含每层个股）
2. 悟道 /break-pool → 获取炸板池（含炸板率）
3. 悟道 /limit-pool → 获取跌停池
4. 合并输出完整情绪数据
```

### 流程3：获取板块热点（领涨板块·领涨题材·领涨地区）

**方式A（免费·推荐）**：
```
1. 东方财富行业板块排行 t:2 → 🏭 领涨板块 Top 20
2. 东方财富概念板块排行 t:3 → 💡 领涨题材 Top 20
3. 东方财富地区板块排行 t:1 → 🗺️ 领涨地区 Top 10
4. 每板块均含：涨跌幅、上涨/下跌家数、领涨股名称及涨幅
5. 同花顺北向资金 → 监测外资流向
6. 三表交叉验证 → 识别真主线（行业共振+题材配合+地域集中）
```

**方式B（悟道API·需Key）**：
```
1. 悟道 /hot-concept → 最强风口（按涨停聚集度排名）
2. 悟道 /sector-rotation → 板块四象限动量分析
3. 悟道 /concept-constituents → 热点概念成分股
```

### 流程4：获取资金流向（新增）
```
1. 悟道 /money-flow → 个股/板块/大盘资金流向
2. 或：同花顺北向资金接口 → 外资净买入额
3. 识别主力净流入/流出板块
```

### 流程5：获取龙虎榜（新增·需悟道Key）
```
1. 悟道 /dragon-tiger → 当日龙虎榜数据
2. 提取：买入营业部、卖出营业部、净买额
3. 识别：知名游资席位动向（如：中关村、拉萨、深股通等）
```

### 流程6：获取竞价数据（新增·需悟道Key）
```
1. 悟道 /auction → 个股集合竞价成交数据
2. 分析：竞价量能、撮合价格、分歧程度
3. 用于：开盘前的隔夜方向判断
```

### 流程7：获取个股详情

**轻量版（腾讯·最快·不封IP）**：
```
1. 腾讯 qt.gtimg.cn 批量行情 → PE/PB/市值/换手率/涨停跌停价/五档
```

**标准版（新浪兜底）**：
```
1. 新浪个股快照 → 实时分时数据（稳定兜底）
2. 东方财富个股快照 → 详细指标（市值/换手率/振幅等）
```

**专业版（mootdx·最全）**：
```
1. mootdx quotes() → 46字段实时行情（含盘口深度）
2. mootdx bars() → 多周期K线（替代东财日K）
3. mootdx transactions() → 逐笔成交数据
```

### 流程8：涨跌家数细分统计（新增）
```
1. 全市场涨幅榜(pz=5000) → 拉取全量股票行情
2. 按 f3 涨跌幅统计 → 上涨/下跌/平盘家数
3. 按代码前缀细分 → 沪主板/深主板/创业板/科创板/北交所
4. 累计成交额 → 全市场总成交额(亿)
```

### 流程9：多周期涨幅排名（新增）
```
1. 东方财富全市场榜单 fid=f109 → 5日涨幅降序排名
2. 东方财富全市场榜单 fid=f161 → 10日涨幅降序排名
3. 东方财富全市场榜单 fid=f152 → 20日涨幅降序排名
4. 结合涨停股 → 识别中期强势股
```

### 流程10：连板成功率监控（新增）
```
1. 昨日涨停板块 BK0815 → 获取昨日涨停股今日表现
2. 统计继续涨停数 → 计算连板成功率
3. 统计大阳线数 → 计算溢价率
4. 平均涨幅 → 涨停溢价
```

---

## 四、市场情绪指标体系

### 4.1 基础情绪指标（免费接口即可获取）

| 指标 | 数据来源 | 说明 |
|------|---------|------|
| 涨停数 | 涨幅榜筛选 / 悟道涨停统计 | 全市场严格涨停家数 |
| 连板数 | 涨幅榜+日K反推 / 悟道涨停梯队 | 2板及以上连板股数量 |
| 最高板 | 日K反推 / 悟道涨停梯队 | 当前最高连板高度 |
| 涨停晋级率 | 昨日首板→今日2板比例 | 反映接力意愿 |
| 跌停数 | **akshare `stock_zt_pool_dtgc_em(date)`**（唯一正确源） | 市场亏钱效应 |
| 涨跌家数比 | 涨幅榜 | 上涨/下跌家数 |
| 炸板率 | 悟道炸板池 / 涨幅榜估算 | 涨停打开比例 |
| 成交额 | 指数快照 / 腾讯行情 | 全市场总成交额 |
| 涨跌停比 | 涨停数/跌停数 | 情绪强度 |
| 指数强度 | 新浪/腾讯指数快照 | 主要指数涨跌幅 |
| 市场温度 | 综合以上 | 冰点/低迷/正常/活跃/高潮 |

### 4.2 进阶情绪指标（需悟道Key）

| 指标 | 数据来源 | 说明 |
|------|---------|------|
| 涨停溢价 | 悟道 涨停溢价 | 涨停股次日开盘溢价率 |
| 封板率 | 悟道 涨跌停统计 | 封板数/（封板数+炸板数） |
| 最强风口 | 悟道 最强风口 | 按涨停聚集度排名的板块 |
| 封板事件 | 悟道 封板事件流 | 实时封板/炸板事件，精确到秒 |
| 北向资金 | 悟道 资金流向 / 同花顺 | 外资当日净买入额 |
| 主力净流入 | 悟道 资金流向 | 主力资金流入/流出 |
| 板块轮动 | 悟道 板块轮动 | 板块动量与强度四象限 |
| 异动检测 | 悟道 异动检测 | 个股异常交易行为 |
| 龙虎榜 | 悟道 龙虎榜 | 游资席位动向 |
| 竞价数据 | 悟道 竞价数据 | 集合竞价成交情况 |

---

## 五、输出格式模板

### 大盘情绪快照输出模板
```
📊 大盘情绪（YYYY-MM-DD HH:MM）
上证 {点位} {涨跌幅}% | 深证 {点位} {涨跌幅}%
创业板 {点位} {涨跌幅}% | 科创50 {点位} {涨跌幅}%
成交额 {万亿} ｜ 涨停 {N} 只 ｜ 跌停 {N} 只 ｜ 炸板率 {N}%
```

### 涨停梯队输出模板
```
📈 连板梯队
- 4板：{名称}({代码})
- 3板：{名称}({代码})、{名称}({代码})
- 2板：{名称}({代码})、{名称}({代码}) ...
- 首板：共 {N} 只
- 全市场严格涨停：{N} 只
```

### 板块排行输出模板

```
🔥 行业板块涨幅前5
1. {板块名称} {涨幅}% ｜ 领涨：{领涨股}({涨幅}%) ｜ {上涨家数}/{下跌家数}

🔥 概念板块涨幅前5
1. {板块名称} {涨幅}% ｜ 领涨：{领涨股}({涨幅}%) ｜ {上涨家数}/{下跌家数}

🔥 地区板块涨幅前5
1. {省份/直辖市} {涨幅}% ｜ 领涨：{领涨股}({涨幅}%) ｜ {上涨家数}/{下跌家数}
```

### 快速调用的 Python 示例

```python
# 一行获取领涨行业
industries = em_board_ranks("t:2", top_n=10)
for b in industries:
    print(f"{b['f14']}: {b['f3']:+.2f}%  领涨:{b['f128']}({b['f136']}%)  {b['f104']}/{b['f105']}家")

# 一行获取领涨题材
concepts = em_board_ranks("t:3", top_n=10)

# 一行获取领涨地区
regions = em_board_ranks("t:1", top_n=10)
```

### 龙虎榜输出模板（新增·需悟道Key）
```
🐅 龙虎榜（YYYY-MM-DD）
{名称}({代码})：净买入 {N} 万
├─ 买一：{营业部} {金额}万
├─ 买二：{营业部} {金额}万
├─ 卖一：{营业部} {金额}万
└─ 卖二：{营业部} {金额}万
```

### 资金流向输出模板（新增·需悟道Key）
```
💰 资金流向
大盘：主力净流入 {N} 亿 ｜ 北向净买入 {N} 亿
板块净流入前3：{板块1}、{板块2}、{板块3}
板块净流出前3：{板块1}、{板块2}、{板块3}
```

### 竞价数据输出模板（新增·需悟道Key）
```
⏰ {名称}({代码}) 集合竞价
竞价量：{N} 万手 ｜ 撮合价：{price}
隔夜方向：{抢筹/分歧/砸盘}
```

### 涨跌家数细分模板（新增）
```
📊 全市场情绪
上涨 {N} 家 ｜ 下跌 {N} 家 ｜ 平盘 {N} 家 ｜ 成交额 {N} 亿
├─ 沪主板: ↑{N} ↓{N}
├─ 深主板: ↑{N} ↓{N}
├─ 创业板: ↑{N} ↓{N}
├─ 科创板: ↑{N} ↓{N}
└─ 北交所: ↑{N} ↓{N}
```

### 多周期涨幅排名模板（新增）
```
📈 5日涨幅 Top 5
1. {名称}({代码}) +{N}% ｜ {所属行业}
2. ...

📈 20日涨幅 Top 5
1. {名称}({代码}) +{N}% ｜ {所属行业}
```

### 连板成功率模板（新增）
```
🎯 连板晋级率
昨日涨停 {N} 只 → 今日继续涨停 {N} 只
晋级率: {N}% ｜ 大阳率(>=7%): {N}% ｜ 平均涨幅: {N}%
```

---

## 六、Python工具函数

```python
import requests, json
from datetime import datetime

# ---------- 新浪接口 ----------
def sina_quote(codes_str: str) -> str:
    """获取新浪实时快照"""
    url = f"https://hq.sinajs.cn/list={codes_str}"
    headers = {"Referer": "https://finance.sina.com.cn"}
    resp = requests.get(url, headers=headers, timeout=10)
    resp.encoding = 'gbk'
    return resp.text

def parse_sina_index(text: str) -> list[dict]:
    """解析新浪指数数据"""
    results = []
    for line in text.strip().split('\n'):
        if not line.startswith('var hq_str_s_'):
            continue
        parts = line.split('="')[1].rstrip('";').split(',')
        name, price, change, pct = parts[0], parts[1], parts[2], parts[3]
        results.append({"name": name, "price": float(price), "change": float(change), "pct": float(pct)})
    return results

def parse_sina_stock(text: str) -> dict:
    """解析新浪个股数据"""
    line = text.strip().split('\n')[0]
    parts = line.split('="')[1].rstrip('";').split(',')
    return {
        "name": parts[0], "open": float(parts[1]), "yclose": float(parts[2]),
        "price": float(parts[3]), "high": float(parts[4]), "low": float(parts[5]),
        "volume": int(parts[8]), "amount": float(parts[9])
    }

# ---------- 东方财富接口 ----------
def em_market_ranks(pz: int = 500) -> list[dict]:
    """获取全市场涨幅榜"""
    url = ("https://push2.eastmoney.com/api/qt/clist/get"
           f"?pn=1&pz={pz}&po=1&np=1&fltt=2&fid=f3"
           "&fs=m:0+t:6,m:0+t:80,m:1+t:2,m:1+t:23,m:0+t:81+s:2048"
           "&fields=f12,f13,f14,f2,f3,f4,f5,f6,f8,f15,f16,f17,f18,f20,f21,f62")
    headers = {"Referer": "https://quote.eastmoney.com"}
    resp = requests.get(url, headers=headers, timeout=15)
    data = resp.json()
    return data.get('data', {}).get('diff', [])

def em_stock_kline(secid: str, beg: str = "20260601", end: str = None) -> list[str]:
    """获取日K线"""
    if end is None:
        end = datetime.now().strftime("%Y%m%d")
    url = (f"https://push2his.eastmoney.com/api/qt/stock/kline/get"
           f"?secid={secid}&fields1=f1,f2,f3,f4,f5,f6"
           f"&fields2=f51,f52,f53,f54,f55,f56,f57,f58,f59,f60,f61"
           f"&klt=101&fqt=1&beg={beg}&end={end}")
    headers = {"Referer": "https://quote.eastmoney.com"}
    resp = requests.get(url, headers=headers, timeout=15)
    data = resp.json()
    return data.get('data', {}).get('klines', [])

def em_board_ranks(board_type: str = "t:2", top_n: int = 20) -> list[dict]:
    """获取板块排行（含重试）
    board_type: 't:1'=地区板块, 't:2'=行业板块, 't:3'=概念板块
    返回字段：板块名称(f14)、涨跌幅(f3)、上涨家数(f104)、下跌家数(f105)、
             领涨股(f128)、领涨股涨幅(f136)、换手率(f8)、总市值(f20)
    """
    import time
    url = (f"https://push2.eastmoney.com/api/qt/clist/get"
           f"?pn=1&pz={top_n}&po=0&np=1&fltt=2&invt=2"
           f"&fs=m:90+{board_type}&fields=f2,f3,f4,f8,f12,f14,f20,f21,f62,f104,f105,f128,f136&fid=f3")
    headers = {"Referer": "https://quote.eastmoney.com"}
    for attempt in range(3):
        try:
            resp = requests.get(url, headers=headers, timeout=15)
            data = resp.json()
            items = data.get('data', {}).get('diff', [])
            return sorted(items, key=lambda x: x.get('f3', 0), reverse=True)
        except Exception:
            if attempt < 2:
                time.sleep(1)
                continue
            return []  # 断连时返回空列表，由调用方决定兜底策略

# ---------- 涨停筛选 ----------
def is_limit_up(item: dict) -> bool:
    """判断是否涨停"""
    pct = float(item.get('f3', 0))
    code = str(item.get('f12', ''))
    if code.startswith(('688', '300')):  # 科创板/创业板
        return pct >= 19.5
    elif code.startswith('4') or code.startswith('8'):  # 北交所
        return pct >= 29.0
    elif item.get('f14', '').startswith(('ST', '*ST')):
        return pct >= 4.8
    else:
        return pct >= 9.8

def calc_board_days(klines: list[str], code: str) -> int:
    """从日K反推连续涨停天数"""
    days = 0
    for kline in klines:
        fields = kline.split(',')
        pct = float(fields[8])  # 涨跌幅%
        if code.startswith(('688', '300')):
            if pct >= 19.5:
                days += 1
            else:
                break
        else:
            if pct >= 9.8:
                days += 1
            else:
                break
    return days

# ---------- 腾讯财经接口（不封IP·最稳定估值） ----------
def tx_quote(codes: str) -> str:
    """获取腾讯批量实时行情"""
    url = f"https://qt.gtimg.cn/q={codes}"
    resp = requests.get(url, timeout=10)
    resp.encoding = 'gbk'
    return resp.text

def parse_tx_quote(text: str) -> list[dict]:
    """解析腾讯行情数据"""
    results = []
    for line in text.strip().split('\n'):
        if not line.startswith('v_'):
            continue
        parts = line.split('="')[1].rstrip('";').split('~')
        results.append({
            "name": parts[1], "code": parts[2], "price": float(parts[3]) if parts[3] else 0,
            "pct": float(parts[32]) if parts[32] else 0, "change": float(parts[31]) if parts[31] else 0,
            "volume": int(parts[6]) if parts[6] else 0, "amount": float(parts[37]) if parts[37] else 0,
            "high": float(parts[33]) if parts[33] else 0, "low": float(parts[34]) if parts[34] else 0,
            "open": float(parts[35]) if parts[35] else 0, "yclose": float(parts[36]) if parts[36] else 0,
            "pe": float(parts[39]) if parts[39] else 0, "turnover": float(parts[38]) if parts[38] else 0,
            "total_mv": float(parts[45]) if parts[45] else 0, "float_mv": float(parts[44]) if parts[44] else 0,
            "high_limit": float(parts[48]) if parts[48] else 0, "low_limit": float(parts[49]) if parts[49] else 0,
        })
    return results

# ---------- mootdx 通达信接口（不封IP·最全K线） ----------
def mootdx_kline(code: str, frequency: int = 9, count: int = 120) -> list:
    """获取mootdx K线数据
    frequency: 9=日线, 5=周线, 6=月线, 8=60分, 3=30分, 2=15分, 1=5分
    """
    try:
        from mootdx.quotes import Quotes
        client = Quotes.factory(market='std')
        return client.bars(symbol=code, frequency=frequency, start=0, count=count)
    except ImportError:
        raise ImportError("请先安装 mootdx: pip install 'mootdx[all]'")

def mootdx_quote(codes: list) -> list:
    """获取mootdx实时行情（46字段）"""
    try:
        from mootdx.quotes import Quotes
        client = Quotes.factory(market='std')
        return client.quotes(symbols=codes)
    except ImportError:
        raise ImportError("请先安装 mootdx: pip install 'mootdx[all]'")

# ---------- 涨跌家数细分统计 ----------
def market_sentiment(pz: int = 5000) -> dict:
    """获取全市场情绪：涨跌家数细分 + 总成交额"""
    items = em_market_ranks(pz=pz)
    result = {
        "total_up": 0, "total_down": 0, "total_flat": 0, "total_amount": 0,
        "sh_main": {"up": 0, "down": 0},         # 沪主板 60xxxx
        "sz_main": {"up": 0, "down": 0},         # 深主板 00xxxx
        "cyb": {"up": 0, "down": 0},             # 创业板 30xxxx
        "kcb": {"up": 0, "down": 0},             # 科创板 688xxx
        "bj": {"up": 0, "down": 0},              # 北交所 4/8xxxx
    }
    for item in items:
        code = str(item.get('f12', ''))
        pct = float(item.get('f3', 0))
        amount = float(item.get('f6', 0))
        result["total_amount"] += amount
        if pct > 0:
            result["total_up"] += 1
        elif pct < 0:
            result["total_down"] += 1
        else:
            result["total_flat"] += 1
        # 按板块细分
        if code.startswith('60'):
            target = result["sh_main"]
        elif code.startswith('00'):
            target = result["sz_main"]
        elif code.startswith('30'):
            target = result["cyb"]
        elif code.startswith('688'):
            target = result["kcb"]
        elif code.startswith(('4', '8')):
            target = result["bj"]
        else:
            continue
        if pct > 0:
            target["up"] += 1
        elif pct < 0:
            target["down"] += 1
    result["total_amount_亿"] = round(result["total_amount"] / 10000, 2)
    return result

# ---------- 多周期涨幅排名 ----------
def multi_day_ranks(period: str = "f109", top_n: int = 20) -> list[dict]:
    """获取多周期涨幅排名
    period: 'f109'=5日, 'f161'=10日, 'f152'=20日
    """
    import time
    url = ("https://push2.eastmoney.com/api/qt/clist/get"
           f"?pn=1&pz={top_n}&po=0&np=1&fltt=2&fid={period}"
           "&fs=m:0+t:6,m:0+t:80,m:1+t:2,m:1+t:23,m:0+t:81+s:2048"
           "&fields=f12,f14,f2,f3,f4,f5,f6,f8,f109,f152,f161,f100,f103")
    headers = {"Referer": "https://quote.eastmoney.com"}
    for attempt in range(3):
        try:
            resp = requests.get(url, headers=headers, timeout=15)
            data = resp.json()
            items = data.get('data', {}).get('diff', [])
            return sorted(items, key=lambda x: x.get(period, 0), reverse=True)
        except Exception:
            if attempt < 2:
                time.sleep(1)
                continue
            return []

# ---------- 昨日涨停今日表现（连板成功率） ----------
def yest_limitup_today() -> list[dict]:
    """获取昨日涨停股今天的表现"""
    url = ("https://push2.eastmoney.com/api/qt/clist/get"
           "?pn=1&pz=500&po=0&np=1&fltt=2"
           "&fs=b:BK0815&fields=f12,f14,f2,f3,f4,f5,f6,f8,f15,f16,f17,f18")
    headers = {"Referer": "https://quote.eastmoney.com"}
    resp = requests.get(url, headers=headers, timeout=15)
    data = resp.json()
    return data.get('data', {}).get('diff', [])

def calc_promotion_rate(items: list[dict]) -> dict:
    """计算连板成功率"""
    total = len(items)
    up2 = sum(1 for i in items if float(i.get('f3', 0)) >= 9.8)
    up7 = sum(1 for i in items if float(i.get('f3', 0)) >= 7)
    avg_pct = sum(float(i.get('f3', 0)) for i in items) / total if total else 0
    return {
        "total": total,
        "continue_limitup": up2,
        "promotion_rate": round(up2/total*100, 1) if total else 0,
        "big_positive_rate": round(up7/total*100, 1) if total else 0,
        "avg_pct": round(avg_pct, 2),
    }
```

---

## 七、完整使用示例

```python
# ==================== 方式一：原始方案（新浪+东财） ====================

# 1. 获取大盘指数（新浪）
text = sina_quote("s_sh000001,s_sz399001,s_sz399006,s_sh000688")
indices = parse_sina_index(text)
for idx in indices:
    print(f"{idx['name']}: {idx['price']}  {idx['pct']:+.2f}%")

# 2. 获取涨停股（东财涨幅榜）
items = em_market_ranks(pz=500)
limit_up_stocks = [i for i in items if is_limit_up(i)]
print(f"涨停总数: {len(limit_up_stocks)}")

# 3. 反推连板天数（东财日K）
for stock in limit_up_stocks[:5]:
    code = stock['f12']
    market = 0 if code.startswith(('00','30')) else 1
    secid = f"{market}.{code}"
    klines = em_stock_kline(secid)
    boards = calc_board_days(klines, code)
    print(f"{stock['f14']}({code}): {boards}连板")


# ==================== 方式二：推荐方案（腾讯+mootdx，不封IP） ====================

# 1. 大盘指数 + 个股估值（腾讯，不封IP）
tx_data = tx_quote("sh000001,sz399001,sz399006,sh000688")
quotes = parse_tx_quote(tx_data)
for q in quotes:
    print(f"{q['name']}: {q['price']}  {q['pct']:+.2f}%  PE={q['pe']}")

# 2. 个股K线（mootdx通达信，不封IP）
kline = mootdx_kline('600519', frequency=9, count=30)
print(kline.tail())  # pandas DataFrame 格式

# 3. 个股实时行情（mootdx，46字段）
quotes = mootdx_quote(['600519', '000001'])
for q in quotes:
    print(f"{q['code']} {q['price']} 涨幅{q['pct']}%")

# ==================== 板块排行（领涨板块·题材·地区） ====================

# 领涨行业板块 Top 10
industries = em_board_ranks("t:2", top_n=10)
print("🏭 领涨行业板块：")
for b in industries:
    print(f"  {b['f14']}: {b['f3']:+.2f}%  领涨:{b.get('f128','')}({b.get('f136',0):+.2f}%)  {b.get('f104',0)}/{b.get('f105',0)}家")

# 领涨概念题材 Top 10
concepts = em_board_ranks("t:3", top_n=10)
print("💡 领涨题材：")
for b in concepts:
    print(f"  {b['f14']}: {b['f3']:+.2f}%  领涨:{b.get('f128','')}({b.get('f136',0):+.2f}%)")

# 领涨地区 Top 10
regions = em_board_ranks("t:1", top_n=10)
print("🗺️ 领涨地区：")
for b in regions:
    print(f"  {b['f14']}: {b['f3']:+.2f}%  领涨:{b.get('f128','')}({b.get('f136',0):+.2f}%)")


# ==================== 涨跌家数细分 ====================

sentiment = market_sentiment(pz=5000)
print(f"📊 全市场: ↑{sentiment['total_up']} / ↓{sentiment['total_down']} / 成交{sentiment['total_amount_亿']}亿")
print(f"  沪主板: ↑{sentiment['sh_main']['up']} / ↓{sentiment['sh_main']['down']}")
print(f"  深主板: ↑{sentiment['sz_main']['up']} / ↓{sentiment['sz_main']['down']}")
print(f"  创业板: ↑{sentiment['cyb']['up']} / ↓{sentiment['cyb']['down']}")
print(f"  科创板: ↑{sentiment['kcb']['up']} / ↓{sentiment['kcb']['down']}")


# ==================== 多周期涨幅排名 ====================

top5 = multi_day_ranks("f109", 5)  # 5日涨幅前5
print("📈 5日涨幅排名：")
for s in top5:
    print(f"  {s['f14']}({s['f12']}): 5日+{s.get('f109',0):.2f}%  所属:{s.get('f100','')}")

top20 = multi_day_ranks("f152", 5)  # 20日涨幅前5
print("📈 20日涨幅排名：")
for s in top20:
    print(f"  {s['f14']}({s['f12']}): 20日+{s.get('f152',0):.2f}%  所属:{s.get('f100','')}")


# ==================== 连板成功率 ====================

items = yest_limitup_today()
rate = calc_promotion_rate(items)
print(f"🎯 连板成功率: {rate['promotion_rate']}% ({rate['continue_limitup']}/{rate['total']})")
print(f"   平均涨幅: {rate['avg_pct']:+.2f}%  | 大阳率(>=7%): {rate['big_positive_rate']}%")


# ==================== 方式三：专业方案（悟道API，需Key） ====================

# import requests
# LB_API_KEY = "your_key_here"
# LB_API_BASE = "https://stock.quicktiny.cn/api"
# headers = {"Authorization": f"Bearer {LB_API_KEY}"}

# # 涨停梯队
# resp = requests.get(f"{LB_API_BASE}/ladder?date=2026-06-18", headers=headers)
# print(resp.json())

# # 龙虎榜
# resp = requests.get(f"{LB_API_BASE}/dragon-tiger?date=2026-06-18", headers=headers)
# print(resp.json())

# # 竞价数据
# resp = requests.get(f"{LB_API_BASE}/auction?code=600519", headers=headers)
# print(resp.json())
```

---

## 八、注意事项

### 8.1 各数据源注意事项

| 数据源 | 注意点 |
|--------|--------|
| **新浪** | 返回GBK编码，需做 `resp.encoding = 'gbk'` 或 `iconv -f GBK -t UTF-8` 转换 |
| **东方财富** | `po=1`（升序）从最小涨幅开始排，需结合涨幅筛选使用；偶尔出现 `RemoteDisconnected` 断连。若全市场/板块接口连续返回空或断连，立即用 akshare 兜底：`python3 -m pip install --user -q akshare` 后调用 `stock_zh_a_spot_em()`、`stock_zt_pool_em(date=YYYYMMDD)`、`stock_board_industry_name_em()`、`stock_board_concept_name_em()`；若 `stock_zh_a_spot_em()` 也断连，不要放弃，继续尝试 `stock_zt_pool_em` / `stock_zt_pool_dtgc_em` / `stock_zt_pool_zbgc_em`，涨停池、跌停池、炸板池常可独立成功。 |
| **腾讯财经** | 返回字段较多（60+字段），按位置索引取值，注意空字段保护；不封IP，推荐优先使用 |
| **mootdx** | 首次使用需 `pip install 'mootdx[all]'`；M1/M2 Mac 需用 Rosetta：`arch -x86_64 pip install mootdx`；如连接超时，加 `bestip=True` 参数或手动指定服务器 |
| **百度股市通** | 接口稳定性一般，建议作为辅助数据源 |
| **同花顺** | 接口可能会变化，需定期验证；北向资金接口返回原始数据 |
| **悟道A股** | 需要 API Key（`https://stock.quicktiny.cn/developer` 注册）；免费额度有调用次数限制；MCP 协议配置可选 |

### 8.2 数据源选择优先级

| 场景 | 首选 | 兜底 |
|------|------|------|
| 大盘指数 | 新浪 → 腾讯 | 东财 |
| 涨停股列表 | **akshare `stock_zt_pool_em`**（含连板数） | 东财涨幅榜筛选 |
| 涨停原因/概念 | 东财涨停池(f103) / 悟道 | akshare `stock_zt_pool_em`（含所属行业） |
| **连板梯队** | **akshare `stock_zt_pool_em`**（连板数列直接分组） | 悟道 / 自推算 |
| **炸板池/炸板率** | **akshare `stock_zt_pool_zbgc_em`** | 悟道炸板池 |
| **跌停池** | **akshare `stock_zt_pool_dtgc_em`（唯一正确源）** | ⚠️ 禁止用涨幅榜筛选f3<=-9.8%自己算跌停！见下方错误说明 |

### ⚠️ 跌停判定错误说明（2026-06-28 修正）

**历史错误**：自己从全市场行情用涨跌幅阈值判定跌停，导致数据严重偏差。

| 错误方式 | 结果 | 原因 |
|---------|------|------|
| `pct <= -9.8` 判定所有板块跌停 | 74只（高估2倍+） | 把跌超9.8%但未跌停的也算入 |
| 688/300开头跌-10%算跌停 | 多算20+只 | 科创板/创业板跌停是-20%，跌-10%不是跌停！ |
| 新浪API按-10%±0.15%匹配但未区分板块 | 56只 | 688/300跌-10%被错误归入主板判定 |

**正确方式**：直接调用 `ak.stock_zt_pool_dtgc_em(date="YYYYMMDD")`，返回东财官方跌停池数据。

```python
import akshare as ak

# ✅ 正确：用跌停池API
df = ak.stock_zt_pool_dtgc_em(date="20260626")
print(f"跌停: {len(df)}只")  # 30只

# ❌ 错误：自己从全市场行情算
# spot = ak.stock_zh_a_spot_em()
# dt = spot[spot['涨跌幅'] <= -9.8]  # 会多算很多非跌停股！
```

**跌停幅度规则**（仅作参考理解，不要自己实现判定）：
- 主板（60/000/002/003等）：跌停 -10%，ST -5%
- 创业板（300/301）：跌停 -20%
- 科创板（688）：跌停 -20%
- 北交所（4/8）：跌停 -30%

**铁律：跌停数据只用 `stock_zt_pool_dtgc_em`，永远不要自己算。**
| 连板天数反推 | akshare连板数列 / mootdx（不封IP） | 东财日K |
| 估值（PE/PB/市值） | **腾讯**（唯一可靠来源） | — |
| K线数据 | **mootdx**（多周期） | 东财 → 百度 |
| **龙虎榜** | **efinance `get_daily_billboard`**（含上榜原因+解读） | 悟道（需Key） |
| **个股资金流向** | **efinance `get_today_bill`**（逐分钟5档资金流） | 悟道（需Key） |
| **板块归属/成分股** | **efinance `get_belong_board`/`get_members`** | 东财手搓拼接 |
| 竞价数据 | 悟道（需Key） | — |
| 板块热点（行业/概念/地区） | `em_board_ranks()` | 悟道最强风口 |
| **涨跌家数细分** | `market_sentiment()` | — |
| **5日/10日/20日涨幅排名** | `multi_day_ranks()` | mootdx计算 |
| **连板成功率/晋级率** | `yest_limitup_today()` | 悟道 |
| **IPO审核队列** | **efinance `get_latest_ipo_info`** | — |
| 北向资金 | 同花顺 | 悟道资金流向 |

### 8.3 通用规则

1. **日K反推**：从最新一天向历史回溯，遇到不涨停即停止
2. **secid规则**：深市=`0.代码`，沪市=`1.代码`，北交所按f13组合
3. **板块排序**：`fid=f3`排序可能不生效，建议在应用层自行排序
4. **涨停池接口**：东方财富专用涨停池接口可能返回null，不可作为唯一来源
5. **mootdx 服务器**：通达信行情服务器可能变更，定期执行 `pip install -U mootdx`

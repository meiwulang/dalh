# A股行情数据拉取技能

基于 Abu量化、VeighNa(vnpy)、天勤量化(TqSdk) 三大开源量化框架的数据层设计，提供标准化的 A 股实时行情、历史数据和市场统计信息获取能力。

---

## 数据源架构（参考三大框架）

```
┌────────────────────────────────┐
│        你的调用层 (Skill)         │
├────────────────────────────────┤
│    DataHub (统一数据枢纽)         │ ← Abu量化 的 DataHub 模式
├────────┬───────────┬───────────┤
│  Sina  │ EastMoney │   Tencent │ ← 多数据源容灾
│  Public│  Push API │  Stock   │
│   API  │           │   API    │
└────────┴───────────┴───────────┘
```

### 三大框架的数据层设计参考

| 框架 | 数据源 | 核心设计模式 | 特点 |
|------|--------|-------------|------|
| **Abu量化** | Tushare(主) + AKShare(备) | `DataHub` 统一枢纽、`robust_fetch` 重试机制、Redis 分布式缓存 | 双源容灾，自动降级 |
| **VeighNa** | TuShare/AKShare/Wind/... | `BaseDatafeed` 抽象基类 + 插件式数据源注册 | 插件式架构，扩展性强 |
| **天勤TqSdk** | 自有行情服务器(WebSocket) | `TqApi.get_quote()` 实时推送 + `wait_update()` 事件驱动 | 低延迟，专业版支持A股 |

---

## 基础工具函数

```python
import urllib.request, json, re, ssl
from datetime import datetime

# 解决SSL证书问题（参考VeighNa的ssl补丁模式）
ssl._create_default_https_context = ssl._create_unverified_context
```

---

## 接口一：大盘指数（来源：Sina Finance）

**参考来源**：Abu量化使用新浪财经免费接口作为实时行情来源之一

```bash
# 获取指数行情
curl -s 'https://hq.sinajs.cn/list=s_sh000001,s_sz399001,s_sz399006,s_sh000688' \
  -H 'Referer: https://finance.sina.com.cn'

# 返回格式（CSV）：
# 上证指数,4093.6090,1.7173,0.04,4496086,107154324
# 字段：名称,最新价,涨跌额,涨跌幅%,成交量(万手),成交额(万元)
```

**指数代码表**：
| 代码 | 名称 |
|------|------|
| s_sh000001 | 上证指数 |
| s_sz399001 | 深证成指 |
| s_sz399006 | 创业板指 |
| s_sh000688 | 科创50 |
| s_sh000016 | 上证50 |
| s_sh000300 | 沪深300 |
| s_sh000905 | 中证500 |
| s_sh000852 | 中证1000 |

---

## 接口二：板块/概念行情（来源：EastMoney Push API）

**参考来源**：AKShare 的 `stock_zh_a_spot_em` 底层即调用东方财富API

```bash
# 行业板块TOP
curl -s 'https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=10&po=1&np=1&ut=bd1d9ddb04089700cf9c27f6f7426281&fltt=2&invt=2&fid=f3&fs=m:90+t:2&fields=f14,f2,f3,f4'

# 概念板块TOP
curl -s 'https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=10&po=1&np=1&ut=bd1d9ddb04089700cf9c27f6f7426281&fltt=2&invt=2&fid=f3&fs=m:90+t:3&fields=f14,f2,f3,f4'
```

**返回字段**：`f14`=名称, `f2`=最新价, `f3`=涨跌幅%, `f4`=涨跌额

---

## 接口三：个股行情与涨跌家数（来源：EastMoney Push API）

**参考来源**：VeighNa 的 `datafeed.py` 插件式数据源注册模式

```bash
# 全市场个股行情（含涨跌幅、成交量、成交额等）
curl -s 'https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=5000&po=1&np=1&ut=bd1d9ddb04089700cf9c27f6f7426281&fltt=2&invt=2&fid=f3&fs=m:0+t:6,m:0+t:80,m:1+t:2,m:1+t:23&fields=f12,f14,f2,f3,f4,f20,f15,f16,f8'

# 字段说明
# f12=股票代码, f14=名称, f2=最新价, f3=涨跌幅%, f4=涨跌额
# f20=成交额, f15=买一量, f16=买一价(涨停价), f8=换手率%
```

---

## 接口四：个股日K线数据（来源：EastMoney）

**参考来源**：AKShare 的 `stock_zh_a_hist` 接口实现

```bash
# 获取单只股票日K线
curl -s 'https://push2his.eastmoney.com/api/qt/stock/kline/get?secid=0.002141&fields1=f1,f2,f3,f4,f5,f6&fields2=f51,f52,f53,f54,f55,f56,f57,f58,f59,f60,f61&klt=101&fqt=1&end=20260617&lmt=120'

# secid = 市场.代码（0=深交所, 1=上交所）
# klt=101 日线, fqt=1 前复权
# 返回：日期,开盘,收盘,最高,最低,成交量,成交额,振幅,涨跌幅%,涨跌额,换手率%
```

---

## 接口五：市场综合数据（来源：新浪+东方财富组合）

**参考 Abu 的 DataHub 模式**：主源失败自动降级

```python
import subprocess, json

def get_market_snapshot():
    """获取市场快照 — 统一入口，多源容灾"""
    result = {"indices": {}, "breadth": {}, "sectors": {}}
    
    # 1. 大盘指数（Sina）
    raw = subprocess.check_output([
        'curl', '-s', 
        'https://hq.sinajs.cn/list=s_sh000001,s_sz399001,s_sz399006,s_sh000688',
        '-H', 'Referer: https://finance.sina.com.cn'
    ]).decode('gbk', errors='ignore')
    
    for line in raw.strip().split('\n'):
        m = re.search(r'="([^"]+)"', line)
        if m:
            parts = m.group(1).split(',')
            result["indices"][parts[0]] = {
                "price": float(parts[1]),
                "pct": float(parts[3]),
                "volume": parts[4],
                "amount": parts[5]
            }
    
    # 2. 板块/涨跌统计（EastMoney）
    try:
        raw2 = subprocess.check_output([
            'curl', '-s',
            'https://push2.eastmoney.com/api/qt/clist/get?pn=1&pz=10&po=1&np=1&ut=bd1d9ddb04089700cf9c27f6f7426281&fltt=2&invt=2&fid=f3&fs=m:90+t:2&fields=f14,f2,f3,f4'
        ])
        result["sectors"] = json.loads(raw2)
    except:
        result["sectors"] = {"error": "secondary source failed"}
    
    return result
```

---

## 与三大框架的对应关系

| 本 Skill 的接口 | Abu量化中的对应 | VeighNa中的对应 | 天勤TqSdk中的对应 |
|----------------|-----------------|-----------------|-------------------|
| Sina指数API | `abu.data_source.sina` | n/a（侧重国内源） | n/a（专业版自有源） |
| EastMoney板块API | AKShare（底层同源） | `Datafeed` 插件 | n/a |
| EastMoney个股行情 | `DataHub.get_stock_data()` | `BaseDatafeed.query_bar()` | `TqApi.get_quote()` |
| 统一容灾DataHub | 核心模式 `DataHub` | `BaseDatafeed` + 多实现 | 付费专业版 |

---

## 安装依赖

本项目无需安装任何额外依赖（使用系统自带的 `curl`, `python3`），数据全部通过 HTTP API 获取。如需更专业的分析，可安装：

```bash
# Abu量化风格 - Tushare + AKShare
pip install akshare tushare pandas

# VeighNa风格
pip install vnpy

# 天勤风格
pip install tqsdk
```

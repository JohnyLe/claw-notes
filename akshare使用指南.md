# akshare-financial-data 使用指南

> 整理自 SiYuan「python工具集」中的碎片化安装随记《akshare-financial-data 技能安装记录》，并结合 2026-09-06 本地实测校正了其中已过时 / 错误的接口写法。

## 一、这是什么

[akshare](https://github.com/akfamily/akshare) 是一个开源 Python 财经数据接口库（GitHub `akfamily/akshare`，22.4k★，本人已 star）。它封装了股票、基金、期货、债券、外汇、宏观等大量免费数据源，是日常财经快讯 / 数据抓取类脚本的底层数据源之一。

## 二、安装与环境（2026-09-06 实测）

- **版本**：`akshare==1.18.94`（已安装并验证）
- **Python**：系统 Python 3.11
- **重装命令**：`pip install akshare==1.18.94`
- ⚠️ 未装在 WorkBuddy 托管 venv（托管 venv 当前不完整、无 akshare），脚本直接调用系统 Python 3.11。

## 三、已落地为 WorkBuddy 技能

- 技能目录：`~/.workbuddy/skills/akshare-financial-data/`
- 原安装随记中记录的技能封装此前在磁盘上丢失，已据笔记线索重建。
- 调用方式：技能脚本直接调用系统 Python 3.11 的 akshare。

## 四、数据缓存约定（5 个 json）

原随记约定把结果落本地缓存便于复用：

`stock-data.json` / `fund-data.json` / `futures-data.json` / `macro-data.json` / `bond-data.json`

## 五、常用接口（⚠️ 已按 1.18.94 实测校正）

### 债券（✅ 本机实测可用）

```python
import akshare as ak
df = ak.bond_china_yield(start_date="20240101", end_date="20240131")
# 返回列：曲线名称 / 日期 / 3月 / 6月 / 1年 / 3年 / 5年 / 7年 / 10年 / 30年 ...
```

> ❌ **旧笔记 `symbol=` 参数已失效**（1.18.94 实测 `TypeError: unexpected keyword argument 'symbol'`）。
> 正确签名：`bond_china_yield(start_date: str = '20200204', end_date: str = '20210124')`。

### 股票历史 K 线

```python
df = ak.stock_zh_a_hist(symbol="000001", period="daily",
                        start_date="20240901", end_date="20240910", adjust="")
```

### 股票实时全市场（⚠️ 见可靠性红区）

```python
df = ak.stock_zh_a_spot_em()   # East Money 端点，易限流
```

### 其他

- 基金：`fund_*`、期货：`futures_*`、宏观：`macro_*` 系列。

## 六、可靠性红区（必读）

1. **East Money 类接口间歇性 `RemoteDisconnected: Remote end closed connection without response`**
   本机实测 `stock_zh_a_spot_em()` 与 `stock_zh_a_hist()` 均触发 → 属服务端限流 / 重置，**非代码错误**。
   对策：加重试（try/except + sleep 退避）、降频、或换新浪 / 同花顺源接口。
2. **接口签名随版本频繁变动** → 调用前务必 `inspect.signature(ak.xxx)` 核对，**切勿照搬旧笔记 / 旧文章**。
3. **强网络依赖**，离线不可用。

## 七、与本项目其他数据源的关系

| 源 | 特点 | 角色 |
|---|---|---|
| **akshare** | 广覆盖、实时 / 历史，稳定性参差 | 取"活数据" |
| 本地 MySQL `wind`（`stock_sw_mapping`） | 已清洗申万映射，稳定只读 | 取"稳数据" |
| 通达信 vipdoc | 本地行情，只读 | 取"本地盘后数据" |

三者互补：akshare 拉实时广度，wind / vipdoc 提供已清洗的稳定底表。

## 八、典型用法示例

```python
import akshare as ak, time

def safe_fetch(fn, retries=3, delay=2):
    for i in range(retries):
        try:
            return fn()
        except Exception as e:
            if i == retries - 1:
                raise
            time.sleep(delay * (i + 1))   # 退避重试，规避 East Money 限流

# 债券收益率（稳）
bond = safe_fetch(lambda: ak.bond_china_yield("20240101", "20240131"))

# 单股历史（如触发 Remote end closed，会自动重试）
hist = safe_fetch(lambda: ak.stock_zh_a_hist(
    symbol="000001", period="daily",
    start_date="20240901", end_date="20240910", adjust=""))
```

---

*本文档采用 CC BY-NC 4.0 许可（署名 · 非商业），详见仓库根目录 [LICENSE](../LICENSE)。*

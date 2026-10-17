# FE5209 Volatility Rotation 策略

NUS RMI · FE5209 Financial Econometrics 小组项目
截止日期：**2026-10-20**

## 1. 策略一句话

每天美股收盘后，看 SPY 是否站上 120 日均线（大方向），再看各 ETF 的 10 日 RSI（短期涨跌力度），按决策树选出下一交易日要持有的 ETF，T+1 开盘执行。

| 角色 | 标的 |
|---|---|
| 信号来源 | SPY、QQQ、SQQQ（只用来算信号） |
| 正向杠杆 | TQQQ、TECL、SPXL |
| 反向杠杆 | TECS |
| 波动率 | UVXY |
| 防御 | BSV |
| 新加坡资产 | ES3.SI（SPDR STI ETF）+ A35.SI（ABF Singapore Bond Index Fund）：GARCH 波动率目标决定 ES3 仓位，其余持有 A35（`10_sti_garch`） |

三个研究版本：

| 版本 | 文件 | 说明 |
|---|---|---|
| 原策略（基线） | `01_baseline` | 单组参数，每天持有 1 只 ETF，无止损止盈 |
| 止损止盈版 | `02`–`06` | 36 组止损/止盈参数，样本内按邻域平均 Sharpe 选参 |
| **参数投票版（拟作为最终策略）** | `09_parameter_ensemble` | 7 个参数共 6336 组，每组每天投 1 票，得票比例 = 仓位 |
| 新加坡部分 | `10_sti_garch` | GARCH 预测 ES3 明日波动，ES3 仓位 = min(目标波动 / 预测波动, 1)，其余 1 − w 持有 A35 债券 |

## 2. 怎么运行

1. 用 PyCharm 或 VS Code 打开整个文件夹，Python 3.10 以上。
2. 安装依赖：`python -m pip install -r requirements.txt`
3. 打开 `pipeline/07_run_pipeline.ipynb`，点击 **Run All**，即可跑完 01–06 全部阶段。
4. 参数投票版单独打开 `pipeline/09_parameter_ensemble.ipynb`，点击 Run All（6336 组，约需几十秒）。
5. 新加坡部分打开 `pipeline/10_sti_garch.ipynb`：**第一次**把最后一格 `DOWNLOAD` 改成 `True` 运行（下载 ES3.SI、A35.SI 与 USD/SGD 汇率到 `data/raw/sti_daily.csv`），之后改回 `False`，再 Run All。
6. 更新数据：先 `pip install yfinance`，打开 `pipeline/08_data_update.ipynb`，把 `DOWNLOAD` 改成 `True` 后 Run All。新数据保存为 `data/raw/etf_daily_download.csv`，检查无误后再替换 `data/raw/etf_daily.csv`。

成交假设：T 日收盘出信号、T+1 开盘成交；成本按 `00_engine` 的 `ONE_WAY_BPS` 逐标的扣（如 ES3 10 bps、A35 10 bps、TECL 35 bps），全项目统一；空仓收益为 0。止损止盈用日线 High/Low 判断，跳空按开盘价成交，同日同时触及止损和止盈时按止损先发生处理。

## 3. 目录结构

```text
Volumn-Rotation-Me/
├── README.md                     本文件
├── requirements.txt              依赖
├── .gitignore
│
├── data/
│   ├── raw/etf_daily.csv         美股 ETF 原始数据（yfinance，auto_adjust=False），不要手动改
│   ├── raw/sti_daily.csv         ES3.SI + A35.SI 日线 + USD/SGD 汇率（由 10 下载）
│   └── submission/               【待生成】提交用：资产清单 + 每日收盘价
│
├── pipeline/                     唯一的代码目录
│   ├── 00_engine.ipynb           公共函数：读数据、复权、Wilder RSI、决策树、回测、指标、选参
│   ├── 01_baseline.ipynb         原策略 vs 买入持有 SPY / TQQQ
│   ├── 02_is_grid.ipynb          样本内（2012-08 ~ 2019-12）止损止盈网格
│   ├── 03_oos2020_walkforward.ipynb  冻结参数的 OOS（2020 起）+ 逐年 walk-forward
│   ├── 04_oos2024.ipynb          OOS 改为 2024 起（看过数据，并非盲测）
│   ├── 05_live_0915.ipynb        模拟 2026-09-15 起实盘
│   ├── 06_live_oct.ipynb         10 月实盘窗口
│   ├── 07_run_pipeline.ipynb     ★ 总入口：Run All 跑 01–06
│   ├── 08_data_update.ipynb      下载 / 更新数据
│   ├── 09_parameter_ensemble.ipynb  ★ 6336 组参数投票版
│   ├── 10_sti_garch.ipynb       ★ 新加坡部分：ES3 GARCH 波动率目标 + A35 债券（导出 sti_weights_history.csv）
│   ├── 11_sti_sleeve.ipynb      新加坡插槽接口：读取 10 号仓位，与 09 合成 90% + 10%（ES3 + A35）
│   └── 12_live_oct_09.ipynb     10 月实盘窗口成交账本（09 + 新加坡插槽）
│
├── results/                      【待生成】全部由 pipeline 自动输出
│   ├── tables/                   指标表 CSV
│   ├── figures/                  图（PNG）
│   └── signals/                  每日目标持仓 / 权重
│
├── report/
│   ├── strategy_explainer.docx   策略通俗说明
│   ├── slides/                   【待做】presentation
│   └── contribution.md           【待填】分工表
│
└── archive/                      旧版代码，只作记录，不再修改
    ├── notebook_v0/              原 notebook/ 文件夹
    └── 使用说明_v0.md
```

## 4. 数据如何在代码里流动

```text
data/raw/etf_daily.csv
  表头 3 行：Price（字段）/ Ticker / Date
  字段：Adj Close, Close, High, Low, Open, Volume，每个字段 × 9 只 ETF
      │
      ▼ prep()：按 Adj Close / Close 复权 Open/High/Low
O, H, L, C        DataFrame，行 = 交易日，列 = 9 只 ETF
      │
      ▼ signals(C, p)：SPY 120 日均线 + 10 日 Wilder RSI → 决策树
tg                Series：日期 → 当天收盘选出的 ETF
      │                                   │
      ▼ backtest(..., sl, tp)              ▼ 09：6336 组参数各投 1 票
equity + trades                      weights：DataFrame（日期 × ETF 权重）
      │                                   │
      ▼ metrics()                          ▼ backtest_weighted_portfolio()
{CAGR, Vol, Sharpe, Sortino, MDD, Calmar} → results 字典
```

## 5. 目前结果（数据截至 2026-10-07，含交易成本）

| 方案 | 区间 | CAGR | Sharpe | 最大回撤 |
|---|---|---|---|---|
| 原策略 | IS 2012-08 ~ 2019 | 44.6% | 0.99 | -73.8% |
| 原策略 | OOS 2020+ | 158.4% | 1.66 | -50.8% |
| 原策略 | OOS 剔除 2020 | 93.9% | 1.34 | -50.8% |
| 止损止盈版（固定 12% / 15%） | OOS 2020+ | 96.7% | 1.54 | -52.2% |
| **参数投票版** | IS | 53.2% | 1.19 | -58.2% |
| **参数投票版** | OOS 2020+ | 123.7% | 1.65 | -47.1% |
| **参数投票版** | OOS 剔除 2020 | 66.0% | 1.21 | -47.1% |
| 买入持有 SPY | OOS 2020+ | 15.4% | 0.82 | -33.7% |
| 买入持有 TQQQ | OOS 2020+ | 35.3% | 0.79 | -81.7% |

---

## 6. ⚠️ 还缺什么（截止 10/20）

### 🔴 必须完成

- [ ] **加入新加坡资产**：老师要求至少一个新加坡资产。
  - ✅ 已写好 `10_sti_garch.ipynb`：ES3.SI 的 GARCH 波动率目标策略（ARCH 检验 → 模型选择 → 残差诊断 → 预测评估 / DM 检验 → Mincer-Zarnowitz 回归 → 策略回测与 bootstrap）。已用替身数据测试通过。
  - ✅ 已用真实数据跑出结果（数据截至 2026-10-12），见 `results/tables/sti_*.csv`、`results/figures/sti_*.png`。
    - 结论：BIC 选中 GJR-GARCH(1,1)-t。样本外波动预测显著优于 EWMA 和 20 日滚动（DM 检验），Mincer-Zarnowitz 不拒绝无偏。
    - 策略在 OOS 的 Sharpe 与买入持有无显著差异（0.91 对 0.88，bootstrap p≈0.84），但波动更低（12.1% 对 15.4%），最大回撤更浅（-23% 对 -37%）。
    - 数据注意：yfinance 缺 2026-10-13 一天；2012–2017 年零收益日占约 20%。
    - A35 早年成交清淡、有错误报价：10 号按规则清洗（单日 |r|>2.5% 且次日反向回去一半以上 → 用前一日价格），共 19 天，可用 `CLEAN_BOND=False` 关闭。
  - ✅ 剩余资金由现金改为 **A35.SI 债券**；新增对照组「固定比例 ES3/A35」「GARCH + 现金」「买入持有 A35」，新增 `sti_bond_stats`（A35 波动、股债相关性）。
  - ⏳ 待办：
    - **重新下载**（10 号最后一格 `DOWNLOAD=True`）以获得 A35 数据，再用真实数据跑出 A35 版本结果；
    - 确定 ES3 部分在总组合中的比例（暂定 10%）；
    - 把「美股轮动 + ES3」合成一个总组合并回测：两个市场交易日不同，缺失日按前值处理。
- [ ] **提交用 CSV**（`data/submission/`）：
  - `asset_list.csv`：ticker、名称、类型、交易所、币种；
  - `daily_close.csv`：一行一个日期、一列一个 ticker 的每日收盘价。
  - 现在的 `etf_daily.csv` 有 3 行表头，不能直接提交。
- [ ] **Presentation**（`report/slides/`）。
- [ ] **分工表**（`report/contribution.md`）：每人写清楚自己的贡献。

### 🟠 应该完成

- [ ] **确定最终策略**：拟用 09 参数投票版，原策略和止损止盈版作为对比。
- [ ] **自动更新**：项目要求策略能自动更新。要做：
  - 一键流程「08 下载 → 09 计算 → 输出 `results/signals/latest_signal.csv`」，写明明天每只 ETF 的仓位；
  - 视情况再做定时运行。
- [ ] **更新数据到最新**：当前数据只到 2026-10-07，10 月实盘窗口要到 10/24，结果显示「窗口未完成」。
- [ ] **结果落盘**：01–09 的表和图目前只显示在 notebook 里，要用 `to_csv` / `savefig` 存到 `results/`，方便做 PPT。

### 🟡 计量与稳健性（加分项，也是可能被问到的地方）

- [ ] **2020 年依赖**：原策略 2020 年一年收益约 13.9 倍。要报告剔除 2020 的结果，并解释原因：疫情期 UVXY / 反向杠杆赚到极端行情。
- [ ] **统计检验**：
  - Sharpe 的 bootstrap 置信区间；
  - 多参数下的 Deflated Sharpe Ratio；
  - 策略收益对 SPY 回归（alpha / beta），这是本课回归章节可以直接用上的内容。
- [ ] **样本内不"干净"**：决策树阈值（81 / 80 / 30 / 74 / 84 / 34）来自现成策略，原作者很可能看过全部历史。要注明出处，并说明 2012–2019 并非真正的样本内。
- [ ] **止损止盈结论**：35 组参数里只有 3 组在 OOS 超过原策略。这是有效结论，在报告中正面解释。
- [ ] **策略说明文档补漏**：
  - 产品表漏了 SQQQ（只用作信号）；
  - 弱势分支第三关写得太模糊，应补完整决策树（见 `00_engine.ipynb` 的 `signals()`）。

## 7. 本文件夹相对原仓库的改动

整理自 `drmfslx1234567/fe5209_vol_rotation`，原文件夹未做任何修改。

- `notebook/` 旧版移入 `archive/notebook_v0/`，只保留 `pipeline/` 一套代码。
- `data/etf_daily.csv` → `data/raw/etf_daily.csv`。`00_engine` 的自动寻路已同步，旧位置也仍能找到。
- `08_data_collection_optional` → `08_data_update`，默认输出改到 `data/raw/`。
- `09_parameter_ensemble_2` → `09_parameter_ensemble`。
- `requirements.txt` 移到根目录。`README.txt` 和 `使用说明.md` 已归档，由本文件替代。
- 策略、参数、回测规则、成本假设均未改动。已在新结构下重跑 07（01–06 全部阶段）和 09，结果与原仓库一致。

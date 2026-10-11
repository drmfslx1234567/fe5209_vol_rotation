运行方式：用 Jupyter / VS Code 打开 notebook，按编号顺序运行即可（pip install pandas numpy matplotlib jupyter）。
etf_daily.csv 与 notebook 放同一文件夹。每个 notebook 开头的 `%run 00_engine.ipynb` 会载入引擎和数据，所以 00_engine.ipynb 必须在同一文件夹。

00_engine          数据读取、RSI、策略决策树、带止损止盈回测引擎、指标、选参函数（被其他 notebook 调用）
01_baseline        无止损基线 + 买入持有基准（IS/OOS）
02_is_grid         样本内(2012-08~2019-12)止损/止盈网格
03_oos2020_walkforward  冻结参数的 OOS(2020起) + walk-forward
04_oos2024         OOS 改为 2024-01 起
05_live_0915       假设 2026-09-15 开始实盘
06_live_oct        10月实盘窗口：改 LIVE_START / LIVE_END 即可；10/24 后重新下载CSV再从头运行

数据下载：
import yfinance as yf
yf.download(['SPY','QQQ','TQQQ','UVXY','TECL','SPXL','SQQQ','TECS','BSV'],start='2011-10-01',auto_adjust=False).to_csv('etf_daily.csv')

假设：T日收盘出信号、T+1开盘成交；成本5bps(UVXY 10bps)；空仓收益0；止损止盈用日线High/Low，跳空按开盘价，同日同时触及按止损先发生。

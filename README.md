# MO Derivative-Skew Strategy 8.14

本项目使用 MO 期权构造以 Derivative Skew 暴露为主的组合，并用同期限 IM 期货对冲解析 Forward Delta。项目包含数据清洗、Forward 与 Repo 构造、隐含波动率拟合、选券与仓位优化、手续费和保证金计算，以及多种 PnL 归因。

## 默认策略口径

- 波动率模型：`QUADRATIC`
- Skew：`DERIVATIVE`
- Curvature：固定使用 ATM 处二阶导数
- 期权范围：仅 OTM
- 默认要求候选期权的权利金金额/卖方单张保证金不低于 10%
- 每个候选期限最多抽取 9 只期权
- 每天最多比较最近 3 个有效期限，最终只持有一个期限
- 6 月目标 Skew exposure：`+10000`
- 7 月目标 Skew exposure：`-10000`
- 默认不在优化中对冲 Theta
- 期权仓位为整数张，IM 对冲仓位为整数手
- MO 合约乘数：100
- IM 合约乘数：200

默认模型及输出标签为：

```text
QUADRATIC_SKEW-DERIVATIVE_CURVATURE-DERIVATIVE
```

全部有效参数统一设置在 `00_config.ipynb`。

权利金/保证金过滤由以下接口控制：

```python
STRATEGY_USE_PREMIUM_MARGIN_FILTER = True
STRATEGY_MIN_PREMIUM_MARGIN_RATIO = 0.10
```

启用时，对每个候选 Call/Put 计算

```text
(100 × 期权价格) / 卖方单张保证金
```

其中卖方单张保证金为：

```text
100 × 期权价格
+ max(0.12 × 100 × 标的价格 - 100 × 虚值额,
      0.06 × 100 × 标的价格)
```

Call 的虚值额为 `max(行权价 - 标的价格, 0)`，Put 的虚值额为
`max(标的价格 - 行权价, 0)`。代码只保留比例大于等于阈值的期权。
该筛选在成交量、OTM、`|k|` 和模型 IV 检查之后、候选数量检查及仓位优化之前执行；
因此无论最终优化方向是多头还是空头，候选期权都先使用同一卖方保证金公式检查。

## 项目文件

1. `00_config.ipynb`：项目路径及 01--06 使用的全局参数。
2. `01_data_processing.ipynb`：读取并清洗指数、IM 期货和 MO 期权数据。
3. `02_basic_functions.ipynb`：合约代码解析、到期日、Black--76 和基础 Greeks 函数。
4. `03_repo_forward.ipynb`：主力期货、Repo、各期限 Forward 及期权--Forward 面板。
5. `04_volatility_model.ipynb`：隐含波动率反解及 Quadratic/SVI 曲面拟合。
6. `05_skew_curvature.ipynb`：Skew 期限结构和相关图片。
7. `06_skew_strategy.ipynb`：选券、仓位优化、整数化、IM 对冲、手续费、保证金、资本和 PnL 归因。
8. `run_all.ipynb`：在同一内核中完整运行以上模块。

由于 `01_data_processing.ipynb` 使用 `02_basic_functions.ipynb` 中的代码解析和期限函数，实际依赖顺序为：

```text
00 → 02 → 01 → 03 → 04 → 05 → 06
```

## 数据路径

原始数据位于项目目录内的 `数据/` 文件夹：

```text
skew_strategy8.14/
├── 数据/
│   ├── 标的_2026-06~07.csv
│   └── MO_2026-06~07.csv
├── 00_config.ipynb
├── ...
└── run_all.ipynb
```

`00_config.ipynb` 使用相对于项目目录的路径：

```python
UNDERLYING_DATA_PATH = Path('数据/标的_2026-06~07.csv')
FUTURE_DATA_PATH = UNDERLYING_DATA_PATH
OPTION_DATA_PATH = Path('数据/MO_2026-06~07.csv')
```

必须从项目根目录 `skew_strategy8.14/` 启动并运行 Notebook，使 `Path.cwd()` 指向本项目。项目不再依赖 `/Users/mac/Desktop/实习/数据/` 等项目外的绝对路径。

## 运行方法

推荐在项目根目录打开并运行 `run_all.ipynb`。也可以在同一个 Jupyter 内核中依次运行：

```text
00_config.ipynb
02_basic_functions.ipynb
01_data_processing.ipynb
03_repo_forward.ipynb
04_volatility_model.ipynb
05_skew_curvature.ipynb
06_skew_strategy.ipynb
```

## PnL 归因

### Traditional Taylor（主要归因）

策略研究主要参考 `traditional_taylor_attribution.csv`，其中包括：

- Forward Delta
- Smile Roll
- Gamma
- ATM Vol
- Skew
- Curvature
- Theta
- Rho
- ATM-vol Vanna
- ATM-vol Volga
- Model Residual、Market Noise、Total Residual
- 手续费前 Market PnL、手续费后 Actual PnL 和 Fee PnL

### 完整 T1/T2

- `T1`：七因子的一阶 Taylor PnL。
- `T2`：七个一阶项、七个纯二阶项和 21 个二阶交叉项构成的完整二阶 Taylor PnL。
- `complete_taylor_pnl_daily_and_cumulative.csv`：将每个一阶项、纯二阶项和交叉项分别输出，不把高阶项合并回对应的一阶因子。
- T2 累计比较图为了按七个因子展示，会把每个交叉项平均分配给相关的两个因子；完整 CSV 中的交叉项仍保持独立。

### 七因子 Shapley

全项目使用统一七因子状态：

```text
DELTA, SMILE_ROLL, ATM, SKEW, CURV, TAU, R
```

七因子 Shapley 使用全重估归因；其 Model Residual 仅应为浮点误差。

## 输出目录

所有运行结果写入项目内的 `outputs/`，不会读取或覆盖其他项目的输出。主要目录为：

```text
outputs/01_processed_data/
outputs/03_repo_forward/
outputs/04_volatility_model_QUADRATIC_SKEW-DERIVATIVE_CURVATURE-DERIVATIVE/
outputs/05_skew_curvature_QUADRATIC_SKEW-DERIVATIVE_CURVATURE-DERIVATIVE/
outputs/06_skew_strategy_QUADRATIC_SKEW-DERIVATIVE_CURVATURE-DERIVATIVE/
```

06 的主要结果包括：

- `positions.csv`：每日持有的期权张数和 IM 期货手数。
- `shapley_attribution.csv`：七因子 Shapley PnL 和账户 PnL。
- `traditional_taylor_attribution.csv`：Traditional Taylor PnL 归因。
- `traditional_taylor_exposures.csv`：Traditional Taylor 使用的组合 model-Greek exposures。
- `complete_taylor_pnl_daily_and_cumulative.csv`：完整一阶、二阶及交叉项的逐日和累计 PnL。
- `full_first_second_order_exposures.csv`：完整数值梯度和 Hessian exposures。
- `t1_t2_shapley_pnl_comparison.csv`：T1、T2 和 Shapley 的逐日比较。
- `capital_requirement_daily.csv`：期权与期货保证金、权利金成本、开仓手续费和资金需求代理。
- `figures/`：累计 PnL、归因、exposure 和 Skew 诊断图片。

## 注意事项

- 回测使用同一日收盘数据完成曲面拟合、选券，并假设按收盘价成交。
- 选券时会使用“下一交易日是否仍有该期限、曲面和期权代码记录”的可用性信息，但不会使用下一日的具体价格或曲面数值进行仓位优化。
- 连续优化解经过 Skew 目标缩放、边界截断和整数化后不会再次进行整数规划求解，因此最终风险目标只会近似满足。
- 期权保证金逐腿相加，没有考虑交易所组合保证金抵扣。
- `R_recommended` 是资金压力代理，不是严格的自融资、时间加权收益率或内部收益率。

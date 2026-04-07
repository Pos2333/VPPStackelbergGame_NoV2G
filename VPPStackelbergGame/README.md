# VPPStackelbergGame

## 文件目录结构

```
/VPPStackelbergGame
│
├── /timeseries_data             # [文件夹] 存放所有24小时时序数据 (CSV)
│   ├── elec_market_prices.csv   # 电力市场分时价格
│   ├── load_profiles.csv        # 负荷曲线
│   └── pv_generation.csv        # 光伏出力预测 (均值/方差)
│
├── config_params.py             # [文件] 存放静态标量参数 (Python 字典格式，便于读取)
├── main_solver.py               # [文件] 主求解程序
└── README.md                    # 这个文档
```

## 时序数据接口`timeseries_data`

所有 CSV 文件必须在第一行包含**表头**，第一列统一为 `hour` (0-23)。 **统一单位约定**：

- **功率**：kW (千瓦)
- **电价格**：CNY/kWh (元/千瓦时)
- **碳价格**：CNY/kg（元/千克）
- **碳排放**：kgCO2/kWh

### 电力市场分时价格`elec_market_prices.csv`

主网面向不同用户的售电价，以及VPP与主网的交互价格（购电价由售电价×浮动系数决策变量计算）。

| 表头                    | 对应数学符号               | 说明                              |
| :---------------------- | :------------------------- | :-------------------------------- |
| `Hour`                  | $t$                        | 0-23                              |
| `Grid_Price_Res_Summer` | $\rho_{grid,t}^{R}$ (夏)   | 居民阶梯电价（夏季），三档        |
| `Grid_Price_Res_Other`  | $\rho_{grid,t}^{R}$ (非夏) | 居民阶梯电价（非夏），三档        |
| `Grid_Price_Comm`       | $\rho_{grid,t}^{C}$        | 工商业分时电价                    |
| `Grid_Sell_Price`       | $\rho_{grid,t}^{sell}$     | VPP余电上网价（通常固定或低谷价） |

### 负荷`load_profiles.csv`

包含基准负荷的分时数据。

| 表头             | 对应数学符号          | 说明                       |
| :--------------- | :-------------------- | :------------------------- |
| `Hour`           | $t$                   | 0-23                       |
| `Load_Res_Base`  | $P_{load,R,t}^{base}$ | 居民基准负荷（四季一致）   |
| `Load_Comm_Base` | $P_{load,C,t}^{base}$ | 工商业基准负荷（四季一致） |
| `Load_PV_Base`   | $P_{pv,t}^{base}$     | 光伏产消者自身的基准负荷   |

### 光伏出力`pv_generation.csv`

包含光伏预测出力的**均值**与**标准差（标准差暂时不考虑）**。四季数据列在同一表中，代码运行时按需读取。

| 表头                 | 对应数学符号      | 说明                   |
| :------------------- | :---------------- | :--------------------- |
| `Hour`               | $t$               | 0-23                   |
| `PV_Gen_Mean_Spring` | $P_{pv,t}^{mean}$ | 春季光伏出力均值预测   |
| `PV_Gen_Mean_Summer` | $P_{pv,t}^{mean}$ | 夏季光伏出力均值预测   |
| `PV_Gen_Mean_Autumn` | $P_{pv,t}^{mean}$ | 秋季光伏出力均值预测   |
| `PV_Gen_Mean_Winter` | $P_{pv,t}^{mean}$ | 冬季光伏出力均值预测   |
| `PV_Gen_Std_Spring`  | $P_{pv,t}^{std}$  | 春季光伏出力标准差预测 |
| `PV_Gen_Std_Summer`  | $P_{pv,t}^{std}$  | 夏季光伏出力标准差预测 |
| `PV_Gen_Std_Autumn`  | $P_{pv,t}^{std}$  | 秋季光伏出力标准差预测 |
| `PV_Gen_Std_Winter`  | $P_{pv,t}^{std}$  | 冬季光伏出力标准差预测 |

## 静态参数接口`config_params.py`

将所有常数、系数、边界条件写入一个 .py 文件中。下方变量数值为示例值。

```python
# config_params.py

# =========================
# 全局基础设置
# =========================
N_GAME = 200          # 最大博弈次数
S = 50                # 光伏蒙特卡洛模拟次数
T = 24                # 调度周期 (h)
DT = 1                # 时间步长 (h)
CURRENT_SEASON = 'Summer' # 当前模拟季节: 'Spring', 'Summer', 'Autumn', 'Winter'

# =========================
# 博弈决策变量边界参数
# =========================
# 居民电价浮动系数 delta_R
BOUNDS_DELTA_R = (0.9, 1.1)

# 工商业电价浮动系数 delta_C
BOUNDS_DELTA_C = (0.7, 1.4)

# 激励补偿 gamma_inc (元/kWh)
BOUNDS_GAMMA_INC = (0.0, 1.2) # 上限略高于 max(omega)

# =========================
# 碳交易参数 (单位: kg, 元/kg)
# =========================
# 碳排放函数系数 (E = alpha*P^2 + beta*P + gamma)
EMI_ALPHA = 1e-7      # 示例值，需根据实际拟合
EMI_BETA  = 0.85      # 示例值 (kg/kWh)
EMI_GAMMA = 50.0      # 固定排放 (kg)

# 碳配额系数 (E_quota = delta * P)
EMI_QUOTA_DELTA = 0.78

# 碳市场交易价格(rho_carbon)
RHO_CARBON = 0.09  

# =========================
# 灵活火电运行参数 (单位: kW, 元)
# =========================
# 火电燃料成本系数 (C = a*P^2 + b*P + c)
GEN_A = 1e-6          # 燃料成本二次项
GEN_B = 0.25          # 燃料成本一次项 (元/kWh)
GEN_C = 100.0         # 固定损耗

# 启动成本 (元/次)
GEN_STARTUP_COST = 500.0 

# 物理约束
GEN_P_MAX = 2000.0    # 最大出力 (kW)
GEN_P_MIN = 600.0     # 最小出力 (kW, 深度调峰)
GEN_RAMP_UP = 800.0   # 上爬坡 (kW/h)
GEN_RAMP_DOWN = 800.0 # 下爬坡 (kW/h)
GEN_MIN_ON = 3        # 最小开机时间 (h)
GEN_MIN_OFF = 3       # 最小停机时间 (h)

# =========================
# 分布式光伏Prosumer参数
# =========================
# 光伏装机容量(kW)
PV_Gen_MAX = 750.0

# =========================
# 下层纯负荷用户行为特征参数
# =========================
# 居民 (Residential)
RES_PARAMS = {
    'alpha': 2.5,     # 效用函数一次项 (会在代码中根据电价自动校准，此处为初始值)
    'beta':  0.005,   # 效用函数二次项
    'omega': 0.2,     # 平移不便系数 (元/kWh)
    'epsilon': 0.2,   # 最大可平移比例 (20%)
    'P_min_pct': 0.8, # 负荷下限比例 (80% Base)
    'P_max_pct': 1.2  # 负荷上限比例 (120% Base)
}

# 工商业 (Commercial)
COMM_PARAMS = {
    'alpha': 5.0,     # 工商业对电能价值评价更高
    'beta':  0.002,   # 敏感度较低
    'omega': 0.8,     # 停工损失大，不便系数高
    'epsilon': 0.1,   # 调节能力较弱 (10%)
    'P_min_pct': 0.9,
    'P_max_pct': 1.1
}
```


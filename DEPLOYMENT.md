# 部署快速指南

## VeighNa快速部署

### 1. 环境准备

```bash
# 系统要求
- Ubuntu 22.04 LTS / Windows 11
- Python 3.11+
- 内存: 16GB+
- 存储: 500GB SSD

# 安装Python依赖
sudo apt update
sudo apt install python3.11 python3.11-venv python3.11-dev
```

### 2. 安装VeighNa

```bash
# 方式一：VeighNa Studio（推荐新手）
wget https://download.vnpy.com/veighna_studio-4.4.0.exe

# 方式二：pip安装
python3.11 -m venv vnpy_env
source vnpy_env/bin/activate
pip install vnpy

# 安装A股接口
pip install vnpy_xtp      # 中泰XTP
pip install vnpy_tora     # 华鑫奇点
pip install vnpy_emt      # 东方财富

# 安装数据服务
pip install vnpy_xt       # 迅投研
pip install vnpy_tushare  # TuShare

# 安装AI组件
pip install vnpy[alpha]
pip install lightgbm scikit-learn
```

### 3. 数据库配置

```bash
# 安装MongoDB
sudo apt install mongodb
sudo systemctl start mongodb
sudo systemctl enable mongodb

# 配置
mongo
> use quantaxis
> db.createUser({
    user: "quant",
    pwd: "password",
    roles: ["readWrite"]
  })
```

### 4. 第一个策略

```python
# simple_strategy.py
from vnpy.trader.constant import Interval
from vnpy.trader.object import BarData
from vnpy_ctastrategy import CtaTemplate

class SimpleStrategy(CtaTemplate):
    """简单的双均线策略"""
    
    # 参数
    fast_window = 10
    slow_window = 20
    
    # 变量
    fast_ma = 0.0
    slow_ma = 0.0
    
    def on_init(self):
        """初始化"""
        self.write_log("策略初始化")
        
    def on_start(self):
        """启动"""
        self.write_log("策略启动")
        
    def on_stop(self):
        """停止"""
        self.write_log("策略停止")
        
    def on_bar(self, bar: BarData):
        """K线数据"""
        # 计算均线
        # ... 策略逻辑
        
        # 下单
        if self.fast_ma > self.slow_ma:
            self.buy(bar.close_price, 100)
        elif self.fast_ma < self.slow_ma:
            self.sell(bar.close_price, 100)
```

### 5. 启动交易系统

```bash
# 启动VeighNa Trader
python run.py

# 或使用脚本
from vnpy.event import EventEngine
from vnpy.trader.engine import MainEngine
from vnpy.trader.ui import MainWindow, create_qapp
from vnpy_xtp import XtpGateway

# 创建引擎
qapp = create_qapp()
event_engine = EventEngine()
main_engine = MainEngine(event_engine)

# 添加接口
main_engine.add_gateway(XtpGateway)

# 启动界面
main_window = MainWindow(main_engine, event_engine)
main_window.showMaximized()
qapp.exec()
```

---

## QUANTAXIS快速部署

### 1. 安装

```bash
# 克隆仓库
git clone https://github.com/yutiansut/QUANTAXIS.git
cd QUANTAXIS

# 安装（包含Rust组件）
pip install -e .[rust]

# 验证
python -c "import QUANTAXIS as QA; print(QA.__version__)"
```

### 2. 数据配置

```python
# 配置MongoDB
import QUANTAXIS as QA

# 设置数据库
QA.DATABASE = QA.QAUtil.QALogs.QA_Setting.MONGO_URI
# 默认: mongodb://localhost:27017/quantaxis

# 下载历史数据
from QUANTAXIS import QA_fetch_stock_day_adv
data = QA_fetch_stock_day_adv(
    start='2020-01-01',
    end='2026-08-18',
    code=['000001', '000002']
)
```

### 3. 回测示例

```python
from QUANTAXIS.QARSBridge import QARSAccount, QARSBacktest

# 创建账户
account = QARSAccount("my_strategy", init_cash=1000000)

# 交易
account.buy("000001", 10.5, "2026-01-15", 1000)
account.sell("000001", 10.8, "2026-01-16", 500)

# 查询持仓
positions = account.get_positions()
print(positions)

# 获取账户信息
qifi = account.get_qifi()
print(f"账户权益: {qifi['accounts']['balance']}")
```

---

## Qlib快速部署

### 1. 安装

```bash
# pip安装
pip install pyqlib

# 初始化数据
python -m qlib.run.get_data qlib_data
```

### 2. 配置

```python
# 初始化Qlib
import qlib
from qlib.config import HIGH_FREQ_CONFIG

qlib.init(provider_uri="~/.qlib/qlib_data/cn_data")
```

### 3. 训练模型

```python
from qlib.workflow import R
from qlib.workflow.record_temp import SignalRecord

# 加载数据
from qlib.data.dataset import DatasetH
dataset = DatasetH(
    handler={
        "class": "Alpha158",
        "module_path": "qlib.contrib.data.handler"
    }
)

# 训练模型
from qlib.contrib.model.gbdt import LGBModel
model = LGBModel(
    loss="mse",
    feature_importance="gain"
)

with R.start("my_experiment"):
    R.log_params({
        "model": "LightGBM",
        "dataset": "Alpha158"
    })
    model.fit(dataset)
    R.save_objects(model=model)
```

---

## 常见问题

### Q1: MongoDB连接失败？
```bash
# 检查服务状态
sudo systemctl status mongodb

# 重启服务
sudo systemctl restart mongodb

# 检查端口
netstat -an | grep 27017
```

### Q2: XTP接口无法连接？
```
1. 检查券商账号权限
2. 确认服务器地址和端口
3. 检查授权文件是否有效
4. 查看日志: ~/.vnpy/logs/
```

### Q3: 数据下载失败？
```python
# 使用TuShare免费数据
import tushare as ts

# 设置Token
ts.set_token('your_token')

# 下载数据
pro = ts.pro_api()
df = pro.daily(
    ts_code='000001.SZ',
    start_date='20200101',
    end_date='20260818'
)
```

---

## 性能优化建议

### 1. 数据库优化
```bash
# MongoDB配置
# /etc/mongodb.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8
      
# 索引优化
db.bar_data.createIndex({"code": 1, "datetime": -1})
```

### 2. 内存优化
```python
# 使用polars替代pandas
import polars as pl

# 批量处理
for chunk in pd.read_csv('large_file.csv', chunksize=10000):
    process(chunk)
```

### 3. 并行计算
```python
from concurrent.futures import ProcessPoolExecutor

def backtest_stock(code):
    # 回测逻辑
    return result

with ProcessPoolExecutor(max_workers=8) as executor:
    results = list(executor.map(backtest_stock, stock_list))
```

---

**文档版本**: v1.0  
**更新日期**: 2026-08-18
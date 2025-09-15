---
title: 交通路况预测与可视化项目（TTI）
date: 2025-09-09 11:56:00
categories: 技术项目
tags: [机器学习, 数据可视化, Python]
---

# 交通路况预测与可视化项目（TTI）

本仓库包含：数据预处理 → 模型训练（LSTM）→ 预测（LGBM/LSTM/融合）→ 验证评估 → GeoJSON 导出 → Web 前端可视化 的完整流水线。

目录参考（与你当前项目一致）：
```
traffic_project/
├── data/
│   ├── processed/                  # 预处理后的 parquet & 导出的 GeoJSON
│   ├── raw/                        # 原始 csv（link.csv、speed*.csv 等）
│   └── frontend/heatmap.html       # Leaflet 地图前端
├── models/
│   ├── lgbm/                       # 可选：LGBM 模型文件
│   └── lstm/                       # LSTM 权重、曲线
├── outputs/
│   ├── validation_*                # 验证输出（指标/图表）
│   └── pred_tti_*.csv              # 预测结果（link_id, tti_pred）
└── scripts/
    ├── preprocess.py               # 预处理：原始速度数据 → parquet
    ├── train.py                    # 训练 LSTM（带 tqdm 进度条与曲线）
    ├── predict.py                  # 预测（lgbm/lstm/blend）
    ├── validate.py                 # 验证评估（RMSE/MAE/MAPE/R² + 可视化）
    └── export_geojson.py           # 导出 GeoJSON（线/点）便于前端显示
```

---

## 0. 环境准备

### 方式 A：conda（推荐）
```bash
conda create -n traffic python=3.10 -y
conda activate traffic
pip install -r requirements.txt
# 如果没有 requirements.txt，最小依赖：
# pip install numpy pandas pyarrow tqdm matplotlib torch joblib lightgbm
```

### 方式 B：纯 pip
```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

> GPU 可选：若安装了 CUDA 版 PyTorch，会自动用 GPU；否则使用 CPU。

---

## 1. 数据准备

把原始数据放到 `data/raw/`：
- `link.csv`（包含 `Link, Longitude_Start, Latitude_Start, Longitude_End, Latitude_End, Length` 等列）
- `speed*.csv`（列名中包含 `Period/Link/Speed`，Period 可以是时间或步号）

若 `Period` 是步号，需要提供起始时间 `--t0` 与步长分钟数 `--step-mins`。

---

## 2. 预处理（raw → parquet）

```bash
# Period 已是日期时间字符串
python scripts/preprocess.py --raw-dir data/raw --out data/processed/chengdu.parquet

# 若 Period 是步号，从 2018-10-01 00:00 每 5 分钟
python scripts/preprocess.py --raw-dir data/raw --out data/processed/chengdu.parquet \
    --t0 "2018-10-01 00:00" --step-mins 5
```

完成后会打印汇总：链路数、时间戳数，并在 `data/processed/` 生成 `chengdu.parquet`。

---

## 3. 训练（LSTM）

```bash
python scripts/train.py --data data/processed/chengdu.parquet \
    --seq-len 12 --horizon 1 \
    --epochs 30 --batch-size 1024 \
    --hidden 128 --layers 2 --dropout 0.2 \
    --train-ratio 0.8 --patience 10
```
输出：
- `models/lstm/model.pt`（最优权重）
- `models/lstm/metrics.csv`（每轮 train/val 指标）
- `models/lstm/curves.png`（拟合曲线）

> 日志含 tqdm 进度条；早停不激进（默认 `min_epochs=10, patience=10`）。

---

## 4. 预测（TTI）

在项目根目录执行：

### 仅 LSTM
```bash
python scripts/predict.py --mode lstm --horizon 6 --seq-len 12
```

### 仅 LightGBM
```bash
python scripts/predict.py --mode lgbm --horizon 6
# 需要 models/lgbm_tti.pkl（内含 model 与 feats）
```

### 融合（LGBM + LSTM 平均）
```bash
python scripts/predict.py --mode blend --horizon 6 --seq-len 12
```

输出：`outputs/pred_tti_<mode>_h<h>.csv`（两列：`link_id, tti_pred`）。

---

## 5. 验证评估

把预测结果与真实值在**同一目标时刻**对齐，输出指标与图表：

```bash
# 用 parquet 中的最新时刻
python scripts/validate.py \
  --pred outputs/pred_tti_lstm_h6.csv \
  --parquet data/processed/chengdu.parquet \
  --out-dir outputs/validation_lstm_h6

# 或指定目标时刻（建议做严格回测）
python scripts/validate.py \
  --pred outputs/pred_tti_blend_h6.csv \
  --parquet data/processed/chengdu.parquet \
  --target-time "2025-09-09 13:58" \
  --out-dir outputs/validation_blend_h6
```

输出包含：
- `report.json` / `report.txt`：RMSE / MAE / MAPE / R² / 相关系数
- `merged.csv`：对齐后的真值与预测
- 可视化：`scatter_true_pred.png`、`error_hist.png`、`by_bin_metrics.png`、`timeseries_samples.png`

---

## 6. 导出 GeoJSON（用于前端）

```bash
# 输出线段与中点两类 GeoJSON，并附带时间戳
python scripts/export_geojson.py --parquet data/processed/chengdu.parquet \
  --metric tti --time latest

# 也可指定具体时刻（自动选择最近的时间戳）
python scripts/export_geojson.py --parquet data/processed/chengdu.parquet \
  --metric tti --time "2025-09-10 10:00"
```
生成：
```
data/processed/links_latest.geojson
data/processed/points_latest.geojson
```

> 颜色来自 `properties.color`：TTI 低偏绿、TTI 高偏红；Speed 模式反之。

---

## 7. 地图前端查看

前端页面位于 `data/frontend/heatmap.html`。推荐在**项目根目录**开本地静态服务，以便页面能访问 `/data/processed/*.geojson`：
```bash
cd traffic_project
python -m http.server 8000
# 浏览器打开
http://localhost:8000/data/frontend/heatmap.html
```
页面左上角可设置：
- **目录**：默认 `/data/processed/` 或 `../processed/`（根据你开服务的位置）
- **Line / Point 文件名**：`links_latest.geojson`、`points_latest.geojson`
- 勾选 **线段/热力点** 图层、刷新、自适应视图。

> 如果出现 `HTTP 404`，通常是服务开在了 `data/frontend/` 子目录，无法访问上一级目录；建议在**项目根目录**启动服务，或把目录字段改成绝对路径 `/data/processed/`。

---

## 8. 常见问题（FAQ）

- **Q：`python predict.py` 报找不到文件？**
  A：`predict.py` 在 `scripts/` 目录，使用 `python scripts/predict.py ...`。

- **Q：进度条/绘图模块缺失？**
  A：`pip install tqdm matplotlib`。服务器无显示环境已在脚本中设置 `Agg` 后端。

- **Q：LSTM 用 GPU 吗？**
  A：训练/预测会自动检测 CUDA；也可用 `--device cpu/cuda` 指定。

- **Q：前端为什么全是绿色？**
  A：当前时刻大多路段 TTI 偏低（畅通），且聚合泡泡的默认样式是绿色。放大后单点会使用你的颜色；也可以在前端自定义 MarkerCluster 的配色以反映平均 TTI。

- **Q：验证为什么只看到 MAE 线，看不到 RMSE？**
  A：两者数值非常接近，RMSE 可能被 MAE 线覆盖。可把 RMSE 画成虚线或单独成图。

---

## 9. 命令速查

```bash
# 预处理
python scripts/preprocess.py --raw-dir data/raw --out data/processed/chengdu.parquet
python scripts/preprocess.py --raw-dir data/raw --out data/processed/chengdu.parquet --t0 "2018-10-01 00:00" --step-mins 5

# 训练
python scripts/train.py --data data/processed/chengdu.parquet --seq-len 12 --epochs 30

# 预测
python scripts/predict.py --mode lstm  --horizon 6 --seq-len 12
python scripts/predict.py --mode lgbm  --horizon 6
python scripts/predict.py --mode blend --horizon 6 --seq-len 12

# 验证
python scripts/validate.py --pred outputs/pred_tti_lstm_h6.csv --parquet data/processed/chengdu.parquet --out-dir outputs/validation_lstm_h6

# 导出 GeoJSON
python scripts/export_geojson.py --parquet data/processed/chengdu.parquet --metric tti --time latest

# 前端
python -m http.server 8000  # 在项目根目录
# 打开 http://localhost:8000/data/frontend/heatmap.html
```

---

## 10. 可选增强（按需）
- 导出时对 TTI 进行**分位数拉伸**，增强色彩对比；
- 前端为 MarkerCluster 定义 `iconCreateFunction`，使聚合泡泡按平均 TTI 着色；
- 增加 `Makefile`/`run_all.sh` 一键跑通全链路；
- LGBM 模型训练脚本（自动构造滞后特征与滚动统计）。

---

如需我把"分位数拉伸配色 + 自定义聚合泡泡"提交为可直接覆盖的文件，请告诉我你希望使用的默认参数（例如：TTI 上限用 95% 分位还是 90% 分位）。
# Spectrum Analyzer

基于 Fabry-Pérot 干涉的薄膜厚度反演工具。前后端分离架构：前端做交互和绘图，
后端运行核心算法（贪心/DP 匹配、Savitzky-Golay 滤波、Auto-Fit、Fine Search）。

## 在线访问

| 入口 | URL |
|---|---|
| 前端 (GitHub Pages) | https://yx-git-hub.github.io/spectrum-analyzer/ |
| 后端 (Render) | https://spectrum-backend-uhp8.onrender.com |
| 后端 API 文档 | https://spectrum-backend-uhp8.onrender.com/docs |

> Render 免费套餐空闲 15 分钟后会休眠，第一次访问需等约 30 秒冷启动。

## 数据集

| ID | 材料 | 厚度范围 (nm) | 步长 (nm) | 拟合 |
|---|---|---|---|---|
| `mos2` | MoS₂ | 85 – 1999.5 | 0.5 | ✓ |
| `moo3` | MoO₃（偏振 [100]）| 80 – 999.5 | 0.5 | ✓ |
| `mos2_moo3` | MoS₂ / MoO₃ 堆叠 | 50 – 1030 × 50 – 1030 | 20 / 20 | — (仅查看) |

双材料拟合算法尚未实现；选中双材料数据集时滑块可以扫光谱，但 Auto-Fit /
Fine Search 按钮会置灰。

## 技术栈

- **后端**：Python 3.11 / FastAPI / numpy / scipy / numba JIT / slowapi
- **前端**：原生 HTML/JS / Plotly / SheetJS
- **部署**：Render (后端) + GitHub Pages (前端)

## 项目结构

```
spectrum/
├── index.html                 # 根重定向到 frontend/spectrum_analyzer.html
├── frontend/
│   └── spectrum_analyzer.html # 单文件前端（自动检测本地/生产 API URL）
└── backend/
    ├── app/
    │   ├── main.py            # FastAPI 入口、CORS、限流
    │   ├── datasets.py        # 启动时加载 data/*.npy + *.json
    │   ├── schemas.py         # Pydantic 请求/响应模型
    │   └── algorithms/
    │       ├── smoothing.py   # Savitzky-Golay 分段滤波
    │       ├── detection.py   # scipy.find_peaks + numba 加速
    │       ├── matching.py    # 贪心 / DP 峰谷匹配
    │       └── fitting.py     # Auto-Fit (粗扫) + Fine Search (插值精搜)
    ├── data/                  # mos2.npy/json, moo3.npy/json, mos2_moo3.npy/json
    ├── tools/
    │   └── convert_csv_to_npy.py
    ├── requirements.txt
    ├── render.yaml            # Render Blueprint
    └── .python-version        # 锁定 3.11.9
```

## 本地开发

### 环境

- Python 3.11+
- 后端依赖在 conda 环境 `test` 中（见 [requirements.txt](backend/requirements.txt)）

### 起后端

```bash
cd backend
# 把 D:/anaconda/envs/test/python.exe 替换成你的 Python
D:/anaconda/envs/test/python.exe -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

- API 文档：http://127.0.0.1:8000/docs
- 健康检查：http://127.0.0.1:8000/health

### 起前端

直接双击 `frontend/spectrum_analyzer.html`（或 `index.html`）。

前端会根据 `location.hostname` 自动判断后端：
- `localhost` / `127.0.0.1` / `file://` → `http://127.0.0.1:8000`
- 其他（如 GitHub Pages）→ Render 生产 URL

如需临时把线上前端指到本地后端：在 URL 加 `?api=http://127.0.0.1:8000`。

## 工作流：日常更新与发布

```
你改代码 → git commit → git push origin master
                                ↓
                ┌───────────────┴────────────────┐
                ↓                                ↓
        Render 自动重新部署               GitHub Pages 自动重新发布
        （build ~3-5 分钟）                （生效 ~1-2 分钟）
                ↓                                ↓
        backend live                      frontend live
```

### 仅改后端
```bash
# 改 backend/ 下文件
git add backend/
git commit -m "..."
git push
# 等 Render dashboard 显示 Live
```

### 仅改前端
```bash
# 改 frontend/spectrum_analyzer.html
git add frontend/
git commit -m "..."
git push
# 等 GH Pages 生效后 Ctrl+F5 刷新浏览器
```

### 同时改前后端
两套都改 → 一次 commit + push → Render 和 GH Pages 各自更新。

> ⚠️ 部署前一定要先在本地验证后端能起来，不然 Render 会 build 失败发邮件。

## 添加新数据集

### 1. 把源 CSV 放进 `origin_data/`
（这个目录在 `.gitignore` 里，不进版本控制）

CSV 格式：
- 一行波长（首行或末行；脚本自动识别）
- 其他行：反射率 + 末尾 1 或 2 列厚度

### 2. 转成 .npy

单材料：
```bash
cd backend
python tools/convert_csv_to_npy.py \
  ../origin_data/your_file.csv \
  data NEW_ID \
  --name "Display Name" \
  --material MoX
```

双材料：
```bash
python tools/convert_csv_to_npy.py \
  ../origin_data/double_file.csv \
  data NEW_ID \
  --name "MoA on MoB" \
  --material MoA --material MoB \
  --thickness-cols 2
```

带偏振信息：再加 `--polarization "[100]"`。

### 3. 提交并推送

```bash
git add backend/data/NEW_ID.npy backend/data/NEW_ID.json
git commit -m "Add NEW_ID dataset"
git push
```

Render 重启后 `datasets.load_all()` 会自动扫到，前端下拉菜单会出现新项。

> ⚠️ 单文件不能超过 100 MB（GitHub 硬上限）；压缩后的 .npy 通常远低于此。
> 如果超了，用 Git LFS 或先降采样。

## 安全与限制

- **CORS 白名单**：后端只允许 `https://yx-git-hub.github.io` 以及 localhost /
  `file://`（`null`）发起请求。新增前端域名需改 `main.py` 的
  `_DEFAULT_ORIGINS`，或在 Render 设置 `ALLOWED_ORIGINS` 环境变量
  （逗号分隔）。
- **速率限制（按 IP）**：
  - `/datasets/{id}/full`、`/api/auto-fit`、`/api/fine-search`：**15 次/分钟**
  - 其他端点（detect/smooth/match）：不限（拖滑块会高频调用）
- **Render 免费套餐**：512 MB 内存 / 15 分钟无访问自动休眠。`/datasets/{id}/full`
  已改成流式 JSON 输出，避免 OOM。

## 算法说明（要点）

- **Auto-Fit 代价函数**：`cost = rmse + penalty × unmatched_above_700nm + penalty × (1 - matchRatio)`
- **Auto-Fit** 取 cost 最小；**Fine Search** 取 RMSE 最小。
- 差值 ≥ `rmseThreshold` 的匹配对：用 `rmseThreshold²` 计入平方和，仍算 matchedN。
- `findPeaks` 距离单位为 nm，内部按采样间距换算成点数：
  `distSamps = max(1, round(distanceNm / spacing))`
- 详细等价点参见 [CLAUDE.md](CLAUDE.md) 中"关键算法参考"小节。

## 排错

| 症状 | 原因 / 解决 |
|---|---|
| 页面状态栏卡在 "Connecting to server..." | Render 冷启动，等 30s 再刷 |
| `/api/smooth` 422 | 实验数据有 NaN 或长度不一致；前端解析器已会自动对齐 + 填充，重新加载 Excel |
| Render build 失败提示 numpy 编译超时 | 检查 `backend/.python-version` 是否还是 `3.11.9` |
| Render OOM 警告邮件 | 大概率是 `/datasets/{id}/full` 又被改回 `.tolist()`；保持流式输出 |
| 429 Too Many Requests | 触发限流，等一分钟。或在 `main.py` 调大 `HEAVY_RATE` |

## 阶段进度

完整开发历史参见 [CLAUDE.md](CLAUDE.md)。所有 6 个阶段（含 4.5 数据集扩展）
均已完工。

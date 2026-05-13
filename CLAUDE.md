# Spectrum Analyzer — 前后端分离项目

## 项目背景

本项目将一个单文件 HTML 光谱分析工具（`spectrum_analyzer.html`，~2700 行）
重构为前后端分离架构。核心目标：

1. **保护算法 IP**：把贪心匹配、Auto-Fit、Fine Search、Savitzky-Golay 等核心算法
   从前端 JS 迁移到 Python 后端，部署到云服务器后用户拿不到源码
2. **保留页面 UI**：前端 HTML 的视觉、布局、交互一行都不能变
3. **预存数据集**：200MB 的生成光谱数据库不再由用户上传，而是预先转换成
   `.npy` 格式存在服务器上。用户在前端从下拉菜单选材料即可
4. **支持多人在线使用**：前端部署到 GitHub Pages，后端部署到 Render，
   通过 HTTPS 通信

## 技术栈

- **后端**：Python 3.11+ / FastAPI / numpy / scipy / uvicorn
- **前端**：原生 HTML/JS（基于现有 `spectrum_analyzer.html`）+ Plotly + SheetJS
- **部署**：Render（后端）+ GitHub Pages（前端）
- **代码托管**：GitHub

## 当前项目文件

工作目录：`./`（即 spectrum/）

已有文件：
- `spectrum_analyzer.html` — 原始单文件应用，包含所有 UI 和算法
- `gen_data_single_MoS2_B85_E2000_step0.5.csv` — 测试用生成数据集
  （材料：MoS₂，波长范围约 B85-E2000，厚度步长 0.5 nm）
- `CLAUDE.md` — 本文件
- `spectrum_analyzer_backup.html` — 原文件备份（用户已在文件夹外另存）

## 最终目标文件结构

```
spectrum/
├── CLAUDE.md                          (本文件)
├── README.md                          (项目说明，最后写)
├── frontend/
│   └── spectrum_analyzer.html         (改造后的前端)
└── backend/
    ├── app/
    │   ├── __init__.py
    │   ├── main.py                    (FastAPI 入口)
    │   ├── datasets.py                (数据集加载 + LRU 缓存)
    │   ├── schemas.py                 (Pydantic 类型定义)
    │   └── algorithms/
    │       ├── __init__.py
    │       ├── smoothing.py
    │       ├── detection.py
    │       ├── matching.py
    │       └── fitting.py
    ├── data/                          (.gitignore 排除)
    │   ├── mos2.npy
    │   └── mos2.json
    ├── tools/
    │   └── convert_csv_to_npy.py
    ├── tests/
    │   └── test_parity.py
    ├── requirements.txt
    ├── render.yaml                    (Render 部署配置)
    ├── .gitignore
    └── .env.example
```

## 工作铁律（CRITICAL — 每一步都必须遵守）

### 1. 阶段化执行

本项目分 **6 个阶段**完成。**每个阶段开始前，必须先告诉用户本阶段要做什么、
等用户明确说"继续"或"开始阶段 X"后，才能动手**。

阶段列表：
- **阶段 1**：本地后端骨架（FastAPI + CSV→npy 工具 + `/api/detect` 端点）
- **阶段 2**：前端：下拉菜单替代"Load CSV"，连上 `/api/detect` 验证链路
- **阶段 3**：迁移 4 个核心算法端点
  - 3a: `/api/smooth` (Savitzky-Golay)
  - 3b: `/api/match` (贪心 + DP 匹配)
  - 3c: `/api/auto-fit` (核心 IP：贪心扫描 + 代价函数)
  - 3d: `/api/fine-search` (线性插值 + RMSE 最小)
- **阶段 4**：前端清理(删除已迁移的算法函数)
- **阶段 5**:部署到 Render + GitHub Pages
- **阶段 6**：CORS 白名单 + 限流加固

每个阶段做完后**停下来**，等用户验证成功标志后再说"准备进入下一阶段"。

### 2. 不擅自发挥

- **绝不在用户没要求的情况下重构、优化、美化、加注释**
- **绝不修改本阶段任务以外的文件**
- **绝不一次完成多个阶段**
- 如果发现需要改的范围超出当前阶段，**先停下来问用户**

### 3. UI 绝对不动

`spectrum_analyzer.html` 的所有视觉元素（CSS、布局、颜色、字号、按钮文字、
图表样式）**一行都不能改**。允许改动的只有：
- JS 函数内部实现（从本地算法 → fetch API）
- "Load CSV" 区域的 HTML 结构（替换为下拉菜单）
- 顶部加少量 API 调用工具代码

如果某个改动可能影响 UI 显示，**先告诉用户具体改什么**，等用户确认。

### 4. Git 提交点

**每个阶段开始前**，提醒用户执行 `git status` 和 `git commit`，确保有撤销点。
项目初次进入时，如果还没初始化 git，先 `git init`、`git add .`、
`git commit -m "Initial: original HTML + test CSV"`。

### 5. 单一职责原则

每个 Python 文件只做一件事：
- `algorithms/smoothing.py` 只放 Savitzky-Golay 相关
- `algorithms/detection.py` 只放 find_peaks 相关
- `algorithms/matching.py` 只放贪心/DP 匹配
- `algorithms/fitting.py` 只放 Auto-Fit 和 Fine Search 主循环
- `datasets.py` 只放数据集加载和缓存
- 不要把多个职责混在一个文件里

### 6. 与原版算法等价

迁移到 Python 的算法**必须与原 JS 版本数值等价**。关键的等价点：

- `findPeaks`: distance 单位前端是 nm，scipy.signal.find_peaks 要的是采样点数，
  必须换算：`distSamps = max(1, round(distanceNm / spacing))`
- `prominence`: 用户填 0 时表示"不启用"，传给 scipy 时要变成 None
- `nearestNeighborMatch`: 实现"全部 (gen, exp) 距离对按距离升序贪心配对"
- `Auto-Fit` 代价函数：
  ```
  cost = rmse + penalty × unmatched_count_above_700nm + penalty × (1 - matchRatio)
  ```
  其中 `unmatched_count_above_700nm` 只统计波长 ≥ 700 nm 的未匹配项（这是领域 know-how）
- `Auto-Fit` 选 cost 最小；`Fine Search` 选 RMSE 最小（两者标准不同）
- 差值 ≥ `rmseThreshold` 的匹配对：用 `rmseThreshold²` 计入平方和，**仍计入** matchedN

### 7. 不要碰 CSV→npy 转换工具的输出

`tools/convert_csv_to_npy.py` 跑过一次后，`data/*.npy` 不要重新生成，
除非用户明确要求。

### 8. 调试约定

- 后端启动命令：`uvicorn app.main:app --reload --host 0.0.0.0 --port 8000`
- 后端 API 文档：`http://127.0.0.1:8000/docs` (Swagger UI)
- 用户主要靠浏览器 F12 → Network 看 API 调用，Console 看错误
- 后端日志直接打到终端

### 9. 不主动 push 到 GitHub

阶段 5 之前，**所有代码只在本地**。不要主动建议或执行 `git remote add`、
`git push`。阶段 5 时由用户决定何时推到 GitHub。

### 10. 沟通语言

用户母语是中文，所有解释和说明用中文，代码和命令保持英文。

## 关键算法参考（来自原 HTML）

| 原 JS 函数 | 原文件行号 | 迁移到 Python 文件 |
|---|---|---|
| `savgolFilter` / `applySegmentedSG` / `polyFit` / `matInv` | 1679–1830 | algorithms/smoothing.py |
| `findPeaks` / `calcProminences` / `applyGapMask` | 1837–1898 | algorithms/detection.py |
| `nearestNeighborMatch` (贪心) | 2109 | algorithms/matching.py |
| `matchValleys` (DP) | 1298 | algorithms/matching.py |
| `autoFitThickness` | 2135 | algorithms/fitting.py |
| `runFineSearch` + `interpolateSpectrum` | 2536, 2512 | algorithms/fitting.py |
| `runDetection` | 1952 | algorithms/detection.py |

## 端点契约（保持稳定）

所有端点接受 JSON，返回 JSON。详细 schema 在每阶段开始时给出。
预定的 5 个端点：

- `GET  /datasets` — 返回可用数据集列表（id, name, metadata）
- `POST /api/detect` — 峰谷检测（输入光谱，返回波长列表）
- `POST /api/smooth` — Savitzky-Golay 滤波
- `POST /api/match` — 贪心或 DP 匹配
- `POST /api/auto-fit` — 厚度粗扫
- `POST /api/fine-search` — 厚度精搜

## 当前进度

- [ ] 阶段 1：本地后端骨架
- [ ] 阶段 2：前端连上后端
- [ ] 阶段 3a：/api/smooth
- [ ] 阶段 3b：/api/match
- [ ] 阶段 3c：/api/auto-fit
- [ ] 阶段 3d：/api/fine-search
- [ ] 阶段 4：前端清理
- [ ] 阶段 5：部署
- [ ] 阶段 6：安全加固

**进入下一阶段前，更新此清单**。

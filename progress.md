# Project Progress

## Overview
<!-- Describe this project briefly -->

## Sessions

### 2026-05-13 - Session 1
- Project initialized, progress tracking started

> 会话结束: 2026-05-13 21:49

### 2026-05-14 - Session 2 — 阶段 1 完成
**任务**: 本地后端骨架(FastAPI + CSV→npy 工具 + /api/detect 端点)

**完成内容**:
- Git 初始化,初始提交 `f463f0c`
- CSV 探测:确认 3831×2536,末行为波长(2535 点,400.24–1599.65 nm),末列为厚度(3830 条,85.0–1999.5 nm,步长 0.5)
- 后端目录结构:`backend/{app/{algorithms/},data/,tools/}`、`requirements.txt`、`.gitignore`
- `tools/convert_csv_to_npy.py`:自动布局检测(用"全行多数值 > 10"判定波长行),float32 压缩存储,200 MB CSV → 34 MB `.npy`
- `app/datasets.py`:启动时扫描 `data/` 下 `.npy + .json` 配对加载到全局 `DATASETS` 字典
- `app/algorithms/detection.py`:基于 scipy.signal.find_peaks,nm→采样点换算、prominence/height=0 映射为 None、谷检测对 -y、gap mask 区间剔除
- `app/schemas.py`:Pydantic 模型,字段全 camelCase 与前端 JS 对齐
- `app/main.py`:lifespan 钩子启动加载,`/health`、`/datasets`、`POST /api/detect` 三个端点,CORS allow all(本地开发)
- Swagger UI 自测三个端点全部返回预期

**文件**:
- 新增:`backend/app/{__init__,main,datasets,schemas}.py`、`backend/app/algorithms/{__init__,detection}.py`、`backend/tools/convert_csv_to_npy.py`、`backend/requirements.txt`、`backend/.gitignore`、`backend/data/{mos2.npy,mos2.json,.gitkeep}`
- 修改:`CLAUDE.md`(阶段 1 复选框打勾)

**下一步**: 阶段 2 — 前端用下拉菜单替换"Load CSV",连接 `/api/detect` 验证链路

> 会话结束: 2026-05-14

### 2026-05-14 - Session 3 — 阶段 2 完成
**任务**: 前端连上后端,链路 浏览器 → fetch → FastAPI → 渲染 全打通

**完成内容**:
- `git mv spectrum_analyzer.html frontend/`(保留 rename 历史)
- HTML 顶部插入 `const API = {...}` 统一 fetch 封装层(base, listDatasets, detect)
- Generated 区块由 "Load CSV" 文件输入替换为 `<select id="dataset-select">` 下拉菜单,
  保留 csv-info / thickness-control / thickness-slider 等所有原 ID
- 新增 `initDatasetList()` + `onDatasetChange()`,前者拉 `/datasets`,后者用流式 fetch 拉 `/datasets/{id}/full`
- `runDetection` 改为 `async`,`'gen'` 分支调 `POST /api/detect`,`'exp'` 分支原地保留(留给阶段 3)
- `window.addEventListener('load', ...)` 里增加 `initDatasetList()` 调用
- 后端新增 `GET /datasets/{id}/full` 端点(本阶段唯一后端改动)
- 下拉菜单文案精简: `MoS2 (400-1600 nm, thickness_step=0.5nm)`(波长四舍五入,去掉光谱数)
- 加载界面增强: 加进度条 + 实时下载 MB / 百分比 / ETA,
  用 `ReadableStream` 流式读 chunk + `Content-Length` 算进度

**文件**:
- 新增/修改: `frontend/spectrum_analyzer.html`(从根目录移入,大量改动)
- 修改: `backend/app/main.py`(加 /datasets/{id}/full 端点)
- 修改: `CLAUDE.md`(阶段 2 复选框打勾)

**已知优化点(留待阶段 5 部署前处理)**:
- `/datasets/{id}/full` JSON 体积 ~185 MB,localhost 6 秒,公网会显著变慢。
  备选方案: npy 二进制流传输,或按需切片(只拉当前 thickness 一条光谱)

**已记忆**: 用户尚未注册 Render 账号,到阶段 5 前需要提醒
(memory/render_account_status.md)

**下一步**: 阶段 3 — 把 4 个核心算法迁移到后端
(3a smoothing, 3b matching, 3c auto-fit, 3d fine-search)

> 会话结束: 2026-05-14

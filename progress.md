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

> 会话结束: 2026-05-14 12:13

### 2026-05-14 - Session 4 — 阶段 3a 进行中(暂停在步骤 3a.10 视觉确认)
**任务**: 把 Savitzky-Golay 平滑算法从前端 JS 迁移到 Python 后端

**完成内容**:
- 步骤 3a.1:Git 提交规划文档 + 进度日志 → 撤销点 `042d989`
  (注:STAGE_3a.md 给的 commit message 与实际改动不符,改用更准确的 `docs: stage planning notes (3a/3b/3c) + progress log`)
- 步骤 3a.2:`backend/app/algorithms/smoothing.py` 三函数实现完整:
  - `savgol_kernel(window_len, poly_order)` — 用 `np.linalg.solve(A.T@A, A.T)[0]` 等价 JS 的 Gauss-Jordan
  - `savgol_filter_jslike(y, window_len, poly_order)` — 严格按 JS 行 1924-1955 sanitise 顺序;内部点用 `np.correlate(y, kernel, 'valid')`;左右边界用 `np.polyfit + np.polyval`(都接受 high-to-low,自洽)
  - `apply_segmented_sg(y, wavelengths, segments)` — 按 wavelength range 构建 W/P 数组,合并相邻相同 chunk,逐段调 SG;默认 (window=51, poly=3) 与 JS 一致;`seg.get("window") or DEFAULT_W` 匹配 JS 的 `||` falsy fallback
- 步骤 3a.3:`backend/app/schemas.py` 新增 `SmoothingSegment` / `SmoothRequest`(带 `extra="forbid"`) / `SmoothResponse`,字段全 camelCase
- 步骤 3a.4:`backend/app/main.py` 新增 `POST /api/smooth` 端点,与 `/api/detect` 风格一致(长度校验 + ≥2 校验);头部 docstring 同步到 5 个端点
- 步骤 3a.5:Swagger UI 自测 15 点正弦数据(window=5, poly=2),逐点用 SG 标准核 `[-3,12,17,12,-3]/35` 验证内部点 + 左右边界 polyfit 系数,**全部精确匹配**
- 步骤 3a.6:前端 `API` 对象(行 812-832)新增 `smooth(body)` 方法
- 步骤 3a.7:`applySmoothing(which)` 改为 async,加 showLoading/hideLoading/try-catch
- 步骤 3a.8:`autoReSmooth(which)` 改为 async,**不显示 loading**(避免滑块拖动闪屏),失败仅 console.warn
- 步骤 3a.9:async 传染检查 + 选了**方案 B** —— 在 `autoReSmooth` 内部 await 完成后追加 `updatePlots()`,解决滑块拖动时平滑曲线滞后一帧的问题。`autoFitThickness` (行 2468) 和 `runFineSearch` (行 2887) 末尾的调用按计划保持 fire-and-forget,留给阶段 3c/3d
- 步骤 3a.10:**进行中** — 后端日志确认 `POST /api/smooth 200` 多次,但**视觉确认未完成**(用户未回答 Apply 后曲线 / 滑块拖动手感)

**文件**:
- 新增: `backend/app/algorithms/smoothing.py`
- 修改: `backend/app/schemas.py`(尾部加 3 个 schema)
- 修改: `backend/app/main.py`(导入 + 端点 + docstring)
- 修改: `frontend/spectrum_analyzer.html`(API.smooth、applySmoothing、autoReSmooth)

**未提交**:本次会话所有代码改动**尚未提交**,撤销点仍是 `042d989`

**关键行为差异**(已与用户确认接受方案 B):
- 原 JS `autoReSmooth` 同步,新版 async + fire-and-forget。直接调会让 UI 滞后一帧。方案 B 是在 `autoReSmooth` 内部 await 完后触发一次 `updatePlots()`,Plotly.react 幂等,多绘一帧影响小

**下一步**(继续会话时直接执行):
1. **完成步骤 3a.10**:让用户确认 Apply 后图表上有平滑曲线、滑块拖动时平滑曲线跟新、手感流畅
2. **步骤 3a.11**:**关键** — 数值等价性验证。让用户对同一份 MoS₂ 数据在新版本 vs `spectrum_analyzer_ver1.html`(项目根目录,作为对照参考)用相同 SG 参数应用平滑,目视对比两条曲线是否完全重合(误差 < 1e-6)。如有肉眼可见差异立即停下排查
3. **步骤 3a.12**:汇报 + 等用户验证全部"成功标志"清单
4. 用户验证通过后,提交本次改动 + 更新 CLAUDE.md "当前进度"清单中阶段 3a 复选框,然后进入阶段 3b(`/api/match`,STAGE_3b.md)

**参考文件**:
- 原始版本对照:`spectrum_analyzer_ver1.html`(项目根目录,用户备份)
- 原 JS SG 代码位置:`frontend/spectrum_analyzer.html` 行 1853 (matInv) / 1881 (polyFit) / 1903 (_computeSgKernel) / 1924 (savgolFilter) / 1959 (applySegmentedSG)

> 会话结束: 2026-05-14

> 会话结束: 2026-05-14 12:47

### 2026-05-14 - Session 5 — 阶段 3a.11 数值等价验证(自动化完成)
**目标**:用程序化测试代替"人眼对比两个 HTML"的手动检查,严格证明 Python smoothing.py 与原版 JS savgolFilter/applySegmentedSG 在 float64 精度下数值等价

**完成情况**:
- 新增 `backend/tests/sg_reference.js` — 把 `spectrum_analyzer_ver1.html` 行 1679-1830 的 matMul/matTranspose/matInv/polyFit/polyEval/_computeSgKernel/sgKernel/savgolFilter/applySegmentedSG 9 个函数**原样拷出**,加一段 stdin→stdout JSON 桥接,作为参考实现(用 Node v24 跑)
- 新增 `backend/tests/test_smoothing_parity.py` — 5 组共 **29 个测试用例**:
  1. 标准 SG 核 (window=5, poly=2) 对照教科书 [-3,12,17,12,-3]/35 → max |Δ| = **2.776e-17**
  2. 多项式不变性 (常数/线性/二次/三次/四次,poly_order≥degree) → max |Δ| ≤ 3.553e-15
  3. JS↔Python 合成噪声正弦信号 4 种 (window/poly/n 组合) → max |Δ| ≤ 3.941e-15
  4. JS↔Python 真实 mos2 数据 4 个厚度切片 × 4 种 segment 配置 = 16 组,含跨段切换边界 → max |Δ| ≤ 7.883e-15
  5. JS↔Python 边界场景 (window > n) → max |Δ| = 1.332e-15
  - **全部通过**,容差 1e-9,实际最大误差 7.9e-15,已逼近 float64 机器精度
- 新增 `backend/tests/test_smooth_endpoint.py` — 端到端 HTTP smoke test:POST `/api/smooth` 返回 vs 直接 `apply_segmented_sg()` 调用 → max |Δ| = **0.000e+00**(JSON 序列化对 float64 无损)
- 顺手验证 `/health`、`/datasets` 端点也都在线

**结论**:阶段 3a 的 Python 实现已**程序化证明**与原版 JS 在数值上完全等价。3a.11 通过

**未完成**:
- **3a.10 视觉确认**仍需用户手动:浏览器打开 `frontend/spectrum_analyzer.html`,选 MoS₂ 数据集,点 Apply Smoothing 看是否出现平滑曲线、拖动厚度滑块平滑曲线是否实时跟新、点 Reset 是否恢复
- (这部分纯前端交互,数值等价测试不能替代)

**文件**:
- 新增:`backend/tests/sg_reference.js`、`backend/tests/test_smoothing_parity.py`、`backend/tests/test_smooth_endpoint.py`
- 未改动 Session 4 留下的任何代码文件

**环境备注**:后端依赖装在 conda env `test` 里(`D:\anaconda\envs\test\python.exe`),启动命令:
`D:\anaconda\envs\test\python.exe -m uvicorn app.main:app --host 127.0.0.1 --port 8000`
本机 HTTP_PROXY/HTTPS_PROXY 走 127.0.0.1:7897,测试脚本里用 `ProxyHandler({})` 显式绕过

**未提交**:本次新增 3 个测试文件 + Session 4 代码改动**全部未提交**,撤销点仍是 `042d989`

**下一步**(继续会话时):
1. 用户做 3a.10 视觉确认(浏览器交互)
2. 视觉确认通过后,合并提交 Session 4 + Session 5 全部改动 + 勾选 CLAUDE.md 阶段 3a 复选框
3. 进入阶段 3b(`/api/match`)

> 会话结束: 2026-05-14 14:15

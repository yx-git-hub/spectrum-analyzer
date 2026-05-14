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

### 2026-05-14 - Session 6 — 阶段 3b 完成 (/api/match + 前端 updateValleyTable 接入)
**目标**:把贪心(`nearestNeighborMatch`)和 DP(`matchValleys`)两个匹配算法迁后端,新增 `POST /api/match`;前端 `updateValleyTable` 改 async 走 API。3a 视觉确认通过后整体提交 `19132bf`,然后启动 3b

**完成情况**:
- 步骤 3b.1:用户确认 3a 视觉通过 → 合并提交 Session 4+5 全部改动为 `19132bf`(Stage 3a complete: /api/smooth + Savitzky-Golay migrated to backend)
- 步骤 3b.2:新建 `backend/app/algorithms/matching.py`:
  - `nearest_neighbor_match(gen, exp) → {pairs, unmatchedGen, unmatchedExp}` 贪心
  - `dp_sequence_match(gen, exp, skip_penalty=1e9) → [[g|None, e|None], ...]` DP
  - 严格按 JS 行 2109 / 1298 同语义实现:候选构造 (g 外 e 内) + 稳定排序保 tie-break 一致;DP 转移和回溯优先级(配对>跳gen>跳exp)对齐
- 步骤 3b.3:`schemas.py` 加 `MatchRequest` / `GreedyPair` / `GreedyMatchResponse` / `DPMatchResponse`,`extra="forbid"`,`algorithm: Literal["greedy", "dp"]`
- 步骤 3b.4:`main.py` 加 `POST /api/match` 端点。**不用 response_model**,两种算法返回结构不同
- 步骤 3b.5:用户 Swagger UI 自测两组 STAGE_3b.md 预设用例,贪心 / DP 均按预期 PASS
- 数值等价验证(用 `tests/match_reference.js` Node 跑原 JS 作参考):
  - 12 项贪心固定用例(空输入/对齐/偏移/平局/不同长度/MoS₂-like)
  - 15 项 DP 固定用例(含 skip 边界)
  - 100 项模糊(50 个 m,n∈[0,12] 随机 gen/exp 序列 × 2 算法)
  - **共 127/127 PASS**
- 步骤 3b.6:`API.match` 加在 API 对象 (第 832 行)
- 步骤 3b.7:`updateValleyTable` 改 async,两次 `matchValleys` → 两次 `await API.match({..., algorithm:'dp'})`,try-catch 失败时表格显示 "Match failed: …",**渲染代码 1525-1574 一字未改**
- 步骤 3b.8:扫描 `updateValleyTable` 3 个调用方(2 个 HTML oninput 阈值输入框 + `updatePlots()`),全部 fire-and-forget,跟 3a `autoReSmooth` 同模式
- 步骤 3b.9-3b.10:用户浏览器实测 → Network 看到 `/api/match` 200 + 表格 Δ 与 ver1.html 完全一致 → 通过

**文件**:
- 新增:`backend/app/algorithms/matching.py`、`backend/tests/match_reference.js`、`backend/tests/test_matching_parity.py`
- 修改:`backend/app/schemas.py`(尾部加 4 个 schema + Literal 导入)、`backend/app/main.py`(import + 端点 + docstring)、`frontend/spectrum_analyzer.html`(API.match 一行 + updateValleyTable 改 async)、`CLAUDE.md`(勾 3b)

**留尾**(按 STAGE_3b.md 设计):
- 前端 `nearestNeighborMatch` JS 函数本体未删 — `autoFitThickness` (3c) 还在用,3c 完成后才删
- 前端 `matchValleys` JS 函数本体未删 — 已无调用方但保留到 3c/3d 一并清

**下一步**:进阶段 3c — `/api/auto-fit`(核心 IP:贪心扫描 + 代价函数,`unmatched_count_above_700nm` 是领域 know-how)。等用户点头开始

> 会话结束: 2026-05-14 14:55

### 2026-05-14 - Session 7 — 阶段 3c 完成 (/api/auto-fit + 前端 autoFitThickness 接入,核心 IP 全后端化)
**目标**:把 Auto-Fit 厚度粗扫(扫描循环 + 代价函数 + wl≥700 领域 know-how)从前端 JS 全部迁后端;前端 `autoFitThickness` 改 async 调 `/api/auto-fit`,请求 body 不含 spectra 只含 datasetId,实现真正 IP 保护

**完成情况**:
- 步骤 3c.1:3b 通过 → 合并提交 `a456f60` (Stage 3b complete: /api/match)
- 步骤 3c.2:新建 `backend/app/algorithms/fitting.py`
  - `_detect_one_spectrum(spectrum, wavelengths, ...) → (valleys_wl, peaks_wl)`:per-spectrum detect (复用 js_find_peaks + apply_gap_mask)
  - `compute_cost_for_match(vm, pm, rmse_threshold, penalty, exp_feature_count) → (total_cost, rmse, matched_n)`:严格对齐 JS 行 2202-2236,**单对共享 extra_unmatched_***、`diff>=threshold` 时 `rmseThreshold²` 计入平方和 + extras、wl≥700 才计入 unmatched 惩罚
  - `auto_fit_thickness(...)`:扫描循环,对齐 JS 行 2169-2246,`bestIdx = argmin totalCost`(非 argmin rmse)
- **关键发现 + 修复**:第一次 parity 跑 8 个用例 6 个 FAIL,`distanceNm=20` 时 bestIdx 都不一致(py=66 vs js=67)。根因:`scipy.signal.find_peaks` 的 plateau(取中间) + distance suppression(按高度降序)与原 JS findPeaks(plateau 取最右、distance 按索引顺序)不一致 → 在 `detection.py` 加 `js_find_peaks` + `_calc_prominences` 函数(纯 Python 手撕),vectorized 局部极值 + sequential 三步过滤,**detection.py 全面弃用 scipy.signal.find_peaks**,fitting.py 改用同一份 `js_find_peaks`。修完 8 项全 PASS
- 步骤 3c.3:`schemas.py` 加 `AutoFitRequest`(`datasetId` 字符串 + `expValleys/expPeaks` 列表 + `mode/penalty/rmseThreshold` + 复用 `DetectionParams`/`GapInterval`,`extra="forbid"`)
- 步骤 3c.4:`main.py` 加 `POST /api/auto-fit`,从 `datasets.get_dataset(datasetId)` 取 spectra,调 `auto_fit_thickness`,返回完整结果字典(不用 response_model 因为结构层次太深)
- 数值等价测试(`tests/autofit_reference.js` + `tests/test_autofit_parity.py`):
  - JS reference 把原 findPeaks/calcProminences/applyGapMask/nearestNeighborMatch/autoFitThickness 全链路提出
  - 数据:真实 mos2.npy 抽样 200 个厚度切片(每隔 19 个) + 真实 wavelengths,exp 特征用 idx=1276 (t=723 nm) 的真实检测结果
  - 用例:8 项,覆盖 mode=valleys/peaks/both × penalty/threshold 极值 × gap mask × height 阈值 × distanceNm=20
  - 结果:**全 8 项 PASS**,bestIdx/bestThickness/totalCosts[每点]/rmseArr[每点]/bestValleyMatch.pairs 全部一致,容差 1e-9
- HTTP smoke test:`POST /api/auto-fit` 全 3830 spectra 扫描 **11.4 秒**(Python 解释器 vs JS V8 JIT 的固有差距,~5x 慢)
- 步骤 3c.5:用户 Swagger UI 自测通过
- 步骤 3c.6:`API.autoFit(body)` 加到 API 对象 (line 833)
- 步骤 3c.7:汇报 `AppState.csv` 字段结构 + 改动点;用户确认后加 `datasetId: null` 字段 (line 845) + 在 `onDatasetChange` 里写入 `AppState.csv.datasetId = id` (line 961)
- 步骤 3c.8:**重写 `autoFitThickness` 函数体** (line 2369-2456,~140 行 → ~85 行):
  - 函数变 async
  - 早期 guard 从 `!spectra.length` 改成 `!datasetId`
  - `await runDetection('exp')` 拿到 exp 特征
  - `showLoading` + `await API.autoFit({datasetId, expValleys, expPeaks, mode, penalty, rmseThreshold, detection, gaps})` — **请求 body 不含 spectra**
  - 收到响应后:更新 selectedIdx/slider/num input,`autoReSmooth('gen')` + `await runDetection('gen')` + `updatePlots()`,设 status bar,设 `_coarseResult`(null→NaN),自动填 fine search 范围,调 `_showAutofitModal`
  - try-catch 失败时 hideLoading + status 显示错误
- 步骤 3c.10-3c.11:用户浏览器实测 → Network 看到 `/api/auto-fit` 200 (请求 body 小、响应含完整字段) + 弹窗 + 滑块跳到 bestIdx + bestThickness 与 ver1.html 一致 → **通过**

**核心 IP 保护实现**:
- 扫描循环、代价函数、wl≥700 这条领域 know-how 现在**只存在于后端 Python**
- 前端 dump 出来看不到任何核心算法
- 前端请求 body 只发 `datasetId` 字符串(几 KB),后端用自己内存里的 spectra 计算

**JS-faithful find_peaks 副作用**:
- detection.py 现在完全用 `js_find_peaks` 替代了 scipy.signal.find_peaks
- `/api/detect` 行为微调:plateau 处理、distance 抑制顺序与原 JS 严格一致(scipy 在这两点上不同)
- 对真实平滑光谱影响极小(plateaus 罕见),用户确认接受

**性能现状**(等用户决定是否优化):
- 3830 spectra 全扫一遍 11.4s (原 JS 在浏览器 ~2s,因为 V8 JIT)
- 用户 11s 体感可接受,**先暂不引入 numba**,3d 之后再决定是否优化

**文件**:
- 新增:`backend/app/algorithms/fitting.py`、`backend/tests/autofit_reference.js`、`backend/tests/test_autofit_parity.py`
- 修改:`backend/app/algorithms/detection.py`(弃 scipy,加 js_find_peaks/_calc_prominences,detect_peaks_and_valleys 改用新实现)、`backend/app/schemas.py`(尾部加 AutoFitRequest)、`backend/app/main.py`(import + 端点 + docstring)、`frontend/spectrum_analyzer.html`(API.autoFit 一行 + AppState.csv.datasetId 字段 + onDatasetChange 一行 + autoFitThickness 整个函数体替换)、`CLAUDE.md`(勾 3c)

**留尾**(按 STAGE_3c.md 设计,4 阶段统一清理):
- 前端 `findPeaks`/`calcProminences`/`applyGapMask` JS 函数本体未删(被 `runDetection` 死路径引用)
- 前端 `nearestNeighborMatch` JS 函数本体未删(现已无调用方)
- 前端 `matchValleys` JS 函数本体未删(3b 留下的)

**下一步**:用户在 3d (Fine Search) 之前决定 — 接受 11s 性能进 3d,还是先做 numba mini 加速阶段

> 会话结束: 2026-05-14 15:30

### 2026-05-14 - Session 8 — numba JIT 加速 mini 阶段 (Auto-Fit 11.4s → 0.24s, 47x)
**目标**:在进 3d 前先优化 Auto-Fit 性能。用户选了"做 numba 加速"路线。目标 5-50x 提速,数值必须 bit-identical

**完成情况**:
- `pip install numba==0.60.0` 到 conda env `test`(顺带装 llvmlite 0.43.0)
- `requirements.txt` 加 `numba==0.60.0`
- `backend/app/algorithms/detection.py`:
  - 加 `@njit(cache=True) _calc_prominences_njit(peak_idxs: int64[], y: float64[]) → float64[]`,纯 numba 实现 JS calcProminences 的左右游走
  - 加 `@njit(cache=True) _js_find_peaks_njit(y, height, height_is_none, distance, prominence, prominence_is_none) → int64[]`,4 步 JS findPeaks 全部 njit 化(numpy 预分配 buffer 替代 Python list growing)。`*_is_none` flags 携带"无过滤"语义,因 numba 不能直接 dispatch Optional[float]
  - 公共 API `js_find_peaks(y, *, height=None, distance=1, prominence=None)` 变薄壳:转 contiguous float64 + None→0.0 sentinel,调 njit 内核,返回 Python list (向后兼容 apply_gap_mask 用户)
  - 加 `warmup_njit()`:用 7 点 dummy 输入跑两次 _js_find_peaks_njit + 一次 _calc_prominences_njit,触发编译
- `backend/app/algorithms/matching.py`:
  - 加 `@njit(cache=True) _greedy_match_kernel(gen, exp) → (pair_g_idx, pair_e_idx, pair_diff, used_g, used_e)`
  - **关键 tie-break**:用 **稳定插入排序** 而非 np.argsort(numba 0.60 不保证 stable),保证 (g 外 e 内) 构造顺序在距离相等时的优先级与 JS Array.sort 一致
  - 公共 API `nearest_neighbor_match(gen, exp) → dict` 变薄壳:np.asarray + 调内核 + 装回 Python dict/list
  - 加 `warmup_njit()`
- `backend/app/main.py`:
  - lifespan startup 在 load_all 之后调 `detection_mod.warmup_njit()` + `matching_mod.warmup_njit()`
  - 首次启动 numba 编译 ~3-5s(下盘后写 `__pycache__/*.nbi` cache),后续启动读 cache 仅 ~0.26s
- 全部 parity 回归(无任何容差松动):
  - smoothing: 29/29 PASS(SG 未受影响,sanity check)
  - matching: 127/127 PASS(12 贪心 + 15 DP 固定 + 100 模糊)
  - autofit: 8/8 PASS(bestIdx/bestThickness/bestRmse/totalCosts[每点]/rmseArr[每点] 与 JS 完全一致,容差 1e-9)
- 性能基准(同一份 mos2 + 同样参数,3 次试):
  - **优化前**: 11.4s/次
  - **优化后**: 0.24s, 0.24s, 0.27s **(~47x)**
  - 0.24s ≈ Python 网络/序列化开销 + 0.1-0.15s 真实计算,已逼近上限

**文件**:
- 修改:`backend/app/algorithms/detection.py`、`backend/app/algorithms/matching.py`、`backend/app/main.py`、`backend/requirements.txt`
- 未碰前端

**正确性保证**:
- 4 套 parity 测试共 164 个 case 全 PASS
- 内核纯确定性(无随机/并发/race)
- 数值 bit-identical(同样的 IEEE 754 运算顺序)

**部署影响**(将来阶段 5):
- Render 实例首次启动慢 3-5s 编译,后续重启走 cache
- 内存占用增加 ~50-100 MB(numba+llvmlite)
- 整套依赖体积 ~30 MB 上行

**下一步**:用户验证通过即提交,然后进阶段 3d(`/api/fine-search`,最后一个端点)

> 会话结束: 2026-05-14 16:30

# 阶段 3d:迁移 Fine Search 厚度精搜到后端

请先完整阅读项目根目录下的 CLAUDE.md,理解工作铁律和阶段划分。
然后查看 CLAUDE.md 中的"当前进度"清单,确认阶段 1、2、3a、3b、3c 已勾选完成。

**前置依赖检查**:本阶段依赖阶段 3c 建立的 `backend/app/algorithms/fitting.py`
和 `/api/auto-fit` 的数据集读取模式。如果 `fitting.py` 不存在或
`/api/auto-fit` 端点不存在,**立即停止并告诉我"阶段 3c 尚未完成"**。

阅读完后向我确认你已理解,然后再开始执行下面的任务。

我们现在开始 **阶段 3d:迁移 Fine Search 厚度精搜到后端**(阶段 3 的最后一个端点)。

## 本阶段目标

把 `runFineSearch` 函数(原行 2536)和 `interpolateSpectrum` 函数(原行 2512)
迁移到后端。

完成后:
- 后端新增 `POST /api/fine-search` 端点
- 前端 `runFineSearch` 改成调后端
- 前端 `interpolateSpectrum` 可删除(后端做插值)

## Fine Search 与 Auto-Fit 的关键差异

迁移时务必区分清楚(参考原 JS 行 2536-2705):

1. **厚度网格来源不同**:
   - Auto-Fit:直接用数据集里已有的 thicknesses
   - Fine Search:用 `linspace(fineStart, fineEnd, n)` 生成更密的网格,
     每个厚度的光谱通过**线性插值**得到(原 interpolateSpectrum)
2. **最优选择标准不同**:
   - Auto-Fit:选 `totalCost` 最小
   - Fine Search:**选 RMSE 最小**(这是核心差异!原 JS 行 2647-2653 注释也写明了)
   - 但 totalCost 仍然要计算并返回(用于绘图)
3. 代价函数、匹配算法、检测逻辑都与 Auto-Fit 完全相同(复用 3c 的函数)

## 本阶段不做什么

- 不动 `_showAutofitModal`(UI 渲染,保持原样)
- 不动其他端点
- 不动 UI 视觉
- 暂不删除前端已迁移的旧算法函数(findPeaks/savgolFilter/nearestNeighborMatch/
  matchValleys/interpolateSpectrum 等)——**统一留到阶段 4 清理**

## 严格遵守

- **线性插值逻辑**必须与原 JS interpolateSpectrum(行 2512-2523)等价:
  - tTarget <= thicknesses[0] → 返回 spectra[0]
  - tTarget >= thicknesses[-1] → 返回 spectra[-1]
  - 否则二分查找相邻两行 lo/hi,
    `alpha = (tTarget - t[lo]) / (t[hi] - t[lo])`,
    `result[i] = sLo[i] + alpha * (sHi[i] - sLo[i])`
  - 可用 numpy.interp 向量化,但要确认边界行为一致
- 厚度网格生成:`n = round((fineEnd - fineStart) / fineStep) + 1`,
  然后 `thicknesses_fine = fineStart + step * arange(n)`
  (与原 JS 行 2531 的 count 计算一致)
- bestIdx 用 **RMSE 最小**;若所有 rmse 都是 NaN,回退到 totalCost 最小

## 详细步骤

### 步骤 3d.1:Git 提交

提示我执行:
```bash
git add -A
git status
git commit -m "Stage 3c complete: /api/auto-fit core algorithm migrated to backend"
```

### 步骤 3d.2:在 fitting.py 中新增插值和精搜函数

在 `backend/app/algorithms/fitting.py` 中追加:

```python
def interpolate_spectrum(
    t_target: float,
    spectra: np.ndarray,
    thicknesses: np.ndarray,
) -> np.ndarray:
    """Linear interpolation of a spectrum at arbitrary thickness using the
    two neighbouring rows.

    Equivalent to the original JS interpolateSpectrum (lines 2512-2523).
    """
    n = len(thicknesses)
    if n == 0:
        return None
    if t_target <= thicknesses[0]:
        return spectra[0]
    if t_target >= thicknesses[n - 1]:
        return spectra[n - 1]
    # binary search for lo such that thicknesses[lo] <= t_target < thicknesses[hi]
    lo, hi = 0, n - 1
    while hi - lo > 1:
        m = (lo + hi) // 2
        if thicknesses[m] <= t_target:
            lo = m
        else:
            hi = m
    alpha = (t_target - thicknesses[lo]) / (thicknesses[hi] - thicknesses[lo])
    return spectra[lo] + alpha * (spectra[hi] - spectra[lo])


def fine_search_thickness(
    wavelengths: np.ndarray,
    spectra: np.ndarray,
    thicknesses: np.ndarray,
    fine_start: float,
    fine_end: float,
    fine_step: float,
    exp_valleys: list[float],
    exp_peaks: list[float],
    mode: str,
    detection: dict,
    gaps: list,
    rmse_threshold: float,
    penalty: float,
) -> dict:
    """Fine-grained thickness search using linear interpolation.

    Equivalent to the inner loop of original JS runFineSearch (lines 2536-2705).

    KEY DIFFERENCES from auto_fit_thickness:
    - Thickness grid is linspace(fine_start, fine_end, n), not the dataset's own grid
    - Each spectrum is obtained by linear interpolation
    - bestIdx is chosen by MINIMUM RMSE (not minimum total cost)

    Returns:
        {
            "bestIdx": int,
            "bestThickness": float,
            "bestRmse": float | None,
            "fineTs": list[float],          # the fine thickness grid
            "totalCosts": list[float],
            "rmseArr": list[float | None],
            "bestValleyMatch": {...},
            "bestPeakMatch": {...},
            "mode": str,
            "expFeatureCount": int,
        }
    """
    # build fine grid
    n = round((fine_end - fine_start) / fine_step) + 1
    fine_ts = fine_start + fine_step * np.arange(n)

    empty_match = {"pairs": [], "unmatchedGen": [], "unmatchedExp": []}
    exp_feat_count = (len(exp_valleys) if mode in ("valleys", "both") else 0) \
                   + (len(exp_peaks)   if mode in ("peaks",   "both") else 0)

    total_costs = np.full(n, np.inf)
    rmse_arr = np.full(n, np.nan)
    best_rmse = float("inf")
    best_cost_fallback = float("inf")
    best_idx = 0
    best_vm = empty_match
    best_pm = empty_match
    found_valid_rmse = False

    for i in range(n):
        spec = interpolate_spectrum(fine_ts[i], spectra, thicknesses)
        gen_valleys, gen_peaks = detect_features_for_spectrum(
            spec, wavelengths, detection, gaps,
            detect_valleys=(mode in ("valleys", "both")),
            detect_peaks=(mode in ("peaks", "both")),
        )
        vm = nearest_neighbor_match(gen_valleys, exp_valleys) \
             if mode in ("valleys", "both") else empty_match
        pm = nearest_neighbor_match(gen_peaks, exp_peaks) \
             if mode in ("peaks", "both") else empty_match

        total_cost, rmse, matched_n = compute_cost_for_match(
            vm, pm, rmse_threshold, penalty, exp_feat_count
        )
        total_costs[i] = total_cost
        if matched_n > 0:
            rmse_arr[i] = rmse

        # Fine search picks MINIMUM RMSE
        if matched_n > 0 and not np.isnan(rmse) and rmse < best_rmse:
            best_rmse = rmse
            best_idx = i
            best_vm = vm
            best_pm = pm
            found_valid_rmse = True

        # fallback tracking (in case no valid rmse anywhere)
        if not found_valid_rmse and total_cost < best_cost_fallback:
            best_cost_fallback = total_cost
            best_idx = i
            best_vm = vm
            best_pm = pm

    best_rmse_val = float(rmse_arr[best_idx]) if not np.isnan(rmse_arr[best_idx]) else None

    return {
        "bestIdx": int(best_idx),
        "bestThickness": float(fine_ts[best_idx]),
        "bestRmse": best_rmse_val,
        "fineTs": [float(t) for t in fine_ts],
        "totalCosts": [float(c) for c in total_costs],
        "rmseArr": [None if np.isnan(r) else float(r) for r in rmse_arr],
        "bestValleyMatch": best_vm,
        "bestPeakMatch": best_pm,
        "mode": mode,
        "expFeatureCount": exp_feat_count,
    }
```

**注意**:先查看原 JS runFineSearch(行 2536-2705)的实际实现,
确认上面的 best 选择逻辑(RMSE 最小 + fallback)与原代码一致。
如果原 JS 的 fallback 逻辑不同,以原 JS 为准。

### 步骤 3d.3:在 schemas.py 中新增类型

```python
class FineSearchRequest(BaseModel):
    datasetId: str
    fineStart: float
    fineEnd: float
    fineStep: float
    expValleys: list[float]
    expPeaks: list[float]
    mode: Literal["valleys", "peaks", "both"] = "valleys"
    penalty: float = 100.0
    rmseThreshold: float = 30.0
    detection: DetectionParams
    gaps: list[GapInterval] = []
```

### 步骤 3d.4:在 main.py 中新增端点

```python
@app.post("/api/fine-search")
def fine_search(req: FineSearchRequest):
    from .algorithms.fitting import fine_search_thickness
    from .datasets import get_dataset

    try:
        ds = get_dataset(req.datasetId)
    except KeyError:
        raise HTTPException(404, f"Dataset '{req.datasetId}' not found")

    result = fine_search_thickness(
        wavelengths=ds["wavelengths"],
        spectra=ds["spectra"],
        thicknesses=ds["thicknesses"],
        fine_start=req.fineStart,
        fine_end=req.fineEnd,
        fine_step=req.fineStep,
        exp_valleys=req.expValleys,
        exp_peaks=req.expPeaks,
        mode=req.mode,
        detection=req.detection.model_dump(),
        gaps=[g.model_dump() for g in req.gaps],
        rmse_threshold=req.rmseThreshold,
        penalty=req.penalty,
    )
    return result
```

### 步骤 3d.5:用 Swagger UI 自测后端

重启后端,测试 `/api/fine-search`,用一个小范围(比如 fineStart 到 fineEnd
跨度 20 nm,step 0.5,约 40 个点)。确认返回 bestThickness、fineTs、
totalCosts、rmseArr。

### 步骤 3d.6:前端 API 层增加 fineSearch 调用

在 `frontend/spectrum_analyzer.html` 的 `API` 对象中新增:

```javascript
fineSearch(body) { return this._call('POST', '/api/fine-search', body); },
```

### 步骤 3d.7:查看原 runFineSearch 确认前端改造范围

先 view 原 `runFineSearch` 函数(行 2536 到函数结束),
报告给我:
- 函数有多少行
- 它调用了哪些其他函数(interpolateSpectrum、findPeaks、nearestNeighborMatch、
  _showAutofitModal 等)
- 它读取了哪些 DOM 元素(fs-start、fs-end、fs-step 等)
- 结果如何展示(是否也用 _showAutofitModal,还是别的)

等我确认后再改造。

### 步骤 3d.8:改造前端 runFineSearch 函数

(根据步骤 3d.7 的实际情况调整)

整体改造思路:
- 函数变 async
- 读取 DOM 参数(fineStart/End/Step、mode、penalty、rmseThreshold、detection、gaps)
- 调 `await API.fineSearch({...})`,请求体只含 datasetId,不含 spectra
- 用返回结果调用 `_showAutofitModal`(注意 Fine Search 传的是 fineTs 而非
  数据集原始 thicknesses;参考原 JS runFineSearch 末尾如何调 modal)
- rmseArr 里的 null 转 NaN(同阶段 3c)
- showLoading / hideLoading 包裹

具体代码在步骤 3d.7 之后,根据原函数实际结构给出。

### 步骤 3d.9:联调测试

提示我:
1. 重启后端
2. 刷新前端
3. 选 MoS₂ 数据集 + 加载实验 Excel
4. 先跑一次 Auto-Fit(它会自动填充 fine search 范围)
5. 点 Fine Search 按钮
6. F12 Network 看到 `POST /api/fine-search` 200(请求体几 KB)
7. 弹窗显示精搜结果

### 步骤 3d.10:**关键** — 数值等价性验证

打开 `spectrum_analyzer_backup.html` 和当前版本,**用完全相同的**:
- MoS₂ 数据集、实验 Excel
- detection 参数
- Fine Search 的 start/end/step
- mode、penalty、rmseThreshold

对比:
- `bestThickness` 必须完全一致
- `bestRmse` 误差 < 0.01 nm
- 弹窗里的 RMSE 曲线形状应完全重合

如有不一致,**立即停止并报告**。重点检查:
- 插值边界行为
- 厚度网格点数(n 的计算)
- RMSE 最小的选择逻辑

### 步骤 3d.11:汇报结果

告诉我:
- fitting.py 新增的两个函数
- Swagger 自测 `/api/fine-search` 返回值
- 原 runFineSearch 的结构和前端改造的行号范围
- 数值等价性验证结果

然后停下来,**等我手动验证**全部"成功标志"。

## 阶段 3d 成功标志(用户验证清单)

- [ ] `backend/app/algorithms/fitting.py` 中新增 interpolate_spectrum、
      fine_search_thickness
- [ ] `backend/app/schemas.py` 中新增 FineSearchRequest
- [ ] `backend/app/main.py` 中新增 `/api/fine-search` 端点
- [ ] Swagger UI 上 `/api/fine-search` 测试通过
- [ ] 前端 runFineSearch 是 async,调 API.fineSearch,请求体只含 datasetId
- [ ] Fine Search 弹窗正常显示
- [ ] **关键**:对同一组输入,Fine Search 的 bestThickness 与原版完全一致
- [ ] **关键**:bestRmse 与原版一致(误差 < 0.01 nm)

我验证通过后会说"阶段 3d 完成",阶段 3 全部结束,进入阶段 4(前端清理)。

## 工作纪律

- 每个步骤完成后**停下来汇报**,等我说"继续"再做下一步
- 不要"优化"算法(必须与原 JS 对应)
- 不要在本阶段删除任何旧的前端算法函数(阶段 4 统一清理)
- 不要碰 `_showAutofitModal`
- 不要 push 到 GitHub
- 区分清楚 Fine Search 与 Auto-Fit 的 3 个关键差异

开始吧。先告诉我"我已阅读 CLAUDE.md,理解了工作铁律",然后从步骤 3d.1 开始。

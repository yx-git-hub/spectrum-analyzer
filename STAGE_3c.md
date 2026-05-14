# 阶段 3c:迁移 Auto-Fit 厚度粗扫到后端(核心 IP)

请先完整阅读项目根目录下的 CLAUDE.md,理解工作铁律和阶段划分。
然后查看 CLAUDE.md 中的"当前进度"清单,确认阶段 1、2、3a、3b 已勾选完成。
阅读完后向我确认你已理解,然后再开始执行下面的任务。

我们现在开始 **阶段 3c:迁移 Auto-Fit 厚度粗扫到后端**。

**这是整个项目最重要的一步**:Auto-Fit 是核心 IP,包含贪心扫描循环、
代价函数和领域 know-how(如波长 ≥ 700nm 才计入惩罚)。迁移后,
前端将不再持有任何核心拟合逻辑。

## 本阶段目标

把 `autoFitThickness` 函数(原行 2135)拆成两部分:
- **后端**:整个扫描循环 + 代价函数 + 返回最佳厚度信息(`POST /api/auto-fit`)
- **前端**:仅保留 UI 触发 + 调 API + 调用现有 `_showAutofitModal` 弹窗展示结果
  (弹窗本身不动,只是数据来源从本地变成 API 返回)

完成后,前端能彻底删除:
- `nearestNeighborMatch` (留在前端的最后一个匹配函数)
- `findPeaks` 在 Auto-Fit 中的调用(本地 detection 仅 runDetection('exp')
  还在用,后续阶段 3d/4 处理)

## 本阶段不做什么

- 不动 `runFineSearch`(阶段 3d 才改)
- 不动 `_showAutofitModal`(它是 UI 渲染,保持原样)
- 不删除 `nearestNeighborMatch` 函数定义(阶段 4 统一清理)
- 不动 UI 视觉、不动 modal 弹窗的样式和内容

## 严格遵守 — 核心 know-how 等价点

后端 Python 实现**必须与原 JS 数值完全等价**。特别注意以下 8 个细节:

1. **谷检测**:对 `-y` 调 find_peaks,height 参数取负
   (`height: p.height !== 0 ? -p.height : null`)
2. **峰检测**:对 `+y` 调 find_peaks,height 取原值(0 时传 null)
3. **distance 单位换算**:
   `distSamps = max(1, round(distanceNm / spacing))`,
   `spacing = (wls[-1] - wls[0]) / (len(wls) - 1)`
4. **prominence == 0** 时传 None 给 scipy
5. **代价函数**(原行 2202-2236)严格按以下逻辑:
   - 对每个匹配对 `pr`(从 vm.pairs 和 pm.pairs):
     - 若 `pr.diff < rmseThreshold`:`matchedSq += pr.diff²`, `matchedN += 1`
     - 若 `pr.diff >= rmseThreshold`:`matchedSq += rmseThreshold²`, `matchedN += 1`
       (注意:**仍计入 matchedN**),并把 `pr.gen` 加入 extraUnmatchedGen,
       `pr.exp` 加入 extraUnmatchedExp
   - `rmse = sqrt(matchedSq / matchedN) if matchedN > 0 else 9999`
   - `matchRatio = matchedN / expFeatureCount if expFeatureCount > 0 else 1`
6. **未匹配项惩罚**:对以下集合的并集统计波长 ≥ 700 的项个数:
   `vm.unmatchedGen + extraUnmatchedGen + vm.unmatchedExp + extraUnmatchedExp
    + pm.unmatchedGen + pm.unmatchedExp`
   (注意:vm 的 extraUnmatched 加进去,但 pm 的 extra 不在这个并集里——
    原 JS 行 2228-2231 就是这么写的,**严格按原代码**)
7. **totalCost 公式**:
   ```
   matchedN > 0:
     totalCost = rmse + penaltyVal × unmatchedCount + penaltyVal × (1 - matchRatio)
   matchedN == 0:
     totalCost = 9999
   ```
8. **bestIdx 选择标准**:`totalCost 最小`(注意:**不是 RMSE 最小**,
   这是 Auto-Fit 与 Fine Search 的关键差异)

## 详细步骤

### 步骤 3c.1:Git 提交

提示我执行:
```bash
git add -A
git status
git commit -m "Stage 3b complete: /api/match with greedy + DP algorithms"
```

### 步骤 3c.2:在后端 algorithms/ 下新建 fitting.py

文件:`backend/app/algorithms/fitting.py`

实现核心函数:

```python
import numpy as np
from typing import Literal
from .detection import apply_gap_mask
from .matching import nearest_neighbor_match
from scipy.signal import find_peaks


def detect_features_for_spectrum(
    spectrum: np.ndarray,
    wavelengths: np.ndarray,
    detection: dict,
    gaps: list,
    detect_valleys: bool,
    detect_peaks: bool,
) -> tuple[list[float], list[float]]:
    """Run findPeaks on a single spectrum, returning (valley_wls, peak_wls).
    Reuses logic from algorithms/detection.py.

    Returns lists of wavelengths (not indices).
    """
    # ...


def compute_cost_for_match(
    vm: dict,
    pm: dict,
    rmse_threshold: float,
    penalty: float,
    exp_feature_count: int,
) -> tuple[float, float, int]:
    """Compute total cost for one matching result.

    Returns: (total_cost, rmse, matched_count)

    Implementation MUST match the original JS lines 2202-2236 exactly,
    including the 'wl >= 700' rule for unmatched_count.
    """
    matched_sq = 0.0
    matched_n = 0
    extra_unmatched_gen = []
    extra_unmatched_exp = []

    for pr in vm["pairs"]:
        if pr["diff"] < rmse_threshold:
            matched_sq += pr["diff"] ** 2
        else:
            matched_sq += rmse_threshold ** 2
            extra_unmatched_gen.append(pr["gen"])
            extra_unmatched_exp.append(pr["exp"])
        matched_n += 1
    for pr in pm["pairs"]:
        if pr["diff"] < rmse_threshold:
            matched_sq += pr["diff"] ** 2
        else:
            matched_sq += rmse_threshold ** 2
            extra_unmatched_gen.append(pr["gen"])
            extra_unmatched_exp.append(pr["exp"])
        matched_n += 1

    if matched_n == 0:
        return 9999.0, float("nan"), 0

    rmse = (matched_sq / matched_n) ** 0.5
    match_ratio = matched_n / exp_feature_count if exp_feature_count > 0 else 1.0

    # Count unmatched items with wavelength >= 700 nm
    # NOTE: per original JS line 2228-2231, only vm.extraUnmatched is added,
    # not pm.extraUnmatched. Keep this exactly as JS does.
    unmatched_count = 0
    for wl in vm["unmatchedGen"]:
        if wl >= 700: unmatched_count += 1
    for wl in extra_unmatched_gen:
        if wl >= 700: unmatched_count += 1
    for wl in vm["unmatchedExp"]:
        if wl >= 700: unmatched_count += 1
    for wl in extra_unmatched_exp:
        if wl >= 700: unmatched_count += 1
    for wl in pm["unmatchedGen"]:
        if wl >= 700: unmatched_count += 1
    for wl in pm["unmatchedExp"]:
        if wl >= 700: unmatched_count += 1

    total_cost = rmse + penalty * unmatched_count + penalty * (1.0 - match_ratio)
    return total_cost, rmse, matched_n


def auto_fit_thickness(
    wavelengths: np.ndarray,
    spectra: np.ndarray,           # shape (n_spec, n_wl)
    thicknesses: np.ndarray,
    exp_valleys: list[float],
    exp_peaks: list[float],
    mode: Literal["valleys", "peaks", "both"],
    detection: dict,
    gaps: list,
    rmse_threshold: float,
    penalty: float,
) -> dict:
    """Scan all spectra, return best thickness index and per-thickness costs.

    Equivalent to the inner loop of original JS autoFitThickness (lines 2176-2246).

    Returns:
        {
            "bestIdx": int,
            "bestThickness": float,
            "bestRmse": float | None,
            "thicknesses": list[float],
            "totalCosts": list[float],
            "rmseArr": list[float | None],     # None where matchedN was 0
            "bestValleyMatch": {...},          # the vm dict at bestIdx
            "bestPeakMatch": {...},            # the pm dict at bestIdx
            "mode": str,
            "expFeatureCount": int,
        }
    """
    empty_match = {"pairs": [], "unmatchedGen": [], "unmatchedExp": []}
    n_spec = len(spectra)
    exp_feat_count = (len(exp_valleys) if mode in ("valleys", "both") else 0) \
                   + (len(exp_peaks)   if mode in ("peaks",   "both") else 0)

    total_costs = np.full(n_spec, np.inf)
    rmse_arr = np.full(n_spec, np.nan)
    best_cost = float("inf")
    best_idx = 0
    best_vm = empty_match
    best_pm = empty_match

    for i in range(n_spec):
        gen_valleys, gen_peaks = detect_features_for_spectrum(
            spectra[i], wavelengths, detection, gaps,
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

        if total_cost < best_cost:
            best_cost = total_cost
            best_idx = i
            best_vm = vm
            best_pm = pm

    best_rmse = float(rmse_arr[best_idx]) if not np.isnan(rmse_arr[best_idx]) else None

    return {
        "bestIdx": int(best_idx),
        "bestThickness": float(thicknesses[best_idx]),
        "bestRmse": best_rmse,
        "thicknesses": [float(t) for t in thicknesses],
        "totalCosts": [float(c) for c in total_costs],
        "rmseArr": [None if np.isnan(r) else float(r) for r in rmse_arr],
        "bestValleyMatch": best_vm,
        "bestPeakMatch": best_pm,
        "mode": mode,
        "expFeatureCount": exp_feat_count,
    }
```

**实现要点**:
- `detect_features_for_spectrum`:复用 `detection.py` 里的 apply_gap_mask,
  但要分别对原信号和反转信号调 find_peaks(参考原 JS 行 2180-2194)
- `compute_cost_for_match`:严格按原 JS,**不要"优化"或"简化"**
- `auto_fit_thickness`:循环结构与原 JS 行 2176-2246 对应

### 步骤 3c.3:在 schemas.py 中新增类型

```python
class AutoFitRequest(BaseModel):
    datasetId: str
    expValleys: list[float]
    expPeaks: list[float]
    mode: Literal["valleys", "peaks", "both"] = "valleys"
    penalty: float = 100.0
    rmseThreshold: float = 30.0
    detection: DetectionParams
    gaps: list[GapInterval] = []
```

**关键**:请求体里只传 `datasetId`(字符串),不传完整 spectra。
后端从已加载的数据集读取 spectra,这是为什么前面阶段要建立数据集缓存。
**这一点至关重要——它把数据集真正保留在服务端**。

### 步骤 3c.4:在 main.py 中新增端点

```python
@app.post("/api/auto-fit")
def auto_fit(req: AutoFitRequest):
    from .algorithms.fitting import auto_fit_thickness
    from .datasets import get_dataset

    try:
        ds = get_dataset(req.datasetId)
    except KeyError:
        raise HTTPException(404, f"Dataset '{req.datasetId}' not found")

    result = auto_fit_thickness(
        wavelengths=ds["wavelengths"],
        spectra=ds["spectra"],
        thicknesses=ds["thicknesses"],
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

### 步骤 3c.5:用 Swagger UI 自测后端

重启后端,在 Swagger 上测试 `/api/auto-fit`,用一组合理参数。
确认能返回 `bestIdx`、`bestThickness`、`totalCosts`(长度等于数据集 spectra 数)、
`bestValleyMatch.pairs`(应有几个匹配对)。

如果跑不通先调试后端,**不要往前端走**。

### 步骤 3c.6:前端 API 层增加 autoFit 调用

在 `frontend/spectrum_analyzer.html` 的 `API` 对象中新增:

```javascript
autoFit(body) { return this._call('POST', '/api/auto-fit', body); },
```

### 步骤 3c.7:确保前端有 datasetId

前端的 `AppState.csv` 现在通过 `onDatasetChange` 加载,但**可能没存** dataset id。
检查 `onDatasetChange` 函数(阶段 2 加的),确认有这一行:
```javascript
AppState.csv.datasetId = id;
```
如果没有,加上。

如果 AppState.csv 没有 datasetId 字段定义,在 AppState 初始化处加:
```javascript
csv: { ..., datasetId: null, ... }
```

报告 AppState.csv 当前的完整字段结构给我,等我确认。

### 步骤 3c.8:改造前端 autoFitThickness 函数

找到原 `autoFitThickness` 函数(约行 2135)。**整个函数体替换为**:

```javascript
async function autoFitThickness() {
  const { thicknesses, datasetId } = AppState.csv;

  const modeEl = document.querySelector('input[name="autofit-mode"]:checked');
  const mode   = modeEl ? modeEl.value : 'valleys';

  if (!datasetId) { setStatus('Auto-fit: select a dataset first'); return; }

  // Auto-detect exp features before fitting (still client-driven detection call)
  setStatus('Auto-detecting features before fitting…');
  await runDetection('exp');

  const expValleys = AppState.valleys.exp;
  const expPeaks   = AppState.peaks.exp;

  if ((mode === 'valleys' || mode === 'both') && !expValleys.length) {
    setStatus('Auto-fit: no experimental valleys detected — adjust detection params'); return;
  }
  if ((mode === 'peaks' || mode === 'both') && !expPeaks.length) {
    setStatus('Auto-fit: no experimental peaks detected — adjust detection params'); return;
  }

  const p          = AppState.detection.gen;
  const penaltyVal = parseFloat(document.getElementById('autofit-penalty').value) || 100;
  const rmseThreshold = parseFloat(document.getElementById('autofit-max-diff').value) || 30;
  const gaps       = AppState.gapMask.intervals;

  showLoading(`Auto-fitting on server… ${thicknesses.length} spectra`);

  try {
    const r = await API.autoFit({
      datasetId,
      expValleys,
      expPeaks,
      mode,
      penalty: penaltyVal,
      rmseThreshold,
      detection: {
        prominence:    p.prominence,
        distanceNm:    p.distanceNm,
        height:        p.height,
        detectValleys: p.detectValleys,
        detectPeaks:   p.detectPeaks,
      },
      gaps,
    });

    hideLoading();

    AppState.csv.selectedIdx = r.bestIdx;
    document.getElementById('thickness-slider').value = r.bestIdx;
    document.getElementById('thickness-num').value    = r.bestThickness;
    autoReSmooth('gen');
    await runDetection('gen');
    updatePlots();

    const validMatchedDisp = r.bestValleyMatch.pairs.filter(p => p.diff < rmseThreshold).length
                           + r.bestPeakMatch.pairs.filter(p => p.diff < rmseThreshold).length;
    const bestRmseDisp = r.bestRmse == null ? 'N/A' : r.bestRmse.toFixed(2);
    setStatus(`Auto-fit: ${r.bestThickness.toFixed(1)} nm  |  RMSE = ${bestRmseDisp} nm  |  valid matched ${validMatchedDisp}/${r.expFeatureCount}`);

    _coarseResult = {
      thicknesses: r.thicknesses,
      rmseArr:     r.rmseArr.map(v => v == null ? NaN : v),
    };

    const _fsS = document.getElementById('fs-start');
    const _fsE = document.getElementById('fs-end');
    if (_fsS) _fsS.value = Math.max(0, +(r.bestThickness - 10).toFixed(2));
    if (_fsE) _fsE.value = +(r.bestThickness + 10).toFixed(2);
    updateFsCountHint();

    _showAutofitModal(
      r.thicknesses,
      r.totalCosts,
      r.rmseArr.map(v => v == null ? NaN : v),
      null,
      r.bestIdx,
      r.bestValleyMatch,
      r.bestPeakMatch,
      r.mode,
      null
    );
  } catch (err) {
    hideLoading();
    setStatus('Auto-fit failed: ' + err.message);
  }
}
```

**注意**:
- 函数变成 async
- `runDetection('exp')` 加 await(它在阶段 2 已变 async)
- `runDetection('gen')` 加 await,确保 detection 完成后再 updatePlots
- `_showAutofitModal` **完全不动**,只是数据来源变了

### 步骤 3c.9:验证 _showAutofitModal 接口契约

`_showAutofitModal` 期望接收的参数(从原 JS 行 2272 调用方看):
```
_showAutofitModal(thicknesses, totalCosts, rmseArr, null, bestIdx, valleyMatch, peakMatch, mode, null)
```

后端返回的字段名要严格匹配:
- `r.thicknesses` (list of float) ✓
- `r.totalCosts` (list of float) ✓
- `r.rmseArr` (list of float | null) — **前端传给 modal 之前要把 null 转 NaN**
  (原 JS 用 Float64Array.fill(NaN))
- `r.bestIdx` ✓
- `r.bestValleyMatch` (含 pairs / unmatchedGen / unmatchedExp) ✓
- `r.bestPeakMatch` ✓
- `r.mode` ✓

如果有字段名不一致,在前端做转换。

### 步骤 3c.10:联调测试

提示我:
1. 重启后端
2. 刷新前端
3. 选 MoS₂ 数据集
4. 加载实验 Excel
5. 在 Generated 侧调一组合理的 detection 参数(prominence、distance、gaps)
6. 在 Auto-Fit 区选 mode、设 penalty、设 max diff
7. 点 Auto-Fit 按钮
8. F12 Network 看到 1 次 `/api/auto-fit` 200(请求体应很小,只有几 KB)
9. 弹窗出现,显示 best thickness、RMSE、cost 曲线、匹配对表格

### 步骤 3c.11:**关键** — 数值等价性验证

打开 `spectrum_analyzer_backup.html` 和当前版本,**用完全相同的**:
- MoS₂ 数据集
- 实验 Excel 文件
- detection 参数(prominence、distance、height、gaps)
- Auto-Fit mode、penalty、max diff

对比两个版本的 Auto-Fit 结果:
- `bestThickness` 必须完全一致(不允许差任何小数位)
- `bestRmse` 误差 < 0.01 nm
- 弹窗里的 cost 曲线形状应完全重合
- 匹配对表格里的所有 pairs 应完全一致

如果有任何不一致,**立即停止并报告**,逐项排查代价函数的 8 个细节。

### 步骤 3c.12:汇报结果

告诉我:
- fitting.py 实现了哪些函数
- Swagger 自测 `/api/auto-fit` 返回的 bestThickness 和 bestRmse
- 前端 autoFitThickness 改造的行号范围
- 数值等价性验证结果(bestThickness 是否完全一致)

然后停下来,**等我手动验证**全部"成功标志"。

## 阶段 3c 成功标志(用户验证清单)

- [ ] `backend/app/algorithms/fitting.py` 存在,3 个函数
- [ ] `backend/app/schemas.py` 中新增 AutoFitRequest
- [ ] `backend/app/main.py` 中新增 `/api/auto-fit` 端点
- [ ] Swagger UI 上 `/api/auto-fit` 测试通过
- [ ] AppState.csv.datasetId 在选数据集后正确设置
- [ ] 前端 autoFitThickness 是 async,调 API.autoFit,
      请求体不含完整 spectra,只含 datasetId
- [ ] Auto-Fit 弹窗仍然正常显示,内容与原版一致
- [ ] **关键**:对同一组输入,bestThickness 与原版完全一致
- [ ] **关键**:对同一组输入,bestRmse 与原版一致(误差 < 0.01 nm)

我验证通过后会说"阶段 3c 完成",再进入阶段 3d(`/api/fine-search`,最后一个端点)。

## 工作纪律

- 每个步骤完成后**停下来汇报**,等我说"继续"再做下一步
- 不要"优化"代价函数(必须与原 JS 字字对应)
- 不要碰 `_showAutofitModal`、`runFineSearch`
- 不要 push 到 GitHub
- 8 个 know-how 等价点是核心,有任何疑问先问我

开始吧。先告诉我"我已阅读 CLAUDE.md,理解了工作铁律",然后从步骤 3c.1 开始。

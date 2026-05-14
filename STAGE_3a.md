# 阶段 3a:迁移 Savitzky-Golay 滤波到后端

请先完整阅读项目根目录下的 CLAUDE.md,理解工作铁律和阶段划分。
然后查看 CLAUDE.md 中的"当前进度"清单,确认阶段 1、2 已勾选完成。
阅读完后向我确认你已理解,然后再开始执行下面的任务。

我们现在开始 **阶段 3a:迁移 Savitzky-Golay 滤波到后端**。

## 本阶段目标

把分段 Savitzky-Golay 平滑算法从前端 JS 迁移到 Python 后端。
完成后:
- 后端新增 `POST /api/smooth` 端点
- 前端 `applySmoothing(which)` 和 `autoReSmooth(which)` 改成调后端
- **前后端数值结果必须等价**(误差 < 1e-6)

## 本阶段不做什么

- 不动 `/api/smooth` 以外的 API
- 不动 `runDetection`(阶段 2 已改)
- 不动 `applySmoothing` 和 `autoReSmooth` 以外的前端函数
- 不动后端 detection 模块
- 不动 UI 视觉

## 严格遵守

- Python 实现要与原 JS 数值等价。关键细节:
  - `windowLen > n` 时:取 `n` 或 `n-1` 使其为奇数(scipy 行为不同,需特殊处理)
  - `windowLen` 必须是奇数,偶数减 1
  - `windowLen < 3` 时强制为 3
  - `polyOrder >= windowLen` 时,polyOrder 取 `windowLen - 1`
  - 边界处理:左右各 hw 个点用对窗口内 `polyfit` 做外推(原 JS 用 polyFit + polyEval)
- 分段逻辑:相邻 index 参数相同时合并为一段,对该段调用 SG;未覆盖区域用默认
  `(window=51, poly=3)`
- 用 scipy.signal.savgol_filter 不能完全替代——边界处理方式不同。
  必须自己实现以保证与原 JS 完全一致

## 详细步骤

### 步骤 3a.1:Git 提交

提示我执行:
```bash
git add -A
git status
git commit -m "Stage 2 complete: dataset selector + /api/detect integration"
```

### 步骤 3a.2:在后端 algorithms/ 下新建 smoothing.py

文件:`backend/app/algorithms/smoothing.py`

实现 3 个函数:

```python
import numpy as np

def savgol_kernel(window_len: int, poly_order: int) -> np.ndarray:
    """Compute SG convolution kernel for interior points.
    Equivalent to the original JS _computeSgKernel function."""
    # ...

def savgol_filter_jslike(y: np.ndarray, window_len: int, poly_order: int) -> np.ndarray:
    """Apply Savitzky-Golay filter with the same boundary handling as the
    original JavaScript implementation in spectrum_analyzer.html lines 1764-1795.

    Boundary handling:
    - Interior points (hw <= i < n-hw): direct convolution with kernel
    - Left edge (0 <= i < hw): polyfit on first window_len points, evaluate at i-hw
    - Right edge (n-hw <= i < n): polyfit on last window_len points, evaluate at i-(n-1-hw)
    """
    # ...

def apply_segmented_sg(y: np.ndarray, wavelengths: np.ndarray, segments: list) -> np.ndarray:
    """Apply SG with different (window, poly) per wavelength range.
    Default for uncovered regions: (window=51, poly=3).

    Equivalent to original JS applySegmentedSG (lines 1799-1830).

    segments: list of dicts with keys wlMin, wlMax, window, poly
              (wlMin/wlMax may be None for open-ended ranges)
    """
    # ...
```

**实现要点**:
- `savgol_kernel`: 用 `numpy.linalg.pinv` 计算伪逆矩阵,与原 JS 用 Gauss-Jordan 等价
- `savgol_filter_jslike`: 严格按原 JS 行 1764-1795 的步骤实现
  - 先做参数 sanitise(window_len/poly_order 边界处理)
  - 内部点用 np.convolve 或手动循环
  - 边界用 np.polyfit + np.polyval(注意 numpy.polyfit 返回的系数顺序与
    原 JS polyFit 是**相反**的:numpy 是 [高次, ..., 低次],
    JS 是 [c0, c1, ...c_p],写代码时要 reverse)
- `apply_segmented_sg`: 按相邻参数相同的 chunk 切分,对每个 chunk 调 `savgol_filter_jslike`

### 步骤 3a.3:在后端 schemas.py 中新增类型

```python
class SmoothingSegment(BaseModel):
    wlMin: float | None = None
    wlMax: float | None = None
    window: int = 51
    poly: int = 3

class SmoothRequest(BaseModel):
    y: list[float]
    wavelengths: list[float]
    segments: list[SmoothingSegment]

class SmoothResponse(BaseModel):
    smoothed: list[float]
```

### 步骤 3a.4:在 main.py 中新增端点

```python
@app.post("/api/smooth", response_model=SmoothResponse)
def smooth_spectrum(req: SmoothRequest):
    from .algorithms.smoothing import apply_segmented_sg
    y = np.asarray(req.y, dtype=np.float64)
    wls = np.asarray(req.wavelengths, dtype=np.float64)
    segs = [s.model_dump() for s in req.segments]
    smoothed = apply_segmented_sg(y, wls, segs)
    return {"smoothed": smoothed.tolist()}
```

注意 `import numpy as np` 在 main.py 顶部如果还没有就加上。

### 步骤 3a.5:用 Swagger UI 自测后端

重启后端,在 http://127.0.0.1:8000/docs 测试 `/api/smooth`,
用一段简单数据(15 点正弦带噪声),确认能返回平滑结果:

```json
{
  "y": [0.1, 0.8, 0.3, 1.0, 0.4, 1.1, 0.5, 1.2, 0.6, 1.1, 0.5, 1.0, 0.4, 0.8, 0.3],
  "wavelengths": [400, 425, 450, 475, 500, 525, 550, 575, 600, 625, 650, 675, 700, 725, 750],
  "segments": [{"wlMin": null, "wlMax": null, "window": 5, "poly": 2}]
}
```

应返回长度 15 的 smoothed 数组。

### 步骤 3a.6:前端 API 层增加 smooth 调用

在 `frontend/spectrum_analyzer.html` 的 `API` 对象中新增方法:

```javascript
smooth(body)  { return this._call('POST', '/api/smooth', body); },
```

### 步骤 3a.7:改造前端 applySmoothing 函数

找到原 `applySmoothing` 函数(约行 2072)。改造为:

```javascript
async function applySmoothing(which) {
  const dataState = which === 'gen' ? AppState.csv : AppState.excel;
  const sm  = AppState.smoothing[which];
  const idx = dataState.selectedIdx;
  const rawY = dataState.spectra[idx];
  const wls  = dataState.wavelengths;

  if (!rawY || !rawY.length) {
    setStatus(`No ${which === 'gen' ? 'CSV' : 'experiment'} data loaded yet`);
    return;
  }

  showLoading('Applying Savitzky-Golay filter...');
  try {
    const r = await API.smooth({
      y: rawY,
      wavelengths: wls,
      segments: sm.segments,
    });
    sm.cached = r.smoothed;
    sm.cachedIdx = idx;
    updatePlots();
    setStatus(`${which === 'gen' ? 'Generated' : 'Experimental'} spectrum smoothed ` +
              `(${sm.segments.length} segment${sm.segments.length > 1 ? 's' : ''})`);
  } catch (err) {
    setStatus('Smoothing failed: ' + err.message);
  } finally {
    hideLoading();
  }
}
```

### 步骤 3a.8:改造前端 autoReSmooth 函数

`autoReSmooth` 是滑块拖动时自动重新平滑的,调用频繁,**不显示 loading**
(不然滑块拖动会卡 loading 闪屏)。改造为:

```javascript
async function autoReSmooth(which) {
  const dataState = which === 'gen' ? AppState.csv : AppState.excel;
  const sm = AppState.smoothing[which];
  if (!sm.cached) return;   // no smooth applied yet
  const idx = dataState.selectedIdx;
  const rawY = dataState.spectra[idx];
  const wls  = dataState.wavelengths;
  if (!rawY || !rawY.length) return;

  try {
    const r = await API.smooth({
      y: rawY,
      wavelengths: wls,
      segments: sm.segments,
    });
    sm.cached = r.smoothed;
    sm.cachedIdx = idx;
  } catch (err) {
    console.warn('autoReSmooth failed:', err);
  }
}
```

### 步骤 3a.9:处理调用方的 async 传染

`autoReSmooth` 和 `applySmoothing` 现在是 async 了。
检查所有调用它们的位置(grep 整个 HTML):

- 行 953:`autoReSmooth('gen');` — `onThicknessSlide` 中,fire-and-forget OK
- 行 969:`autoReSmooth('gen');` — `onThicknessNumInput` 中,fire-and-forget OK
- 行 1115:`autoReSmooth('exp');` — Excel 索引变化,fire-and-forget OK
- 行 2253:`autoReSmooth('gen');` — autoFitThickness 中(阶段 3c 才改,本阶段不动)
- 行 2672:`autoReSmooth('gen');` — runFineSearch 中(阶段 3d 才改,本阶段不动)
- 按钮 `onclick="applySmoothing('gen')"`:fire-and-forget OK

**注意**:对于行 2253 和 2672,如果当前是同步函数,改成 `await autoReSmooth('gen');`
会导致函数变 async,牵连过多。**本阶段暂时保持原样**(fire-and-forget),
后续阶段 3c/3d 改 autoFitThickness 和 runFineSearch 时再统一处理。

### 步骤 3a.10:前后端联调测试

提示我:
1. 重启后端(uvicorn 应该 reload 了,但建议手动 Ctrl+C 重启确保新代码生效)
2. 刷新前端 HTML
3. 选择 MoS₂ 数据集
4. 在 Generated Smoothing 区域设置一段 SG(默认 window=51, poly=3)
5. 点击 "Apply" 按钮
6. F12 → Network 看到 `POST /api/smooth` 状态 200
7. 图表上原始光谱被平滑后的曲线覆盖

### 步骤 3a.11:**关键** — 数值等价性验证

让我打开 `spectrum_analyzer_backup.html`(原始版本,我备份在文件夹外),
和当前改造版本对同一份 MoS₂ 数据的同一个厚度,**用相同的 SG 参数**应用平滑,
**对比两条平滑曲线**:

- 视觉应完全重合
- 如果在图上看不出差异,通过 F12 Console 比对几个数据点的具体数值,
  误差应 < 1e-6

如果存在显著差异(肉眼可见),**立即报告并停下来**,排查问题。

### 步骤 3a.12:汇报结果

告诉我:
- 后端 smoothing.py 实现了哪几个函数,关键边界处理是怎么做的
- Swagger 自测 `/api/smooth` 返回值
- 前端改造的具体行号范围
- 数值等价性验证是否通过(肉眼/数值对比)

然后停下来,**等我手动验证**全部"成功标志"。

## 阶段 3a 成功标志(用户验证清单)

- [ ] `backend/app/algorithms/smoothing.py` 存在,3 个函数实现完整
- [ ] `backend/app/schemas.py` 中新增 SmoothingSegment、SmoothRequest、SmoothResponse
- [ ] `backend/app/main.py` 中新增 /api/smooth 端点
- [ ] Swagger UI 上 /api/smooth 测试返回合理结果
- [ ] 前端 applySmoothing 是 async,调 API.smooth
- [ ] 前端 autoReSmooth 是 async,调 API.smooth
- [ ] 点击 Apply 按钮 → Network 看到 /api/smooth 200
- [ ] 拖动 thickness slider → Network 看到 /api/smooth 200(且滑块流畅,不卡)
- [ ] **关键**:平滑曲线与原版完全一致(对比 backup)

我验证通过后会说"阶段 3a 完成",再进入阶段 3b(/api/match)。

## 工作纪律

- 每个步骤完成后**停下来汇报**,等我说"继续"再做下一步
- 不要一口气把所有步骤跑完
- 不要碰 autoFitThickness、runFineSearch、nearestNeighborMatch 等
  (阶段 3c/3d 才动)
- 不要 push 到 GitHub
- 数值等价性是本阶段的核心,如有任何疑问先问我

开始吧。先告诉我"我已阅读 CLAUDE.md,理解了工作铁律",然后从步骤 3a.1 开始。

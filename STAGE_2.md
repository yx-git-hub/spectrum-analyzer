# 阶段 2:前端连上后端

请先完整阅读项目根目录下的 CLAUDE.md,理解工作铁律和阶段划分。
然后查看 CLAUDE.md 中的"当前进度"清单,确认阶段 1 已勾选完成。
阅读完后向我确认你已理解,然后再开始执行下面的任务。

我们现在开始 **阶段 2:前端连上后端**。

## 本阶段目标

让 `spectrum_analyzer.html` 通过 HTTP API 从本地后端获取数据集列表,
并把 `runDetection('gen')` 函数从本地 JS 算法改成调后端 `/api/detect`。

完成后,链路验证:**前端浏览器 → fetch → 本地 FastAPI → 返回结果 → 渲染**。

## 本阶段不做什么

- 不动 `runDetection('exp')` 以外的任何 JS 算法函数
  (savgolFilter / findPeaks / nearestNeighborMatch 等保留不动,留到阶段 3-4)
- 不动 `/api/detect` 以外的 API
- 不动后端代码(后端阶段 1 已经完成,本阶段不改后端)
- 不动 Excel/CSV 实验数据上传那一块 UI(那是用户自己的实验数据,流程不变)

## 严格遵守

- 后端工作目录是 `backend/`,前端 HTML 现在还在项目根目录
  本阶段开始时**先把 `spectrum_analyzer.html` 移到 `frontend/` 目录下**
  (CLAUDE.md 里定义的最终结构)
- UI 视觉不变。下拉菜单替换 "Load CSV" 区域时,**保持原有的样式风格**
  (用相同的字体、边框、配色)
- 不要修改 `<style>` 块以外的 CSS
- 不能动 Excel/Experimental 那块 UI

## 详细步骤

### 步骤 2.1:Git 提交当前状态

提示我执行:
```bash
git add -A
git status
git commit -m "Stage 1 complete: backend skeleton with /datasets and /api/detect"
```
告诉我:"Git 已提交,准备进入阶段 2"

### 步骤 2.2:确认后端可访问

让我确认后端正在跑(`uvicorn app.main:app --reload --host 0.0.0.0 --port 8000`)。
如果没跑,提醒我用新终端启动。

curl 测试一下:
```bash
curl http://127.0.0.1:8000/datasets
```
应该返回 mos2 数据集信息。

### 步骤 2.3:移动 HTML 到 frontend 目录

```bash
mkdir frontend
git mv spectrum_analyzer.html frontend/spectrum_analyzer.html
```

(用 `git mv` 而非普通 `mv`,保留 git 历史)

### 步骤 2.4:HTML 改造 — 添加 API 调用工具层

在 `frontend/spectrum_analyzer.html` 的 `<script>` 块最顶端
(在 AppState 定义之前)加入以下代码:

```javascript
// ═══════════════════════════════════════════════════════════════
//  API Layer — all backend calls go through this object
// ═══════════════════════════════════════════════════════════════
const API = {
  base: 'http://127.0.0.1:8000',

  async _call(method, path, body) {
    const opts = {
      method,
      headers: { 'Content-Type': 'application/json' },
    };
    if (body !== undefined) opts.body = JSON.stringify(body);
    const resp = await fetch(this.base + path, opts);
    if (!resp.ok) {
      const txt = await resp.text();
      throw new Error(`API ${method} ${path}: ${resp.status} ${txt}`);
    }
    return resp.json();
  },

  listDatasets()    { return this._call('GET', '/datasets'); },
  detect(body)      { return this._call('POST', '/api/detect', body); },
};
```

### 步骤 2.5:HTML 改造 — Generated Spectra 区域换成下拉菜单

在 HTML 中找到原 "Generated Spectra (CSV)" 区块(约行 436-453,
包含 csv-path、Browse 按钮、Load 按钮、csv-info、thickness-control)。

**完全替换**整个 `<div class="load-section">` 块(只换 Generated 那块,
**不要碰下面的 "Experimental Spectra" 那块**)为下面的结构:

```html
<!-- Dataset selector -->
<div class="load-section">
  <div class="load-label">Generated Database (Server)</div>
  <div class="load-row">
    <select id="dataset-select" onchange="onDatasetChange()" style="flex:1;">
      <option value="">— Loading datasets... —</option>
    </select>
  </div>
  <div id="csv-info" class="load-info">No dataset selected</div>
  <div id="thickness-control">
    <label>Thickness:</label>
    <input type="range" id="thickness-slider" min="0" max="0" value="0"
           oninput="onThicknessSlide(this.value)">
    <input type="number" id="thickness-num" step="0.5"
           oninput="onThicknessNumInput(this.value)">
    <span id="thickness-hint" class=""></span>
  </div>
</div>
```

**关键**:
- `id="csv-info"`、`id="thickness-slider"`、`id="thickness-num"`、
  `id="thickness-hint"`、`id="thickness-control"` 这些 ID **必须保留**
  (后续代码还在用)
- 下拉菜单的 select 用与现有 CSS 兼容的样式,
  如果原 CSS 没定义 select 样式,style 里加 width:100%

### 步骤 2.6:HTML 改造 — 实现数据集加载与选择函数

在 JS 中,找一个合适的位置(建议放在原 `onCSVFileSelected` 函数附近,
约行 847),**新增**以下函数。

**不要删除原 `onCSVFileSelected` / `loadCSV` / `parseCSV` 函数**
(暂时留着,本阶段最后再清理)。

```javascript
// ─── Dataset loading from server ───

async function initDatasetList() {
  const sel = document.getElementById('dataset-select');
  try {
    setStatus('Connecting to server...');
    const datasets = await API.listDatasets();
    if (!datasets.length) {
      sel.innerHTML = '<option value="">— No datasets on server —</option>';
      setStatus('No datasets available');
      return;
    }
    sel.innerHTML = '<option value="">— Select a dataset —</option>' +
      datasets.map(d =>
        `<option value="${d.id}">${d.name} ` +
        `(${d.wavelengthRange[0]}-${d.wavelengthRange[1]} nm, ` +
        `${d.numSpectra} spectra, step ${d.thicknessStep} nm)</option>`
      ).join('');
    setStatus(`Connected — ${datasets.length} dataset(s) available`);
  } catch (err) {
    sel.innerHTML = '<option value="">— Server unreachable —</option>';
    setStatus('Cannot reach backend: ' + err.message);
  }
}

async function onDatasetChange() {
  const id = document.getElementById('dataset-select').value;
  if (!id) return;
  showLoading('Loading dataset from server...');
  try {
    // For Stage 2: fetch the dataset payload via a temporary GET /datasets/{id}/full
    // endpoint. For now we'll keep using the local arrays that the original
    // parseCSV would have populated, by calling a new /datasets/{id}/full endpoint.
    //
    // If backend doesn't expose this yet, fall back to the existing /datasets
    // metadata only and ask backend team to add the endpoint.
    const data = await fetch(`${API.base}/datasets/${id}/full`).then(r => {
      if (!r.ok) throw new Error('HTTP ' + r.status);
      return r.json();
    });

    AppState.csv.wavelengths = data.wavelengths;
    AppState.csv.spectra     = data.spectra;
    AppState.csv.thicknesses = data.thicknesses;
    AppState.csv.selectedIdx = 0;

    // Thickness slider (reuse original logic)
    const slider   = document.getElementById('thickness-slider');
    const numInput = document.getElementById('thickness-num');
    const ts = data.thicknesses;
    slider.min   = 0;
    slider.max   = ts.length - 1;
    slider.value = 0;
    const tMin = ts[0];
    const tMax = ts[ts.length - 1];
    const step = ts.length > 1 ? +(ts[1] - ts[0]).toFixed(3) : 0.5;
    numInput.min  = tMin;
    numInput.max  = tMax;
    numInput.step = step;
    numInput.value = tMin;
    document.getElementById('thickness-hint').textContent =
      `range ${tMin}–${tMax} nm, step ${step} nm`;
    document.getElementById('thickness-control').style.display = 'flex';
    document.getElementById('csv-info').textContent =
      `Loaded: ${data.spectra.length} spectra × ${data.wavelengths.length} pts  |  t = ${tMin}–${tMax} nm`;

    _coarseResult = null;
    const _fsMid = ts[Math.floor(ts.length / 2)];
    const _fsS = document.getElementById('fs-start');
    const _fsE = document.getElementById('fs-end');
    if (_fsS) _fsS.value = Math.max(ts[0], +(_fsMid - 50).toFixed(1));
    if (_fsE) _fsE.value = Math.min(ts[ts.length - 1], +(_fsMid + 50).toFixed(1));
    if (typeof updateFsCountHint === 'function') updateFsCountHint();
    updatePlots();

    hideLoading();
    setStatus(`Dataset loaded: ${id}`);
  } catch (err) {
    hideLoading();
    setStatus('Failed to load dataset: ' + err.message);
  }
}
```

### 步骤 2.7:页面启动时调用 initDatasetList

找到 HTML 中页面初始化的位置(原代码末尾应该有 `window.onload` 或在
`<script>` 末尾的初始化调用)。

如果有 `window.onload`,在其中加 `initDatasetList()`;
如果没有,在 `</script>` 之前加:
```javascript
window.addEventListener('DOMContentLoaded', () => {
  initDatasetList();
});
```

注意不要破坏原有的初始化逻辑。

### 步骤 2.8:后端新增 GET /datasets/{id}/full

(本阶段唯一允许的后端改动)

在 `backend/app/main.py` 中新增端点:

```python
@app.get("/datasets/{dataset_id}/full")
def get_dataset_full(dataset_id: str):
    """Return full dataset payload including wavelengths, spectra, thicknesses.
    Used by frontend when user selects a dataset from the dropdown."""
    from .datasets import get_dataset
    try:
        d = get_dataset(dataset_id)
    except KeyError:
        raise HTTPException(404, f"Dataset '{dataset_id}' not found")
    return {
        "id": dataset_id,
        "wavelengths": d["wavelengths"].tolist(),
        "spectra":     d["spectra"].tolist(),
        "thicknesses": d["thicknesses"].tolist(),
    }
```

记得 `from fastapi import HTTPException` 已导入。

**注意**:这个端点会传输几十 MB 数据。性能优化(改成 .npy 二进制流传输或
按需切片)留到阶段 5 部署前考虑,现在先用 JSON 跑通。

### 步骤 2.9:改造 runDetection 函数

找到 `runDetection` 函数(原约 1952 行)。

**只改 `which === 'gen'` 这一支**(也就是生成数据集那边)改成调 API。
**实验数据那一支(`which === 'exp'`)暂时不动**,留到阶段 3 一起改。

具体改法:

```javascript
async function runDetection(which) {
  // For experimental data: keep original local implementation for now
  if (which === 'exp') {
    // ... 保留原代码,从 const dataState = ... 到 setStatus(...) 不变
    return;
  }

  // For generated (server-side dataset): use backend API
  const dataState = AppState.csv;
  const sm  = AppState.smoothing.gen;
  const idx = dataState.selectedIdx;
  const rawY = dataState.spectra[idx];
  const wls  = dataState.wavelengths;

  if (!rawY || !wls.length) {
    setStatus('No dataset loaded');
    return;
  }

  // Prefer smoothed data; fall back to raw
  const y = (sm.cached && sm.cachedIdx === idx) ? sm.cached : rawY;

  const p = AppState.detection.gen;
  const gaps = AppState.gapMask.intervals;

  try {
    const r = await API.detect({
      y, wavelengths: wls,
      detection: {
        prominence:    p.prominence,
        distanceNm:    p.distanceNm,
        height:        p.height,
        detectValleys: p.detectValleys,
        detectPeaks:   p.detectPeaks,
      },
      gaps,
    });
    AppState.valleys.gen = r.valleys;
    AppState.peaks.gen   = r.peaks;
    updatePlots();
    setStatus(`Generated: ${r.valleys.length} valley${r.valleys.length !== 1 ? 's' : ''}` +
              (r.peaks.length ? `, ${r.peaks.length} peak${r.peaks.length !== 1 ? 's' : ''}` : '') +
              ` detected`);
  } catch (err) {
    setStatus('Detect failed: ' + err.message);
  }
}
```

**关键**:函数变成 `async`。所有同步调用 `runDetection('gen')` 的地方
**不用强加 `await`**(fire-and-forget 也行),但凡是依赖 `runDetection` 完成
后才能做下一步的地方要注意。

通常 `runDetection('gen')` 被这些地方调用:
- `onThicknessSlide` (slider 拖动)
- `onThicknessNumInput` (数字输入)
- 点击 Detect 按钮(onclick)
- 这些都是 UI 事件触发,fire-and-forget 完全 OK

### 步骤 2.10:启动后端 + 启动前端 + 自测

提示我:
1. 终端 A:确认后端在跑(`uvicorn ...`)
2. 双击 `frontend/spectrum_analyzer.html` 在浏览器打开
3. F12 → Console 看有没有报错
4. F12 → Network 看有没有 `/datasets` 请求(状态 200)
5. 下拉菜单应显示 "MoS₂ (...)"
6. 选择数据集 → 看到 thickness slider 出现
7. 点击 "Detect"(生成数据侧)或拖动 thickness slider → Network 出现 `/api/detect`
8. 图表上出现 valley/peak 标记

### 步骤 2.11:汇报结果

告诉我:
- HTML 已移到 frontend/ 目录
- 改了哪些位置(行号大致范围)
- 后端新增端点测试结果
- 是否有报错

然后停下来,**等我手动验证**全部"成功标志"。

## 阶段 2 成功标志(用户验证清单)

我会自己验证:
- [ ] `frontend/spectrum_analyzer.html` 存在,原位置已无该文件
- [ ] `git status` 显示 spectrum_analyzer.html 是 renamed(R)状态
- [ ] 浏览器打开 HTML 后,下拉菜单显示 MoS₂
- [ ] F12 Network 看到 GET /datasets 状态 200
- [ ] 选择数据集后,F12 Network 看到 GET /datasets/mos2/full 状态 200
- [ ] thickness slider 出现,范围与数据集元数据一致
- [ ] 拖动 slider,图表正确更新显示对应厚度的光谱
- [ ] 点 Detect(gen 侧)按钮,F12 Network 看到 POST /api/detect 状态 200
- [ ] 图表上出现 valley/peak 标记
- [ ] **关键**:检测结果与原版一致(可以打开 backup 版本对比同一个厚度的同一组检测参数)
- [ ] Experimental 上传那块 UI 没动,仍能上传 Excel(虽然检测还没接 API)

我验证通过后会说"阶段 2 完成",再进入阶段 3。

## 工作纪律

- 每个步骤完成后**停下来汇报**,等我说"继续"再做下一步
- 不要一口气把所有步骤跑完
- 任何超出本阶段任务范围的修改,**先问我**
- 不要 push 到 GitHub
- **关键**:UI 视觉效果保持一致,如果不确定改动是否影响视觉,先问我

开始吧。先告诉我"我已阅读 CLAUDE.md,理解了工作铁律",然后从步骤 2.1 开始。

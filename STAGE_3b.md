# 阶段 3b:迁移峰谷匹配算法到后端

请先完整阅读项目根目录下的 CLAUDE.md,理解工作铁律和阶段划分。
然后查看 CLAUDE.md 中的"当前进度"清单,确认阶段 1、2、3a 已勾选完成。
阅读完后向我确认你已理解,然后再开始执行下面的任务。

我们现在开始 **阶段 3b:迁移峰谷匹配算法到后端**。

## 本阶段目标

把两种匹配算法从前端 JS 迁移到 Python 后端:
- **贪心最近邻匹配**(`nearestNeighborMatch`,原行 2109):用于 Auto-Fit/Fine Search
- **DP 顺序匹配**(`matchValleys`,原行 1298):用于表格展示(Δ 列)

完成后:
- 后端新增 `POST /api/match` 端点,通过 `algorithm` 参数选择两种算法
- 前端 `updateValleyTable` 改成调后端
- `nearestNeighborMatch` 暂时保留(阶段 3c 才删,留给 autoFitThickness 用)

## 本阶段不做什么

- 不删除前端的 `nearestNeighborMatch` 函数(留给阶段 3c 处理)
- 不动 `autoFitThickness`、`runFineSearch`
- 不动 UI 视觉、不动表格的列结构和样式
- 不动其他端点

## 严格遵守

- **两种算法的输出格式不同**,要在端点中区分:
  - 贪心:`{pairs: [{gen, exp, diff}], unmatchedGen: [...], unmatchedExp: [...]}`
  - DP:`{pairs: [[gen_or_null, exp_or_null], ...]}` (顺序保留,unmatched 用 null 表示)
- DP 算法的 SKIP 常数严格等于 `1e9`,与原 JS 一致
- 算法实现要数值等价:对同一对输入,贪心和 DP 各自的结果与原 JS 完全一致

## 详细步骤

### 步骤 3b.1:Git 提交

提示我执行:
```bash
git add -A
git status
git commit -m "Stage 3a complete: /api/smooth + Savitzky-Golay migrated to backend"
```

### 步骤 3b.2:在后端 algorithms/ 下新建 matching.py

文件:`backend/app/algorithms/matching.py`

实现 2 个函数:

```python
import numpy as np
from typing import Union

def nearest_neighbor_match(gen: list[float], exp: list[float]) -> dict:
    """Greedy nearest-neighbor matching.

    Equivalent to the original JavaScript nearestNeighborMatch (line 2109).

    Algorithm:
    1. Compute |gen[g] - exp[e]| for all pairs
    2. Sort pairs by distance ascending
    3. Greedily pick each pair if neither side is used yet

    Returns:
        {
            "pairs": [{"gen": float, "exp": float, "diff": float}, ...],
            "unmatchedGen": [float, ...],
            "unmatchedExp": [float, ...]
        }
    """
    # ...


def dp_sequence_match(gen: list[float], exp: list[float], skip_penalty: float = 1e9) -> list[list]:
    """Sequence-preserving DP matching with skip penalty.

    Equivalent to the original JavaScript matchValleys (line 1298).

    Each row in the result is a pair [gen_or_None, exp_or_None]:
    - both present  → matched pair
    - gen only      → gen unmatched (skipped)
    - exp only      → exp unmatched (skipped)

    The pairs preserve the original order (NOT sorted by wavelength
    explicitly, but since gen/exp are usually pre-sorted, output will be too).

    Returns:
        list of [gen_or_None, exp_or_None]
    """
    # ...
```

**实现要点**:

- `nearest_neighbor_match`:
  - 用 numpy.meshgrid 一次性算所有 (g, e) 距离
  - 用 np.argsort + np.unravel_index 拿到按距离排序的索引对
  - 用两个 set 记录已用的 g/e 索引
  - 注意:**不要排序 gen 或 exp**,要保留原始顺序;输出的 pair 里的 gen/exp 值
    是原始波长,不是索引

- `dp_sequence_match`:
  - dp 矩阵 (m+1) × (n+1),初始化为 inf,dp[0][0] = 0
  - 三种转移:i→i+1(跳过 gen[i],代价 SKIP),j→j+1(跳过 exp[j],代价 SKIP),
    (i,j)→(i+1,j+1)(配对 gen[i] 和 exp[j],代价 |gen[i]-exp[j]|)
  - 回溯时按原 JS 行 1322-1334 的顺序判断
  - **关键**:回溯结果用 `pairs.unshift(...)` 是从末尾倒序插入,
    Python 用 `pairs.insert(0, ...)` 或最后 reverse,效果等价

### 步骤 3b.3:在 schemas.py 中新增类型

```python
from typing import Literal

class MatchRequest(BaseModel):
    gen: list[float]
    exp: list[float]
    algorithm: Literal["greedy", "dp"] = "greedy"

class GreedyPair(BaseModel):
    gen: float
    exp: float
    diff: float

class GreedyMatchResponse(BaseModel):
    pairs: list[GreedyPair]
    unmatchedGen: list[float]
    unmatchedExp: list[float]

class DPMatchResponse(BaseModel):
    # Each pair is [gen|None, exp|None]
    pairs: list[list[float | None]]
```

### 步骤 3b.4:在 main.py 中新增端点

```python
@app.post("/api/match")
def match_features(req: MatchRequest):
    from .algorithms.matching import nearest_neighbor_match, dp_sequence_match
    if req.algorithm == "greedy":
        return nearest_neighbor_match(req.gen, req.exp)
    else:  # dp
        pairs = dp_sequence_match(req.gen, req.exp)
        return {"pairs": pairs}
```

**注意**:返回类型不固定(两种算法格式不同),所以这里不用 response_model。
前端要根据请求时指定的 algorithm 自行处理响应格式。

### 步骤 3b.5:用 Swagger UI 自测后端

重启后端,测试两种算法:

**贪心测试**:
```json
{
  "gen": [620, 740, 850],
  "exp": [625, 745, 845, 920],
  "algorithm": "greedy"
}
```
应返回 3 个 pairs(diff 都是 5),unmatchedExp 含 920。

**DP 测试**:
```json
{
  "gen": [620, 740, 850],
  "exp": [625, 745, 845, 920],
  "algorithm": "dp"
}
```
应返回 4 个 pairs:`[[620,625],[740,745],[850,845],[null,920]]`。

### 步骤 3b.6:前端 API 层增加 match 调用

在 `frontend/spectrum_analyzer.html` 的 `API` 对象中新增:

```javascript
match(body)   { return this._call('POST', '/api/match', body); },
```

### 步骤 3b.7:改造前端 updateValleyTable

找到 `updateValleyTable` 函数(原约行 1338)。

**这个函数被频繁调用**(detection 完成、滑块拖动、阈值改变等),改成 async
后要确保所有调用方都不会因此出错。先确认调用方:

```bash
grep -n updateValleyTable frontend/spectrum_analyzer.html
```

汇报所有调用位置后,等我确认再继续。

----

(等我确认后再做以下改造)

`updateValleyTable` 改造为 async,内部把原来两次 `matchValleys` 改成调 API:

```javascript
async function updateValleyTable() {
  const genV = AppState.valleys.gen, expV = AppState.valleys.exp;
  const genP = AppState.peaks.gen,   expP = AppState.peaks.exp;
  const tbody   = document.getElementById('valley-tbody');
  const statsEl = document.getElementById('vt-stats');

  const hasValleys = genV.length || expV.length;
  const hasPeaks   = genP.length || expP.length;

  if (!hasValleys && !hasPeaks) {
    tbody.innerHTML = '<tr><td colspan="4" style="color:#bbb;padding:14px;text-align:center;font-size:11px;">Run Detect on both spectra</td></tr>';
    statsEl.style.display = 'none';
    return;
  }

  const thr       = parseFloat(document.getElementById('vt-thr-green').value)  || 10;
  const thrOrange = parseFloat(document.getElementById('vt-thr-orange').value) || 20;

  // Get pairs from backend
  let valleyPairs = [], peakPairs = [];
  try {
    if (hasValleys) {
      const r = await API.match({ gen: genV, exp: expV, algorithm: 'dp' });
      valleyPairs = r.pairs;
    }
    if (hasPeaks) {
      const r = await API.match({ gen: genP, exp: expP, algorithm: 'dp' });
      peakPairs = r.pairs;
    }
  } catch (err) {
    console.warn('Match failed:', err);
    // Fallback: show empty table with error
    tbody.innerHTML = `<tr><td colspan="4" style="color:#c00;padding:14px;text-align:center;font-size:11px;">Match failed: ${err.message}</td></tr>`;
    return;
  }

  // ── Rest of function unchanged ──
  // 保留从 const vDiffs = ... 开始的所有渲染代码不变
}
```

**关键**:保留 `updateValleyTable` 内部从 `const vDiffs = ...` 开始
到函数末尾的**全部渲染代码**(可能 60-80 行),只把开头计算 pairs 那两行
替换成 await API.match 调用。

### 步骤 3b.8:处理调用方的 async 传染

`updateValleyTable` 现在是 async 了,所有调用它的地方要检查:
- 如果调用方是同步的 UI 事件回调(onclick / oninput),fire-and-forget 即可
- 如果调用方依赖它完成后才能做下一步(罕见),要加 await

报告所有调用位置和处理方式给我,等我确认。

### 步骤 3b.9:联调测试

提示我:
1. 重启后端
2. 刷新前端
3. 选 MoS₂ 数据集 → 加载实验 Excel(用你常用的实验数据)
4. 点 Detect(gen 和 exp 都点)
5. 查看 Valley/Peak 表格,F12 Network 看到 2 次 `/api/match` 200
6. 表格里的 Δ 列数值与原版应一致

### 步骤 3b.10:**关键** — 数值等价性验证

打开 `spectrum_analyzer_backup.html` 和当前版本,**用同一组实验数据,
同一个 MoS₂ 厚度,同样的检测参数**,对比 Valley 表格的:
- 配对关系是否一致(哪行哪个 gen 对哪个 exp)
- Δ 数值是否一致(应完全一致,误差 < 1e-6)

如果配对关系或 Δ 不一致,**立即停止并报告**。

### 步骤 3b.11:汇报结果

告诉我:
- matching.py 实现了哪两个函数
- Swagger 自测两种算法的返回值
- updateValleyTable 的调用位置和处理方式
- 数值等价性验证结果

然后停下来,**等我手动验证**全部"成功标志"。

## 阶段 3b 成功标志(用户验证清单)

- [ ] `backend/app/algorithms/matching.py` 存在,2 个函数
- [ ] `backend/app/schemas.py` 中新增 MatchRequest / GreedyMatchResponse / DPMatchResponse
- [ ] `backend/app/main.py` 中新增 `/api/match` 端点
- [ ] Swagger UI 上两种算法测试通过(返回格式正确)
- [ ] 前端 updateValleyTable 是 async,调 API.match
- [ ] 加载实验数据 → 点 Detect → 表格出现,Network 看到 /api/match 200
- [ ] **关键**:表格 Δ 列数值与原版完全一致

我验证通过后会说"阶段 3b 完成",再进入阶段 3c(/api/auto-fit,核心 IP)。

## 工作纪律

- 每个步骤完成后**停下来汇报**,等我说"继续"再做下一步
- 不要删除前端的 nearestNeighborMatch(留给阶段 3c)
- 不要碰 autoFitThickness、runFineSearch
- 不要 push 到 GitHub
- 数值等价性是核心,有疑问先问我

开始吧。先告诉我"我已阅读 CLAUDE.md,理解了工作铁律",然后从步骤 3b.1 开始。

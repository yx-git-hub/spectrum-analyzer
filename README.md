# Spectrum Analyzer 使用手册

一个在线的薄膜厚度反演工具，基于 Fabry-Pérot 反射谱。这份文档讲**网页怎么用**——
每一块控件代表什么、按哪个按钮做什么事。

**在线地址**：https://yx-git-hub.github.io/spectrum-analyzer/

> 第一次打开如果状态栏一直在 "Connecting to server..."，是后端在冷启动，
> 等 30 秒左右会自动连上。

---

## 1. 总览

页面分四块，从上到下：

```
┌─────────────────────────────────────────────────────────────────┐
│ ① 顶部加载区   数据集下拉 │ 实验文件 │ Auto-Fit │ Fine Search   │
├─────────────────────────────────────────────────────────────────┤
│ ② 主视图     Gen 谱图 │ Exp 谱图 │ Valley/Peak 对比表          │
├─────────────────────────────────────────────────────────────────┤
│ ③ 折叠的参数面板（默认收起，点 "Settings & Parameters" 展开）   │
├─────────────────────────────────────────────────────────────────┤
│ ④ 底部状态栏                                                    │
└─────────────────────────────────────────────────────────────────┘
```

每次想完整跑一遍，**最简单流程**：

1. **选数据集**（顶部第一栏下拉）
2. **拖入实验文件**（顶部第二栏拖拽框）
3. 看两边谱图是否合理 → 调一下"Settings & Parameters"里的检测/平滑参数
4. 按 **▲ Auto-Fit Thickness** 粗扫
5. 看弹窗里 RMSE 曲线和最佳厚度
6. （可选）按 **🔍 Fine Search** 在最佳点附近精搜

---

## 2. 顶部加载区

### 2.1 Generated Database (Server) — 选择生成谱数据集

| 控件 | 作用 |
|---|---|
| 下拉菜单 | 从服务器列表里选材料 (MoS₂ / MoO₃ / MoS₂ on MoO₃) |
| `Thickness:` 滑块 + 数字框 | 选当前显示的厚度（拖滑块或直接输数字）|
| `MoO3 thickness:` 第二个滑块 | **只在双材料模式出现**，控制第二种材料厚度 |
| 灰色提示文字 | 当前数据集的厚度范围和步长 |

**双材料模式**：选 `MoS2 on MoO3` 时会出现两个滑块；这模式下 Auto-Fit / Fine
Search 按钮会**置灰**（双材料拟合算法还没实现，目前只能用来看谱图）。

### 2.2 Experimental Spectra (Excel / CSV) — 拖入实验数据

| 控件 | 作用 |
|---|---|
| 虚线框 | 拖文件或点击选文件 (`.xlsx` / `.xls` / `.csv`) |
| `Sheet:` 下拉 | 多 sheet 的 Excel 选哪一个（单 sheet 时不显示）|
| `Format:` 下拉 | 数据布局：<br>• **Format B**：每行一条谱，最后一行是波长（默认）<br>• **Format A**：第 0 列波长，其它列每列一条谱 |
| `Spectrum index:` 输入框 + ◀ ▶ | 在多条实验谱之间切换 |

实验文件里如果有空单元格或非数字，加载器会**自动对齐到波长行 + 用最近邻值
填补**，不会因为一个空格就报错。

### 2.3 ▲ Auto-Fit Thickness — 厚度粗扫

把实验谱的波谷/波峰位置和生成谱库里所有厚度的波谷位置匹配，找出代价最小的
那个厚度。

| 控件 | 作用 |
|---|---|
| `Match mode` | 匹配什么特征：仅波谷 / 仅波峰 / 两者都用 |
| `Unmatch penalty` | 没匹配上的特征要扣多少分（默认 100 nm²；越大对完整匹配越严格）|
| `Max match diff (nm)` | 一对波谷距离超过这个就算匹配失败，但仍计入 RMSE（默认 30 nm）|
| **▲ Auto-Fit Thickness 按钮** | 启动；几秒内出结果弹窗 |

代价函数：`cost = rmse + penalty × unmatched_above_700nm + penalty × (1 − matchRatio)`。
注意只统计**波长 ≥ 700 nm 的未匹配项**（领域经验：短波长那段不可靠）。

Auto-Fit 取 **cost 最小**；接下来的 Fine Search 取 **RMSE 最小**——两者标准不同。

### 2.4 🔍 Fine Search — 在最佳点附近精搜

Auto-Fit 给出的最佳厚度是数据集采样步长的整数倍（比如 0.5 nm）。Fine Search
在它附近做**线性插值**，按更细的步长（比如 0.1 nm）再搜一次。

| 控件 | 作用 |
|---|---|
| `Start (nm)` / `End (nm)` | 精搜范围（跑过 Auto-Fit 后会自动填中点 ±50 nm）|
| `Step (nm)` | 精搜步长 |
| 灰色提示 | 显示精搜会跑多少个厚度点 |
| **🔍 Fine Search 按钮** | 启动；输出方式同 Auto-Fit |

只看 RMSE，不再考虑未匹配惩罚。

---

## 3. 主视图

### 3.1 Gen Spectrum（左边图）

显示**当前数据集 + 当前厚度滑块位置**对应的生成反射谱。

- 浅蓝色实线：raw（原始）
- 深蓝色粗线：smooth（如果你点过 Apply SG Smoothing）
- 蓝色短虚线竖线：检测出的 Valley（波谷）
- 绿色点虚线：检测出的 Peak（波峰）
- 灰色阴影矩形：Gap Mask（排除区间）
- 标题里 `t = 250 nm` 显示当前厚度；双材料模式显示 `MoS2=250 nm, MoO3=80 nm`

### 3.2 Exp Spectrum（中间图）

显示**当前实验文件 + Spectrum index 选中的那条**反射谱。布局同 Gen 图，
颜色用红色调（区分两边）。

### 3.3 Valley / Peak Comparison（右边表）

把两边检测到的波谷/波峰按"距离最近"匹配，逐行对比。

| 列 | 含义 |
|---|---|
| `#` | 序号 |
| `Gen (nm)` | 生成谱该位置波谷波长 |
| `Exp (nm)` | 实验谱该位置波谷波长 |
| `Δ (nm)` | 两者差值；按"对比阈值"染色（绿/橙/红）|

表顶部三个汇总：`Overall Δ` `Valleys Δ` `Peaks Δ`——分别是总体、仅波谷、
仅波峰的平均差。是你判断"拟合好不好"的快速指标。

---

## 4. Settings & Parameters（折叠面板）

页面下方那条 `▶ Settings & Parameters` 点一下展开，所有细调参数都在这里。

### 4.1 Quick Actions

| 按钮 | 作用 |
|---|---|
| `▶ Auto-Run: OFF` / `⏸ Auto-Run: ON` | 开启后**拖厚度滑块时自动重新检测波谷**，方便实时看谱图变化 |
| `📋 Overlay View` | 弹窗把两条谱叠到一张图上对比 |

### 4.2 Wavelength Range — 显示范围

| 字段 | 作用 |
|---|---|
| `Min / Max (nm)` | 谱图的 X 轴范围（留空 = 自动用数据范围）|
| `Custom step` 勾选框 | 启用自定义采样步长（用于显示降采样，加快绘图）|
| `Step (nm)` | 自定义步长大小 |

### 4.3 Gen / Exp Spectrum — Savitzky-Golay Smoothing

分段 SG 平滑——可以**给不同波长区间用不同窗长 / 阶数**。

| 字段 | 作用 |
|---|---|
| `λ start` / `λ end` | 这段的波长范围（留空 = 开口端，一直延伸）|
| `win` | 窗长（必须奇数，越大越平滑）|
| `ord` | 多项式阶数（越高保留细节越多）|
| `+ Segment` | 加一段（可以多段拼起来）|
| **`Apply`** | 把当前所有段送到后端计算，绘制平滑后的曲线 |

不点 Apply 不会做平滑。Apply 过后拖滑块或换数据，平滑参数会**自动重新应用**
（不需要再点）。

### 4.4 Exclusion Intervals (Gap Mask) — 排除区间

某些波长区间（比如已知有大气吸收、光源伪峰）你不想让算法看到，就拉个矩形
盖住。默认有两个：810–835 nm 和 965–985 nm。

| 字段 | 作用 |
|---|---|
| `λ start` / `λ end` | 排除区间起止 |
| `+ Interval` | 加一段 |
| `✕` | 删一段 |

灰色阴影矩形会画在两张谱图上，检测时落在这些区间内的波谷波峰**自动剔除**。

### 4.5 Gen / Exp — Valley / Peak Detection — 波谷波峰检测

scipy `find_peaks` 的参数。Gen 和 Exp 两边独立。

| 字段 | 作用 |
|---|---|
| `Prominence` | 凸起度阈值，越大筛选越严格（默认 0.2）|
| `Distance (nm)` | 两相邻波谷的最小波长距离（默认 20 nm）|
| `Min height` | 最低高度阈值（0 = 不启用）|
| `Valleys` / `Peaks` 勾选 | 检测哪种特征 |
| **`Detect`** 按钮 | 立即重新检测；不点就不会跟随参数变化 |

如果你开了 Auto-Run，拖滑块时会自动跑 Detect，不需要按按钮。

### 4.6 Comparison Thresholds — 表格染色阈值

| 字段 | 作用 |
|---|---|
| `Δ green ≤ (nm)` | 差值 ≤ 这个值的行染绿色（拟合好）|
| `Δ orange ≤ (nm)` | 差值 ≤ 这个值染橙色；超过染红色（拟合差）|

### 4.7 CSV Format — 已废弃

**这一节实际上没用**，是迁移前留下的老配置。生成谱数据集现在统一从服务器拉，
不走客户端 CSV 解析。可以无视。

---

## 5. 底部状态栏

左边显示当前操作状态：

| 提示 | 含义 |
|---|---|
| `Connecting to server...` | 正在唤醒后端（冷启动）|
| `Connected — 3 dataset(s) available` | 后端就绪 |
| `Dataset loaded: mos2` | 选中的数据集下载完毕 |
| `Experiment file loaded — 12 spectra` | 实验文件解析完毕 |
| `Generated spectrum smoothed (2 segments)` | 平滑已应用 |
| `Auto-fit: best thickness = 314.5 nm (RMSE 3.2)` | 拟合结果 |
| `Smoothing failed: ...` | 后端报错；看 F12 控制台 |

右边显示当前显示的厚度。

---

## 6. 弹窗

### 6.1 Auto-Fit Result 弹窗

按 Auto-Fit 后弹出：

- **顶部数字**：最佳厚度 + RMSE + 用了几个波谷/波峰
- **左侧图**：RMSE / Cost 随厚度变化曲线；最低点用红色星标记
  - 右上 `Show Cost` / `Show RMSE` 按钮切换两个视图
- **右侧表**：当前最佳厚度下每个 Gen 波谷和最近 Exp 波谷的配对

点空白处或 `✕` 关闭。Fine Search 完后弹的窗结构一样，会把粗扫曲线作为
背景灰色叠在精搜曲线上方便对比。

### 6.2 Overlay View 弹窗

把 Gen 和 Exp 谱画到同一坐标系，方便目视对比。`Hide Markers` 按钮可以
切换是否显示波谷波峰竖线。

---

## 7. 一些典型流程

### 流程 A：拟合一个 MoS₂ 实验样品

1. 顶部下拉选 `MoS2`
2. 拖入实验 Excel
3. 实验数据 Spectrum index 选你要拟合的那条
4. 展开 Settings → Exp Valley/Peak Detection → 按 `Detect` 检查波谷找得对不对
5. 不对就调 `Prominence` / `Distance` 重检测
6. 按 ▲ Auto-Fit
7. 看弹窗 RMSE 曲线 → 关弹窗 → 厚度滑块已经跳到最佳点
8. 按 🔍 Fine Search 进一步精搜（默认范围在 Auto-Fit 最佳点 ±50 nm）

### 流程 B：只看双材料堆叠谱图（不拟合）

1. 选 `MoS2 on MoO3`
2. 拖两个滑块查看不同厚度组合下的反射谱
3. Auto-Fit / Fine Search 按钮置灰，鼠标悬停提示 "Double-material fitting
   coming soon"

### 流程 C：实验数据噪声大，想先平滑再拟合

1. 加载实验数据
2. Settings → Exp SG Smoothing → 调 `win` 和 `ord` → 按 Apply
3. 这时 Detect 会用平滑后的曲线找波谷
4. 继续走流程 A 的步骤 4 起

---

## 8. 排错

| 现象 | 处理 |
|---|---|
| 状态栏一直 `Connecting to server...` | 等 30s（Render 冷启动）；超过 1 分钟刷新页面 |
| 选数据集后 `Failed to load dataset` | 网络问题或后端 OOM，等几分钟重试 |
| Auto-Fit 报 `select a dataset first` | 还没选数据集 |
| Auto-Fit 报 `select an experiment file first` | 还没拖实验文件 |
| 弹窗 RMSE 曲线很平没有明显谷 | 实验波谷太少 / 不够准，调检测参数或换实验数据 |
| 表里 Δ 全是红色 | 检测参数错配或材料选错；先用 Overlay View 目测两条谱 |
| `429 Too Many Requests` | 触发限流；一分钟内 Auto-Fit / Fine Search 别按超过 15 次 |

# 阶段 1:本地后端骨架

请先完整阅读项目根目录下的 CLAUDE.md,理解工作铁律和阶段划分。
阅读完后向我确认你已理解,然后再开始执行下面的任务。

我们现在开始 **阶段 1:本地后端骨架**。

## 本阶段目标

搭建一个最小可运行的 FastAPI 后端,包含:

1. 完整的 backend/ 目录结构(按 CLAUDE.md 里规划的)
2. CSV → npy 转换工具,把根目录的
   `gen_data_single_MoS2_B85_E2000_step0.5.csv`
   转成 `backend/data/mos2.npy` + `backend/data/mos2.json`
3. 数据集加载器(简单版本,暂不做 LRU 缓存,直接全量加载到内存即可)
4. 两个端点:
   - `GET /datasets` — 返回 [{"id": "mos2", "name": ..., "metadata": ...}]
   - `POST /api/detect` — 峰谷检测,等价于原 HTML 的 runDetection 函数

## 严格遵守

- **完全不要碰 spectrum_analyzer.html**(本阶段前端不动)
- 所有新文件只能放在 `backend/` 目录下
- 算法实现要与原 HTML 中的 JS 版本数值等价
  (具体等价点见 CLAUDE.md 第 6 条铁律)

## 详细步骤

### 步骤 1.1:Git 初始化与初始提交

如果项目还没初始化 git,先执行:
```bash
git init
git add CLAUDE.md spectrum_analyzer.html gen_data_single_MoS2_B85_E2000_step0.5.csv
git commit -m "Initial: original HTML, test CSV, project plan"
```
然后告诉我:"Git 初始化完成,继续步骤 1.2"

### 步骤 1.2:CSV 探测

先 view 原始 CSV 的前 5 行和最后 5 行,以及统计行数列数,
告诉我:
- CSV 多少行?多少列?
- 第一行(或最后一行)是波长行吗?
- 最后一列是厚度列吗?
- 厚度范围、波长范围、步长分别是多少?

这些信息影响转换工具的实现。**等我确认数据格式后再继续**。

### 步骤 1.3:建后端目录结构和依赖文件

创建:
- `backend/app/__init__.py` (空文件)
- `backend/app/algorithms/__init__.py` (空文件)
- `backend/requirements.txt`,包含:
  ```
  fastapi==0.115.0
  uvicorn[standard]==0.30.0
  numpy==1.26.4
  scipy==1.13.0
  pydantic==2.8.0
  ```
- `backend/.gitignore`,包含:
  ```
  __pycache__/
  *.pyc
  .venv/
  venv/
  data/*.npy
  data/*.json
  .env
  ```
- `backend/data/` 空文件夹(用 .gitkeep 占位)

### 步骤 1.4:创建虚拟环境并安装依赖

提示我执行(Windows PowerShell):
```bash
cd backend
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```
等我确认装好后再继续。

### 步骤 1.5:写 CSV → npy 转换工具

文件:`backend/tools/convert_csv_to_npy.py`

要求:
- 接受命令行参数:输入 CSV 路径、输出目录、数据集 id
- 用 numpy 解析 CSV(自动处理:第一行或最后一行是波长、最后一列是厚度)
- 输出两个文件:
  - `{id}.npy`:numpy savez 压缩格式,包含 wavelengths、spectra、thicknesses
  - `{id}.json`:元数据,字段包括 id、name、material、wavelengthRange、
    thicknessRange、thicknessStep、numSpectra、createdAt

例如运行:
```bash
python tools/convert_csv_to_npy.py \
  ../gen_data_single_MoS2_B85_E2000_step0.5.csv \
  data/ mos2 --name "MoS₂" --material "MoS₂"
```

写完后让我确认代码,然后跑一次转换。

### 步骤 1.6:写数据集加载模块

文件:`backend/app/datasets.py`

简单版本:
- 启动时扫描 `data/` 文件夹下所有 `.npy + .json` 配对
- 全部加载到内存中的字典:`DATASETS = {id: {meta, wavelengths, spectra, thicknesses}}`
- 提供函数:
  - `list_datasets() -> list[dict]`:返回元数据列表(不含 spectra)
  - `get_dataset(id) -> dict`:返回完整数据
- 暂不实现 LRU,后续阶段再优化

### 步骤 1.7:写算法模块(只写检测相关)

文件:`backend/app/algorithms/detection.py`

实现 3 个函数:
- `apply_gap_mask(peak_idxs, wavelengths, gaps)`:对应原 JS 行 1893
- `detect_peaks_and_valleys(y, wavelengths, params, gaps)`:对应原 JS 行 1952 的 runDetection
- 使用 scipy.signal.find_peaks,记得:
  - distance 从 nm 换算成采样点数
  - prominence 为 0 时传 None
  - 检测谷:对 -y 调 find_peaks,height 取负

参考原 HTML 中:
- 行 1857 findPeaks 的逻辑
- 行 1893 applyGapMask 的逻辑
- 行 1952 runDetection 的逻辑

### 步骤 1.8:写 Pydantic schema

文件:`backend/app/schemas.py`

定义:
- `DetectionParams`:prominence、distanceNm、height、detectValleys、detectPeaks
- `GapInterval`:wlMin、wlMax
- `DetectRequest`:y、wavelengths、detection、gaps
- `DetectResponse`:valleys、peaks(均为波长列表)
- `DatasetInfo`:id、name、material、wavelengthRange、thicknessRange、thicknessStep、numSpectra

### 步骤 1.9:写 FastAPI 主入口

文件:`backend/app/main.py`

要求:
- 启动时调用 datasets 模块扫描加载
- 配置 CORS 允许所有源(本地开发用,部署阶段再收紧)
- 实现两个端点:
  - `GET /datasets`
  - `POST /api/detect`
- 加一个 `GET /health` 返回 `{"status": "ok"}`,方便后续部署探活

### 步骤 1.10:启动后端并自测

提示我执行:
```bash
cd backend
.venv\Scripts\activate
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

启动后,在 Swagger UI(http://127.0.0.1:8000/docs)上自己测一遍:
- `GET /datasets` 应返回 `[{"id": "mos2", ...}]`
- `POST /api/detect` 用一段简单光谱(比如 `y=[1,2,3,5,3,2,1,2,4,2,1]`,
  `wavelengths=[400,410,420,430,440,450,460,470,480,490,500]`)测试,
  应返回 peak 和 valley 位置

### 步骤 1.11:汇报结果给我

告诉我:
- 所有文件都创建成功了吗?
- 转换工具跑通了吗?MoS₂ 数据集元数据是什么?
- 后端启动有报错吗?
- Swagger UI 自测 `/datasets` 返回什么?
- Swagger UI 自测 `/api/detect` 返回什么?

然后停下来,**等我手动验证**所有"成功标志"后,我会说"阶段 1 完成"。

## 阶段 1 成功标志(用户验证清单)

我会自己验证:
- [ ] `backend/` 目录结构与 CLAUDE.md 一致
- [ ] `backend/data/mos2.npy` 和 `mos2.json` 存在
- [ ] `mos2.json` 里的波长范围、厚度范围与 CSV 实际一致
- [ ] uvicorn 启动后控制台显示 "Uvicorn running on http://0.0.0.0:8000"
- [ ] 浏览器打开 http://127.0.0.1:8000/docs 看到 Swagger UI
- [ ] Swagger 上 `/datasets` 返回非空列表
- [ ] Swagger 上 `/api/detect` 测试返回合理结果

我验证通过后会说"阶段 1 完成",再进入阶段 2。

## 工作纪律

- 每个步骤完成后**停下来汇报**,等我说"继续"再做下一步
- 不要一口气把所有步骤跑完
- 任何超出本阶段任务范围的修改,**先问我**
- 不要 push 到 GitHub
- 不要碰 spectrum_analyzer.html

开始吧。先告诉我"我已阅读 CLAUDE.md,理解了工作铁律",然后从步骤 1.1 开始。

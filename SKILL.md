---
name: LumericalFDTD
description: 为 Ansys Lumerical FDTD 构建和调试 Python 自动化仿真脚本。当你需要创建光学仿真、光子器件设计、衍射分析、超表面、波导、光栅、TGV 结构、光场传播等 FDTD 相关任务时使用此 skill。支持 2025 R2 版本，使用内置 Python 解释器自动运行代码并迭代 debug 直到满足验收条件。
---

# Lumerical FDTD 仿真构建器

## 核心工作流

每次处理 FDTD 任务时严格遵循：

1. **理解需求** — 确认器件结构（几何参数、材料）、光源（波长范围、偏振）、监视器要求、验收条件
2. **编写脚本** — 生成完整独立 Python 脚本，使用固定文件名自动保存
3. **运行调试** — 用内置 Python 执行脚本，根据错误信息修复，重复直到成功运行
4. **验证结果** — 检查仿真输出是否满足验收条件，不满足则调整参数
5. **保存交付** — 保存最终 `.fsp` 文件和脚本，供用户检验

> **关键原则**：模型必须自己运行代码并 debug，不能只生成代码让用户自己试。每个错误都是修正 skill 知识的机会。

## 环境适配（首次对话必须执行）

用户环境差异大（OS、安装路径、版本号各异），首次处理 FDTD 任务时必须先确认环境：

### 1. 探测操作系统

通过 `sys.platform` 或环境变量判断：
- **Windows**: 路径含盘符，反斜杠分隔，Python 解释器为 `python.exe`
- **Linux**: 路径以 `/` 开头，正斜杠分隔，Python 解释器为 `python` 或 `python3`

### 2. 定位 Lumerical 安装路径

按以下优先级尝试，找到可用的 Python 解释器：

**Windows 常见路径：**
```
C:\Apps\ANSYS Inc\v252\Lumerical\python\python.exe
C:\Program Files\ANSYS Inc\v252\Lumerical\python\python.exe
C:\Program Files\Lumerical\v252\python\python.exe
```

**Linux 常见路径：**
```
/opt/ansys_inc/v252/Lumerical/python/python
/opt/lumerical/v252/python/python
/usr/local/lumerical/v252/python/python
```

> 版本号 `v252` 可能不同（如 v251、v241），用通配或 `ls` 探索实际目录名。

### 3. 确认方式

用 `bash` 工具检测解释器是否存在：
```powershell
Test-Path -LiteralPath 'C:\Apps\ANSYS Inc\v252\Lumerical\python\python.exe'
```
或 Linux：
```bash
test -f /opt/ansys_inc/v252/Lumerical/python/python && echo "found"
```

若所有默认路径都不存在，**向用户询问**实际安装路径和版本号。

### 4. 构建项目内 Python 脚本

根据探测到的路径，将路径变量动态写入脚本：

```python
# {PYTHON_PATH} 和 {API_PATH} 由环境探测步骤确定
import sys
sys.path.append("{API_PATH}")
import lumapi
import numpy as np

# 无头模式避免 GUI 弹窗报错
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
```

### 5. 调用方式

**Windows PowerShell**（路径含空格时用 `&` 操作符包裹）：
```powershell
& '{PYTHON_PATH}' 'script.py'
```

**Linux / macOS**：
```bash
{PYTHON_PATH} script.py
```

> **注意**：raw string 不能以 `\` 结尾（如 `r"C:\path\"` 会报 SyntaxError），用 `"C:\\path\\"` 代替。

## 脚本总体结构

```python
# 1. 导入
import sys; sys.path.append("C:\\Apps\\ANSYS Inc\\v252\\api\\python\\")
import lumapi; import numpy as np
import matplotlib; matplotlib.use('Agg'); import matplotlib.pyplot as plt

# 2. 参数定义（集中管理，便于调参）
wavelength = 100e-6
structure_params = {...}

# 3. 创建仿真会话
with lumapi.FDTD(hide=True) as fdtd:
    # 3a. 仿真区域
    # 3b. 材料 / 衬底
    # 3c. 几何结构（从底到顶）
    # 3d. 网格覆盖（可选，精细结构才加）
    # 3e. 光源
    # 3f. 监视器
    # 3g. 保存 -> 运行 -> 提取结果
    fdtd.save("C:\\path\\to\\output.fsp")
    fdtd.run()
    result = fdtd.getresult("monitor_name", "E")

# 4. 后处理与可视化（在 with 块外，此时会话已关闭）
```

> 现成模板见 `scripts/template.py`

## 仿真与后处理分离（必须遵守）

仿真（耗时长）和数据分析/绘图（秒级完成）**必须写在两个独立 `.py` 文件中**，禁止合并：

| 文件 | 职责 | 执行时间 |
|------|------|---------|
| `*_sim.py` | 构建结构、运行 FDTD、保存 `.npz`/`.fsp` | 分钟~小时 |
| `*_analysis.py` | 读取 `.npz`、计算指标、绘图保存 `.png` | 秒级 |

### 命名规范
```
project_name_sim.py       # 仿真（含 skip-if-exists 逻辑）
project_name_analysis.py  # 分析（纯数据读取，不调 lumapi.FDTD）
```

### 原因
- 修改配色/图例/坐标轴时不应重跑整个仿真
- `.npz` 数据是仿真和分析之间的唯一接口
- 分析脚本可在仿真运行后反复执行、迭代调参

### 示例

```python
# myproject_sim.py —— 只做仿真
with lumapi.FDTD(hide=True) as fdtd:
    ...
    fdtd.run()
    np.savez(os.path.join(data_dir, "results.npz"), I=I, x=x, y=y)

# myproject_analysis.py —— 只做分析
data = np.load(os.path.join(data_dir, "results.npz"))
plt.plot(data["x"], data["I"])
plt.savefig(os.path.join(pic_dir, "plot.png"))
```

## 关键约束（必须遵守）

| 约束 | 说明 |
|------|------|
| 不用 `addmaterial` + `set("type",...)` | `type` 属性不可用，直接用内置材料名称字符串 |
| 材料全名不可简写 | `"PEC (Perfect Electrical Conductor)"` 不是 `"PEC"` |
| 多边形函数名是 `addpoly` | 不是 `addpolygon` |
| `addcone` 不存在 | 用多层 `addcircle` 堆叠替代 |
| 多边形不支持 `mesh order` | 省略该 set 语句 |
| 监视器必须在仿真域内 | `z_min < monitor_z < z_max`，否则 `Can not find result 'E'` |
| 多边形顶点用 N×2 矩阵 | `set("vertices", vertices)` 而非分别 set x/y |
| 切片结果用 `.copy()` | NumPy 视图原地操作会污染原始数据 |
| raw string 不能以反斜杠结尾 | 用双反斜杠 `"C:\\path\\"` |

## 参考文档索引

根据当前任务需要读取对应的参考文件：

| 参考文件 | 何时读取 |
|---------|---------|
| `references/building-blocks.md` | 需要构建几何结构、设置光源/监视器/网格时 |
| `references/api-reference.md` | 需要了解会话管理、SimObject、数据传递、lumopt 等高级 API 时 |
| `references/common-errors.md` | 运行报错需要排查时 |
| `references/diffraction.md` | 涉及孔衍射、Airy 环、远场/近场分析时 |

## 文件管理规则（必须遵守）

每个仿真项目目录内严格按以下规则分类，不得将输出文件散落在根目录：

| 目录 | 存放内容 |
|------|---------|
| `fsp/` | `.fsp` 仿真项目文件 + `*_p0.log` 仿真日志 |
| `data/` | `.npz` 仿真结果数据 + `.json` 元数据 |
| `pic/` | `.png/.jpg` 图片和图表 |
| 根目录 | `.py` Python脚本 + `.md` 文档 |

### 规则细节
1. 创建新项目时，必须先在项目目录内建立 `fsp/`、`data/`、`pic/` 三个子目录
2. Python 脚本中 `fdtd.save()` 保存 `.fsp` 到 `fsp/` 目录，图表 `plt.savefig()` 保存到 `pic/` 目录，数据 `np.savez()` 保存到 `data/` 目录
3. 子实验目录（如 `TGV_Waist_vs_Lambda/`）内部也遵循同样的 `fsp/data/pic` 分类
4. 禁止将 `.fsp`、`.npz`、`.png` 文件散落在项目根目录

### 脚本示例路径

```python
project_dir = r"C:\Users\Lex\Documents\FDTD\MyProject"
# 确保目录存在
for sub in ["fsp", "data", "pic"]:
    os.makedirs(os.path.join(project_dir, sub), exist_ok=True)

# 文件保存
fdtd.save(os.path.join(project_dir, "fsp", "simulation.fsp"))
np.savez(os.path.join(project_dir, "data", "results.npz"), E=Ex, ...)
plt.savefig(os.path.join(project_dir, "pic", "field.png"), dpi=150)
```

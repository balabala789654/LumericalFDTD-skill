---
name: LumericalFDTD
description: 为 Ansys Lumerical FDTD 构建和调试 Python 自动化仿真脚本。当你需要创建光学仿真、光子器件设计、衍射分析、超表面、波导、光栅、TGV 结构、光场传播等 FDTD 相关任务时使用此 skill。支持 2025 R2 版本，使用内置 Python 解释器自动运行代码并迭代 debug 直到满足验收条件。
---

# Lumerical FDTD 仿真构建器

## 核心工作流

每次处理 FDTD 任务时遵循这个流程：

1. **理解需求** — 确认器件结构（几何参数、材料）、光源（波长范围、偏振）、监视器要求、验收条件
2. **编写脚本** — 生成完整独立 Python 脚本，使用固定文件名自动保存
3. **运行调试** — 用内置 Python 执行脚本，根据错误信息修复，重复直到成功运行
4. **验证结果** — 检查仿真输出是否满足验收条件，不满足则调整参数
5. **保存交付** — 保存最终 `.fsp` 文件和脚本，供用户检验

> **关键原则**：模型必须自己运行代码并 debug，不能只生成代码让用户自己试。每个错误都是修正 skill 知识的机会。

## 环境配置

**Python 解释器路径：**
```
C:\Apps\ANSYS Inc\v252\Lumerical\python\python.exe
```

**PowerShell 调用方式（路径含空格时必须用 & 操作符包裹）：**
```powershell
& 'C:\Apps\ANSYS Inc\v252\Lumerical\python\python.exe' 'script.py'
```

**脚本开头固定模板：**
```python
import sys
sys.path.append("C:\\Apps\\ANSYS Inc\\v252\\api\\python\\")
import lumapi
import numpy as np

# 无头模式避免 GUI 弹窗报错
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
```

> **注意**：raw string 不能以 `\` 结尾（如 `r"C:\path\"` 会报 SyntaxError），用 `"C:\\path\\"` 代替。

## 脚本结构

每个完整的 FDTD 脚本按以下顺序构建：

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

## 详细构建指南

### 仿真区域 (FDTD Region)

```python
fdtd.addfdtd()
fdtd.set("x", 0)
fdtd.set("y", 0)
fdtd.set("z", 0)
fdtd.set("x span", x_span)
fdtd.set("y span", y_span)
fdtd.set("z span", z_span)
fdtd.set("mesh accuracy", 2)        # 1-8，越高越精细
fdtd.set("pml layers", 8)           # PML 吸收边界层数
fdtd.set("simulation time", t_sim)  # 仿真时间（秒），默认自动
```

**尺寸估算**：区域 span 应包含所有结构 + PML 余量（每边至少 2-3 个网格）。

**加速仿真**：减小区大小、降低 mesh accuracy、缩短 simulation time。

### 材料设置

**核心原则**：不要使用 `addmaterial` 后跟 `set("type", ...)` —— `type` 属性不可用。直接用内置材料名称。

**常用内置材料**：
- 介质：`"SiO2 (Glass) - Palik"`, `"Si (Silicon) - Palik"`
- 金属：`"Au (Gold) - CRC"`, `"Al (Aluminum) - CRC"`, `"Ag (Silver) - CRC"`
- 理想导体：`"PEC (Perfect Electrical Conductor)"`（不可简写为 `"PEC"`）
- 挖空（孔洞）：`material="etch"`

**获取材料折射率：**
```python
c = 2.99792458e8
f_range = c / lambda_range  # 频率数组
n_data = fdtd.getfdtdindex("Au (Gold) - CRC", f_range, min(f_range), max(f_range))
```

### 几何结构

#### 矩形块 (addrect)
```python
fdtd.addrect()
fdtd.set("name", "substrate")
fdtd.set("x span", x_span)
fdtd.set("y span", y_span)
fdtd.set("z span", thickness)
fdtd.set("z", z_center)
fdtd.set("material", "SiO2 (Glass) - Palik")
```

#### 圆形孔 (addcircle)
```python
fdtd.addcircle()
fdtd.set("radius", hole_diameter / 2)
fdtd.set("z span", thickness + 2e-6)  # 略大于衬底以完全穿透
fdtd.set("z", 0)
fdtd.set("material", "etch")
```

#### 多边形孔 (addpoly) —— 注意：函数名是 `addpoly`，不是 `addpolygon`

用于圆度研究、多边形孔等非圆形截面。**必须用 N×2 顶点矩阵**，不能用 `set("x", array)` + `set("y", array)`：

```python
n_sides = 6
angles = np.linspace(0, 2*np.pi, n_sides, endpoint=False)
xv = radius * np.cos(angles)
yv = radius * np.sin(angles)
vertices = np.column_stack([xv, yv])  # N×2 矩阵

fdtd.addpoly()
fdtd.set("vertices", vertices)
fdtd.set("z", 0)
fdtd.set("z span", thickness + 10e-6)
fdtd.set("material", "etch")
```

> **限制**：多边形**不支持** `mesh order` 属性（会报 inactive 错误）。

#### 锥形/渐变结构（addcone 替代方案）

`addcone` 在 Python API 中不存在。用多层薄圆柱 (`addcircle`) 堆叠近似：

```python
layers = 15  # 层数越多越平滑
dz = total_thickness / layers
z_centers = np.linspace(-total_thickness/2 + dz/2, total_thickness/2 - dz/2, layers)

for i, zc in enumerate(z_centers):
    r = r_start + (zc + total_thickness/2) * (r_end - r_start) / total_thickness
    fdtd.addcircle()
    fdtd.set("z", zc)
    fdtd.set("radius", max(r, 0.1e-6))
    fdtd.set("z span", dz)
    fdtd.set("material", "etch")
```

### 光源

#### 平面波 (addplane)
```python
fdtd.addplane()
fdtd.set("name", "source")
fdtd.set("injection axis", "z")
fdtd.set("direction", "forward")     # Forward 或 Backward
fdtd.set("x span", x_span)
fdtd.set("y span", y_span)
fdtd.set("z", z_source)
fdtd.set("wavelength start", wl_start)
fdtd.set("wavelength stop", wl_stop)
```

#### 高斯光束 (addgaussian)
```python
fdtd.addgaussian()
fdtd.set("injection axis", "z")
fdtd.set("direction", "forward")
fdtd.set("x span", x_span)
fdtd.set("y span", y_span)
fdtd.set("z", z_source)
fdtd.set("waist radius w0", w0)
fdtd.set("distance from waist", 0)
fdtd.set("use scalar approximation", 1)
fdtd.setglobalsource("wavelength start", wl)
fdtd.setglobalsource("wavelength stop", wl)
```

### 监视器

#### 功率监视器 (addpower)
```python
fdtd.addpower()
fdtd.set("name", "monitor_trans")
fdtd.set("monitor type", "2D Z-normal")
fdtd.set("x span", x_span)
fdtd.set("y span", y_span)
fdtd.set("z", z_monitor)
```

> **关键约束**：监视器 z 位置必须在仿真域 z_span 范围内（`z_min < z_monitor < z_max`），否则会报 `Can not find result 'E'`。

#### 场分布监视器 (addprofile) —— 用于查看横截面光场分布
```python
fdtd.addprofile()
fdtd.set("name", "field_xy")
fdtd.set("monitor type", "2D Z-normal")
fdtd.set("x span", x_span)
fdtd.set("y span", y_span)
fdtd.set("z", z_monitor)
```

**远场衍射条纹捕捉**：要看到 Airy 环，监视器需足够大以包含第一极小值：
```
r₁ = 1.22 × λ × L / D
```
- λ = 波长, L = 传播距离, D = 孔径
- 若 r₁ 超出监视器 → 只看得到中央亮斑
- 建议在多个 z 位置放置监视器（近、中、远场）观察衍射演化

### 结果提取

```python
# 提取结果
E = fdtd.getresult("monitor_name", "E")
Ex = fdtd.getresult("monitor_name", "Ex")
T = fdtd.getresult("monitor_name", "T")   # 透射率

# getresult 返回 numpy 数组或 dict
# 大数据先存到本地变量，避免重复获取
```

**NumPy 坑点**：对 `getresult` 返回数组切片后原地操作（如 `/=`）会污染原始数据，务必用 `.copy()`：
```python
profile = I[:, idx].copy()  # 正确：独立副本
profile /= profile.max()
```

### 网格覆盖 (Mesh Override)

用于局部加密精细结构（如小孔、渐变区域）的网格。**慎用**：

```python
fdtd.addmesh()
fdtd.set("name", "mesh_hole")
fdtd.set("x span", hole_diameter * 1.5)
fdtd.set("y span", hole_diameter * 1.5)
fdtd.set("z span", thickness * 1.2)
fdtd.set("dx", grid_size)
fdtd.set("dy", grid_size)
fdtd.set("dz", grid_size)
```

> **关键警告**：mesh override 的 dx/dy/dz 必须**小于或等于**自动网格间距，否则会反向降低精度。不确定时直接不设 mesh override，信赖自动网格。

## 衍射仿真要点

研究孔衍射效应时的关键决策：

| 场景 | 衬底材料 | 说明 |
|------|---------|------|
| 纯孔衍射 | `"PEC (Perfect Electrical Conductor)"` | 光只从孔通过，得到纯净衍射图案 |
| 真实 TGV 场景 | `"SiO2 (Glass) - Palik"` | 光同时通过玻璃和孔 |
| 孔衍射+反射 | `"Au (Gold) - CRC"` | 高反射金属，类似 PEC 但考虑损耗 |

## 常见错误速查

| 错误信息 | 原因 | 解决 |
|---------|------|------|
| `'type' is inactive` | 用了 `addmaterial` + `set("type",...)` | 直接用内置材料名称字符串 |
| `'FDTD' object has no attribute 'addpolygon'` | 函数名记错 | 用 `addpoly`（无 `gon`） |
| `'FDTD' object has no attribute 'addcone'` | API 无此函数 | 用多层 `addcircle` 堆叠替代 |
| `'mesh order' is inactive` | 多边形不支持此属性 | 删除该 set 语句 |
| `Can not find result 'E'` | 监视器在仿真域外 | 确认 `z_min < monitor_z < z_max` |
| `vertices` 设置后 I_max=0 | 用 `set("x", arr)` 而非 vertices | 构造 N×2 矩阵用 `set("vertices", ...)` |
| PEC 材料无效 | 简写 `"PEC"` | 用全名 `"PEC (Perfect Electrical Conductor)"` |
| `SyntaxError: unterminated string` | raw string 以 `\` 结尾 | 用 `"C:\\path\\"` 双反斜杠 |
| view 操作污染原数组 | 切片原地归一化 | 用 `.copy()` 创建副本 |
| PowerShell 解析错误 | 路径含空格时用双引号包裹 | 用 `& 'path' 'script'` 而非 `"path"` |

## 高级功能

### 参数扫描

```python
for wl in [80e-6, 100e-6, 120e-6]:
    with lumapi.FDTD(hide=True) as fdtd:
        # ... 构建仿真，使用 wl 作为波长 ...
        fdtd.save(f"C:\\path\\result_{wl*1e6:.0f}um.fsp")
        fdtd.run()
```

### 多监视器同时观察

```python
for z_pos in [5e-6, 30e-6, 60e-6]:
    fdtd.addpower()
    fdtd.set("name", f"monitor_z{z_pos*1e6:.0f}um")
    fdtd.set("z", z_pos)
    # ...
```

## 参考

完整的 Python API 文档见 [API_SUMMARY.md](./API_SUMMARY.md)，包含：
- 会话管理（构造函数参数、with 语句、远程连接）
- SimObject 属性访问
- 数据传递（Python ↔ Lumerical 类型转换）
- lumopt 逆设计优化
- lumslurm HPC 作业调度

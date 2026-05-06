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

## 脚本交付规范

- 输出文件保存到 `C:\Users\Lex` 下
- `.fsp` 文件和 `.py` 脚本使用一致且有意义的文件名
- 后处理图表保存为 `.png` 文件

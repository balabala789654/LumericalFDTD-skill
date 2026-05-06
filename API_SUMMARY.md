# Lumerical Python API 总结

> 原文来源: https://optics.ansys.com/hc/en-us/sections/360005039073-Automation-API
>
> 更新时间: 2026-04-23

---

## 目录

1. [概述](#1-概述)
2. [安装与导入](#2-安装与导入)
3. [会话管理 (Session Management)](#3-会话管理-session-management)
4. [Lumerical 类](#4-lumerical-类)
5. [SimObject 类](#5-simobject-类)
6. [SimObjectResults 类](#6-simobjectresults-类)
7. [脚本命令作为方法](#7-脚本命令作为方法)
8. [传递数据](#8-传递数据)
9. [lumopt 逆设计优化](#9-lumopt-逆设计优化)
10. [lumslurm 作业调度](#10-lumslurm-作业调度)
11. [Interop Server 远程 API](#11-interop-server-远程-api)

---

## 1. 概述

Lumerical Python API 提供三类功能:

- **lumapi**: 自动化仿真工作流
- **lumopt**: 逆设计与优化模块
- **lumslurm**: Slurm 调度器集成

系统要求: Lumerical 2019a R3+, 需要 GUI 许可证

---

## 2. 安装与导入

### 使用 CAD 内置编辑器

直接 import:

```python
import lumapi
```

### 外部编辑器 (Windows)

```python
import sys
sys.path.append("C:\\Program Files\\Ansys Inc\\v252\\api\\python\\")
import lumapi
```

Ansys 安装路径:
```
C:\\Program Files\\Ansys Inc\\[[verpath]]\\Lumerical\\api\\python\\
```

### 永久添加路径

在 Python site-packages 目录添加 .pth 文件,内容为 lumapi.py 所在路径。

### 使用 importlib.util 显式导入

```python
import importlib.util
spec = importlib.util.spec_from_file_location(
    'lumapi',
    'C:\\Program Files\\Lumerical\\v252\\api\\python\\lumapi.py'
)
lumapi = importlib.util.module_from_spec(spec)
spec.loader.exec_module(lumapi)
```

---

## 3. 会话管理 (Session Management)

### 创建会话

```python
fdtd = lumapi.FDTD()  # 启动 FDTD 会话
mode = lumapi.MODE()  # MODE
device = lumapi.DEVICE()  # DEVICE
ic = lumapi.INTERCONNECT()  # INTERCONNECT
```

### 构造函数参数

```python
fdtd = lumapi.FDTD(
    filename=None,      # 项目文件(.fsp/.lms等)或脚本文件(.lsf)
    key=None,           # 已废弃
    hide=False,         # True=隐藏GUI/禁用弹窗
    serverArgs={},      # 命令行参数字典
    remoteArgs={},      # 远程连接字典 {hostname, port}
    project=None,       # 项目文件路径
    script=None         # 脚本文件路径(列表)
)
```

### serverArgs 示例

```python
fdtd = lumapi.FDTD(
    serverArgs={
        'use-solve': True,
        'platform': 'offscreen',
        'threads': '2'
    }
)
```

### remoteArgs 示例 (连接 Interop Server)

```python
fdtd = lumapi.FDTD(remoteArgs={"hostname": "192.168.215.129", "port": 8989})
```

### with 语句 (上下文管理器)

```python
with lumapi.FDTD() as fdtd:
    fdtd.addrect()
    fdtd.run()
# 离开作用域后自动关闭会话
```

### 关闭会话

```python
fdtd.close()
```

---

## 4. Lumerical 类

### 产品与会话类对应

| 产品 | 类 |
|------|-----|
| Ansys Lumerical FDTD | FDTD |
| Ansys Lumerical MODE | MODE |
| Ansys Lumerical Multiphysics | DEVICE |
| Ansys Lumerical INTERCONNECT | INTERCONNECT |

### 默认方法

- `close()`: 关闭当前 Lumerical 会话
- `getObjectById(id)`: 通过 ID 获取特定对象
- `getObjectBySelection()`: 获取当前选中的对象
- `getAllSelectedObjects()`: 获取所有选中对象的列表

### 低级脚本工作区方法

- `eval(script_string)`: 执行 Lumerical 脚本字符串
- `getv(variable_name)`: 从 Lumerical 工作区获取变量到 Python
- `putv(varname, value)`: 将 Python 变量放入 Lumerical 工作区

### eval 示例

```python
fdtd = lumapi.FDTD()
fdtd.eval("addrect; set('x span', 1e-6);")
```

### getv / putv 示例

```python
# putv: Python -> Lumerical
fdtd.putv("my_var", 3.14)

# getv: Lumerical -> Python
value = fdtd.getv("my_var")
```

---

## 5. SimObject 类

SimObject 代表对象树中的仿真对象。

### 属性访问

属性名与 Lumerical 属性名相同,空格替换为下划线:

```python
rect_obj = fdtd.addrect(x=0, y=0, z=0, x_span=2e-6)
rect_obj.y_span = 2e-6              # 属性访问
rect_obj["z span"] = 0.5e-6         # 字典访问(保留原名)
print(rect_obj.x_span)
```

### 警告

同一名称的两个对象会导致未定义行为。即使 Python 变量不同,也不要创建同名对象。

### 方法

- `getParent()`: 获取当前对象的父对象
- `getChildren()`: 获取当前对象的所有子对象列表

### results 属性

通过 `.results` 访问仿真对象的所有结果:

```python
monitor = fdtd.addpower()
E_field = monitor.results.E
```

---

## 6. SimObjectResults 类

包含仿真对象的所有结果,通过 `SimObject.results` 访问。

- 属性名与结果名相同,空格替换为下划线
- 可通过 `obj["result name"]` 访问
- 结果数据每次访问时重新获取,大数据应先存到本地变量

---

## 7. 脚本命令作为方法

几乎所有 Lumerical 脚本命令都可以作为方法调用:

```python
fdtd.addfdtd()
fdtd.set("x", 0)
fdtd.set("z span", 0.5e-6)
```

### 添加对象时直接设置属性

```python
fdtd.addrect(x=0, y=0, z=0, x_span=2e-6)
```

### 常用脚本命令

- `addfdtd()`: 添加 FDTD 仿真区域
- `addrect()`, `addcircle()`, `addpolygon()`: 添加几何形状
- `addplane()`, `addgaussian()`, `adddipole()`: 添加光源
- `addpower()`, `addprofile()`, `addfieldmonitor()`: 添加监视器
- `set()`, `get()`: 设置/获取属性
- `run()`: 运行仿真
- `save()`: 保存文件

### getfdtdindex 示例

```python
c = 2.99792458e8
f_range = np.linspace(c/1100e-9, c/400e-9, 1000)
au_index = fdtd.getfdtdindex("Au (Gold) - CRC", f_range, np.min(f_range), np.max(f_range))
```

---

## 8. 传递数据

### 数据类型转换

| Lumerical | Python |
|-----------|--------|
| String | str |
| Real | float |
| Complex | np.array |
| Matrix | np.array |
| Cell array | list |
| Struct | dict |
| Dataset | dict |

### Python -> Lumerical

- **String**: 直接转换
- **Real**: 转换为 float (整数也会转)
- **Complex**: 必须封装为 1x1 numpy 数组,Python 用 `j` 表示虚部
- **numpy array**: 转换为矩阵,支持复数
- **list**: 转换为 cell 数组
- **dict**: 转换为结构体

### Lumerical -> Python

- **String**: str
- **Real**: float
- **Complex**: 1x1 numpy 数组
- **Matrix**: numpy 数组
- **Struct**: dict
- **Cell array**: list

### 复数示例

```python
fdtd.eval("function return_complex(){return 1+1i;}")
complex_value = fdtd.return_complex()
# complex_value[0] = [1.+1.j]
```

### 结构体示例

```python
fdtd.eval('function return_struct(){return {"name":"MyStruct","value": 1e-6};}')
struct_returned = fdtd.return_struct()
# struct_returned = {'name': 'MyStruct', 'value': 1e-06}
```

---

## 9. lumopt 逆设计优化

lumopt 是基于伴随方法的逆设计优化框架,使用 Lumerical FDTD 求解麦克斯韦方程组,scipy 进行优化。

### 核心概念

- **Figure of Merit (FOM)**: 优化目标
  - `ModeMatch`: 导模功率耦合
  - `IntensityVolume`: 表面/体积光强
- **优化参数**: 形状或拓扑参数

### ModeMatch FOM

```python
from lumopt.optimizers import Optimizer
from lumopt.figure_of_merit import ModeMatch
from lumopt.geometries import PolygonGeometry

fom = ModeMatch(
    wavelength=1550e-9,
    target_T_fwd=0.95
)
```

### 形状优化几何

```python
from lumopt.geometries import PolygonGeometry

initial_points_y = [0, 0.5e-6, 0.5e-6, 0]
geometry = PolygonGeometry(
    initial_points_x=points_x,
    initial_points_y=initial_points_y
)
```

### 优化器

```python
optimizer = Optimizer(
    max_iter=500,
    method='L-BFGS-B'
)
```

### 运行优化

```python
opt_result = optimizer.run(geometry, fom)
```

---

## 10. lumslurm 作业调度

lumslurm 模块自动化 Slurm 集群上的作业提交和后处理。

### 用途

- HPC 集群批量仿真
- 参数扫描
- 优化迭代

---

## 11. Interop Server 远程 API

允许在远程 Linux 机器上运行 Lumerical,通过 IP 和端口连接。

### 启动 Interop Server

```bash
./interop-server --port 8989
```

### 连接远程会话

```python
fdtd = lumapi.FDTD(remoteArgs={"hostname": "192.168.215.129", "port": 8989})
```

---

## 相关资源链接

- [Lumerical Python API Reference](https://optics.ansys.com/hc/en-us/articles/38660003331859-Lumerical-Python-API-Reference)
- [Python API Overview](https://optics.ansys.com/hc/en-us/articles/360037824513-Python-API-overview)
- [Session Management - Python API](https://optics.ansys.com/hc/en-us/articles/360041873053-Session-Management-Python-API)
- [Script Commands as Methods - Python API](https://optics.ansys.com/hc/en-us/articles/360041579954-Script-Commands-as-Methods-Python-API)
- [Working with Simulation Objects - Python API](https://optics.ansys.com/hc/en-us/articles/39744946400659-Working-with-Simulation-Objects-Python-API)
- [Passing Data - Python API](https://optics.ansys.com/hc/en-us/articles/360041401434-Passing-Data-Python-API)
- [Accessing Simulation Results - Python API](https://optics.ansys.com/hc/en-us/articles/39744236202771-Accessing-Simulation-Results-Python-API)
- [Getting Started with lumopt - Python API](https://optics.ansys.com/hc/en-us/articles/360050995394-Getting-Started-with-lumopt-Python-API)
- [Photonic Inverse Design Overview - Python API](https://optics.ansys.com/hc/en-us/articles/360049853854-Photonic-Inverse-Design-Overview-Python-API)
- [Optimizable Geometry - Python API](https://optics.ansys.com/hc/en-us/articles/360052044913-Optimizable-Geometry-Python-API)
- [Getting Started with lumslurm - Python API](https://optics.ansys.com/hc/en-us/articles/20990924220691-Getting-Started-with-lumslurm-Python-API)
- [Interop Server - Remote API](https://optics.ansys.com/hc/en-us/articles/15499581457811-Interop-Server-Remote-API)
- [Installation and Getting Started - Python API](https://optics.ansys.com/hc/en-us/articles/39744901602707-Installation-and-Getting-Started-Python-API)

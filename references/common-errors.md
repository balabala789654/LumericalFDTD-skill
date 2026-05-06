# 常见错误速查

> 此文件在脚本运行报错时读取，按错误信息快速定位原因和解决方案。

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

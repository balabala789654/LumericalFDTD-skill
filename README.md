# LumericalFDTD Skill

基于 Ansys Lumerical 2025 R2 Python API 的 FDTD 自动化仿真 skill，支持光子器件设计、衍射分析、超表面、波导、光栅、TGV 结构等场景。

## 目录结构

```
LumericalFDTD/
├── SKILL.md                     # skill 入口：核心工作流、环境配置、快速索引
├── references/
│   ├── building-blocks.md       # 仿真构建：几何、光源、监视器、网格的详细用法
│   ├── api-reference.md         # 高级 API：会话管理、SimObject、数据传递、lumopt
│   ├── common-errors.md         # 调试速查：常见错误现象与解决方案
│   └── diffraction.md           # 衍射专项：衬底选材、Airy 环捕捉策略
├── scripts/
│   └── template.py              # 即用脚本模板，修改参数区即可运行
├── assets/                      # 预留资源目录
├── LICENSE.txt
└── README.md
```

## 设计原则

采用 skill-creator 规范的**三级渐进加载**：

1. **SKILL.md** — 始终在上下文中，定义核心工作流和关键约束
2. **references/** — 根据当前任务按需读取对应参考文档
3. **scripts/** — 提供即用的模板和工具脚本

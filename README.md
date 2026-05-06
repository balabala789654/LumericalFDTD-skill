# LumericalFDTD Skill

为 Ansys Lumerical FDTD 自动构建和调试 Python 仿真脚本。只需描述你的光学器件需求（结构、材料、光源、监视器），skill 会自动生成脚本、运行仿真、迭代 debug，最终交付 FSP 文件和结果图表。

## 适用场景

光子器件设计、衍射分析、超表面、波导、光栅、TGV 通孔、光场传播等需要 FDTD 仿真的任务。

## 使用方式

在对话中直接描述你的仿真需求，例如：

- "设计一个直径 30μm 的圆孔在 50μm 厚 SiO2 衬底上，用 100μm 波长平面波照射，观察透射衍射图案"
- "模拟金光栅的反射谱，周期 10μm，占空比 0.5，波长 1-2μm"
- "参数扫描：圆孔直径从 20μm 到 60μm，步长 10μm，对比透射率"

Skill 会自动完成：理解需求 → 生成脚本 → 运行调试 → 验证结果 → 保存交付。

## 文件管理规则

| 目录 | 存放内容 |
|------|---------|
| `fsp/` | `.fsp` 仿真项目文件 + `*_p0.log` 仿真日志 |
| `data/` | `.npz` 仿真结果数据 + `.json` 元数据 |
| `pic/` | `.png/.jpg` 图片和图表 |
| 根目录 | `.py` Python脚本 + `.md` 文档 |

## Star History

<a href="https://www.star-history.com/?repos=balabala789654%2FLumericalFDTD-skill&type=date&legend=top-left">

 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=balabala789654/LumericalFDTD-skill&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=balabala789654/LumericalFDTD-skill&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=balabala789654/LumericalFDTD-skill&type=date&legend=top-left" />
 </picture>
</a>




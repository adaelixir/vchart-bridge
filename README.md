# echarts-to-vchart-migrator

GitHub Copilot skill，用于将 ECharts 封装的图表库迁移到 VChart。

## 安装

```bash
npx degit huanxi/echarts-to-vchart-migrator ~/.agents/skills/echarts-to-vchart-migrator
```

> 把 `huanxi` 替换成你的 GitHub 用户名。

安装后**无需重启 VS Code**，在 Copilot Chat 输入 `/` 即可看到 `echarts-to-vchart-migrator`。

## 更新

重新运行安装命令即可覆盖更新（`degit` 不带 `.git`，不会有冲突）。

## 使用方式

### Mode A — 扫描整个项目，生成迁移计划

在 Copilot Chat 里：

```
/echarts-to-vchart-migrator 帮我分析一下这个项目要怎么迁移
```

产出：
- 图表类型清单（风险级别 A/B/C/D）
- 结构耦合点分析（plugin 数据改写、hooks 类型依赖、formatter、主题 token）
- 逐图 todo checklist
- 分阶段迁移计划
- 可直接打开的 HTML 对比 demo（不需要 build）

### Mode B — 转换单张 option

```
/echarts-to-vchart-migrator 帮我转换这个 option：
{ series: [{ type: 'bar', data: [120, 200, 150] }], xAxis: { data: ['Mon','Tue','Wed'] } }
```

产出：
- `NormalizedChartSpec`（中间语义层）
- `VChart ISpec`（可直接传给 `new VChart(spec, { dom })`）
- 字段映射表（mapped / partially-mapped / not-mapped）
- 不支持的字段列表 + 手动处理指引
- 迁移决策：migrate-now / migrate-with-fallback / keep-echarts

## 项目特定补充

如果你的项目有特定的适配器架构、已知的耦合点或当前迁移进度，可以在项目的 `.github/skills/echarts-to-vchart-migrator/SKILL.md` 里补充说明。AI 会自动读取并与通用 workflow 合并，跳过已知信息的重复发现。

示例：在 mchart 项目里已有项目级补充文件，记录了：
- 适配器层文件路径
- 各图表 plugin 的已知数据改写行为
- 当前迁移进度快照

## 文件结构

```
SKILL.md                                ← 执行指令（AI 读取此文件）
templates/
  migration-input.schema.json           ← 输入契约（mode: scan | convert）
  migration-output.schema.json          ← 输出契约（scanResult | convertResult）
  acceptance-checklist.template.json    ← 验收清单模板
```

## 风险级别说明

| 级别 | 含义 |
|---|---|
| **A** | 基础声明式图表（bar/line/area/pie/scatter），可自动迁移 |
| **B** | VChart 有对应能力，但 spec 结构差异较大，或 plugin 层改写了数据，需半自动处理 |
| **C** | ECharts 独有特性、自定义 series 扩展、渲染模型根本不同，需人工评估 |
| **D** | 无 VChart 对应实现，永久保留 ECharts adaptor |

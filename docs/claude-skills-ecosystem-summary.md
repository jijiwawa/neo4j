# Claude Skills 生态系统总结

> 生成时间：2026-03-27
> 目标读者：鲲鹏CPU性能优化工程师

---

## 1. Skills 基础概念

### 1.1 什么是 Skills？

Skills 是 Claude 的可复用能力模块，包含指令、脚本和资源文件，Claude 会在执行相关任务时动态发现并加载。

### 1.2 Skills 加载机制

Skills 采用**渐进式披露架构**（Progressive Disclosure）：

| 层级 | 内容 | Token 消耗 |
|------|------|-----------|
| 元数据扫描 | name + description | ~100 tokens |
| 完整指令 | SKILL.md 内容 | <5k tokens |
| 附加资源 | scripts/references/assets | 按需加载 |

### 1.3 Skills vs 其他方案对比

| 工具 | 最佳用途 |
|------|---------|
| **Skills** | 跨对话的可复用过程性知识 |
| **Prompts** | 一次性指令和即时上下文 |
| **Projects** | 工作区内的持久化背景知识 |
| **Subagents** | 独立任务执行，具有特定权限 |
| **MCP** | 连接 Claude 到外部数据源/API |

---

## 2. Skills 存储位置

### 2.1 全局插件目录

```
~/.claude/plugins/marketplaces/claude-plugins-official/
├── plugins/
│   ├── <plugin-name>/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json          # 插件元数据
│   │   ├── skills/
│   │   │   └── <skill-name>/
│   │   │       ├── SKILL.md          # 主文件（必需）
│   │   │       ├── scripts/          # 可执行脚本
│   │   │       ├── references/       # 参考文档
│   │   │       └── assets/           # 资源文件
│   │   └── commands/
│   │       └── <command-name>.md     # 命令定义
```

### 2.2 当前内置 Skills

| Skill | 描述 |
|-------|------|
| **update-config** | 配置 settings.json，包括权限、环境变量、hooks |
| **simplify** | 审查代码变更，检查重用性、质量和效率 |
| **loop** | 按重复间隔运行提示或斜杠命令 |
| **claude-api** | 使用 Claude API 或 Anthropic SDK 构建应用 |

### 2.3 当前环境已安装的 Skills

```
/Users/estuary/.claude/plugins/marketplaces/claude-plugins-official/plugins/
├── agent-sdk-dev/
├── claude-automation-recommender/
├── claude-md-management/
├── code-review/
├── commit-commands/
├── example-plugin/
├── feature-dev/
├── frontend-design/
├── hookify/
├── playground/
├── plugin-dev/
├── pr-review-toolkit/
├── ralph-loop/
├── skill-creator/          # 存在但未加载
└── stripe/ (external)
```

---

## 3. 现有性能优化相关 Skills

### 3.1 灰色港湾-性能优化 (LobeHub)

- **地址**: https://lobehub.com/zh/skills/greyhaven-ai-claude-code-config-performance-optimization
- **覆盖内容**:
  - 算法复杂度优化 (O(n²)→O(n))
  - 数据库优化 (N+1查询、索引)
  - React 性能 (记忆化、虚拟列表)
  - 打包优化 (代码拆分)
  - API 缓存与异步处理
  - 内存泄漏修复
- **局限**: 主要面向应用层（Web/React/数据库），**非底层CPU/系统级优化**

### 3.2 Performance Profiling & Optimization (MCP Market)

- **地址**: https://mcpmarket.com/tools/skills/performance-profiling-optimization-1
- **功能**: 瓶颈识别、火焰图生成、应用效率优化

### 3.3 johnlindquist/claude perf skill

- **地址**: https://agentskills.so/skills/johnlindquist-claude-perf
- **功能**: 基准测试、热点分析、Lighthouse 审计

### 3.4 obra/superpowers

- **安装**: `/plugin marketplace add obra/superpowers-marketplace`
- **功能**: 20+ 技能包括 TDD、调试、代码分析模式

### 3.5 Trail of Bits Security Skills

- **功能**: CodeQL/Semgrep 静态分析、代码审计
- **适用**: 源码深度分析

---

## 4. 鲲鹏CPU性能优化工程师 - Gap 分析

### 4.1 工作职责

| 领域 | 描述 |
|------|------|
| 系统瓶颈观测 | 使用性能分析工具识别系统瓶颈 |
| BIOS/OS 配置优化 | 调整系统配置以提升性能 |
| 热点函数抓取 | 使用 perf/systemtap/bpftrace 采集热点 |
| 高性能库替换 | 使用 KML、OpenBLAS 等高性能库 |
| SIMD 优化 | NEON/SVE 向量化优化 |
| 算子融合 | 合并计算算子减少开销 |
| 编译优化 | PGO/LTO 编译器优化技术 |
| 4+1 视图生成 | 架构视图文档化 |

### 4.2 现有 Skills 覆盖情况

| 工作领域 | 现有覆盖 | 状态 |
|----------|---------|------|
| BIOS/OS 配置优化 | 无 | ❌ 需自定义 |
| perf/systemtap/bpftrace 热点抓取 | 部分 | ⚠️ 需扩展 |
| 鲲鹏/ARM SIMD (NEON/SVE) | 无 | ❌ 需自定义 |
| 算子融合优化 | 无 | ❌ 需自定义 |
| PGO/LTO 编译优化 | 无 | ❌ 需自定义 |
| 4+1 视图生成 | 无 | ❌ 需自定义 |
| 高性能库替换 (OpenBLAS → KML 等) | 无 | ❌ 需自定义 |

---

## 5. 推荐自定义 Skills 方案

### 5.1 Skill 1: kunpeng-perf-analyzer（性能分析）

```
kunpeng-perf-analyzer/
├── SKILL.md                    # 主文件
├── scripts/
│   ├── hotspot_collector.sh    # perf 采集脚本
│   └── flamegraph_gen.py       # 火焰图生成
├── references/
│   ├── perf_events.md          # perf 使用指南
│   ├── kunpeng_tuning.md       # 鲲鹏调优手册
│   └── bpftrace_probes.md      # bpftrace 探针
└── templates/
    └── perf_report.md          # 性能报告模板
```

**功能描述**：
- 分析 perf 热点数据，识别瓶颈函数
- 生成火焰图可视化
- 根据函数特征推荐优化策略

### 5.2 Skill 2: simd-optimizer（SIMD优化）

```
simd-optimizer/
├── SKILL.md
├── references/
│   ├── arm_neon_intrinsics.md  # NEON 内置函数参考
│   ├── arm_sve_guide.md        # SVE 向量扩展指南
│   └── simd_patterns.md        # SIMD 优化模式
└── examples/
    ├── loop_vectorization.c    # 循环向量化示例
    └── matrix_ops_neon.c       # 矩阵操作 NEON 示例
```

**功能描述**：
- 识别可向量化的循环和数据结构
- 生成 NEON/SVE 优化代码
- 提供数据布局优化建议（AoS → SoA）

### 5.3 Skill 3: compiler-optimizer（编译优化）

```
compiler-optimizer/
├── SKILL.md
├── references/
│   ├── gcc_pgo_lto.md          # GCC PGO/LTO 指南
│   ├── llvm_passes.md          # LLVM Pass 使用
│   └── auto_vectorization.md   # 自动向量化选项
└── scripts/
    └── pgo_workflow.sh         # PGO 工作流脚本
```

**功能描述**：
- PGO（Profile-Guided Optimization）配置
- LTO（Link-Time Optimization）设置
- 编译器优化选项推荐

### 5.4 Skill 4: architecture-view（4+1视图）

```
architecture-view/
├── SKILL.md
├── templates/
│   ├── logical_view.md         # 逻辑视图
│   ├── process_view.md         # 进程视图
│   ├── development_view.md     # 开发视图
│   ├── physical_view.md        # 物理视图
│   └── scenarios.md            # 场景视图
└── scripts/
    └── generate_diagrams.py    # 图表生成脚本
```

**功能描述**：
- 分析源码生成 4+1 架构视图
- 支持多种图表格式输出
- 热点函数架构分析

---

## 6. Skills 创建与管理

### 6.1 创建 Skill 的两种方式

#### 方式 1：使用 skill-creator（推荐）

```bash
# 加载 skill-creator 插件
claude plugin install /Users/estuary/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator

# 然后在对话中使用
/skill-creator
```

#### 方式 2：手动创建

1. 创建目录结构：
```bash
mkdir -p ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/my-plugin/skills/my-skill/{scripts,references,templates}
```

2. 创建 SKILL.md：
```markdown
---
name: my-skill
description: 简洁描述，用于 skill 发现
---

# 详细指令

Claude 会在激活此 skill 时读取这些指令。

## 使用方法
解释如何使用...

## 示例
提供清晰示例...
```

### 6.2 SKILL.md 模板

```markdown
---
name: kunpeng-perf-analyzer
description: 鲲鹏CPU性能分析skill，用于分析perf热点数据、识别瓶颈函数、生成火焰图、推荐优化策略。当用户提到性能分析、热点函数、perf、火焰图、CPU瓶颈、性能调优时使用。
---

# 鲲鹏 CPU 性能分析器

## 功能

1. **热点分析**: 分析 perf 数据，识别 CPU 密集函数
2. **火焰图生成**: 生成可视化火焰图
3. **优化建议**: 根据函数特征推荐 SIMD/算子融合/PGO 等优化策略

## 使用方法

### 1. 采集热点数据

\`\`\`bash
perf record -g -p <pid> -- sleep 30
perf report --stdio
\`\`\`

### 2. 分析热点函数

读取 perf 报告，识别：
- CPU 占用最高的函数
- 调用栈深度
- 缓存命中率

### 3. 生成优化建议

根据函数特征推荐：
- **内存密集型**: 考虑数据布局优化、预取
- **计算密集型**: 考虑 SIMD 向量化
- **调用频繁**: 考虑内联、算子融合

## 参考文件

- `references/perf_events.md`: perf 使用详细指南
- `references/kunpeng_tuning.md`: 鲲鹏调优手册
```

### 6.3 常用命令

```bash
# 列出已安装插件
claude plugin list

# 安装本地插件
claude plugin install <path>

# 从 marketplace 安装
claude plugin marketplace add <marketplace-name>

# 加载插件目录启动
claude --plugin-dir <path>
```

---

## 7. 参考资源

### 7.1 官方文档

- [Claude Skills 官方公告](https://www.anthropic.com/news/claude-skills)
- [Skills API 文档](https://docs.anthropic.com/claude/docs/skills)
- [官方 Skills 仓库](https://github.com/anthropics/skills)

### 7.2 社区资源

- [Awesome Claude Skills](https://github.com/travisvn/awesome-claude-skills)
- [LobeHub Skills Marketplace](https://lobehub.com/skills)
- [MCP Market Skills](https://mcpmarket.com/tools/skills)
- [obra/superpowers](https://github.com/obra/superpowers)

### 7.3 教程

- [How to Create Claude Code Skills](https://websearchapi.ai/blog/how-to-create-claude-code-skills)
- [Claude Code Deep Dive - Plug and Play](https://medium.com/@the.gigi/claude-code-deep-dive-plug-and-play-af03f77c6568)

---

## 8. 下一步行动

1. **加载 skill-creator 插件**:
   ```bash
   claude plugin install /Users/estuary/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator
   ```

2. **创建第一个自定义 Skill**:
   使用 `/skill-creator` 创建 `kunpeng-perf-analyzer`

3. **安装社区 Skills**:
   ```bash
   claude plugin marketplace add obra/superpowers-marketplace
   ```

4. **持续迭代**:
   根据实际使用反馈优化 Skills

---

*文档结束*

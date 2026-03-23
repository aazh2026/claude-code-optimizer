---
name: claude-code-optimizer
description: 基于工程实践优化 Claude Code 配置，诊断上下文膨胀、工具冗余、Skills 设计等问题，输出可执行的改进建议。适用于 Claude Code 重度用户进行配置治理和性能调优。

input_schema:
  project_path:
    type: string
    required: true
    description: 项目根目录路径
  check_scope:
    type: string
    required: false
    default: "deep"
    enum: ["basic", "deep"]
    description: 检查范围

output_schema:
  summary:
    type: object
    properties:
      context_health: { type: string, enum: ["healthy", "needs_attention", "critical"] }
      config_score: { type: number }
      priority_issues: { type: integer }
  context_audit:
    type: object
    properties:
      fixed_overhead: { type: string }
      mcp_bloat: { type: string }
      recommendation: { type: string }
  action_items:
    type: array
    items: { type: string }

verification:
  - check: "output.config_score >= 0 and output.config_score <= 100"
    severity: error
  - check: "len(output.action_items) > 0"
    severity: error
  - check: "output.summary.context_health is valid"
    severity: error
  - check: "output.priority_issues == count_critical_items(output.action_items)"
    severity: warning

---

# Claude Code 优化器

基于 Tw93 的「六层架构」方法论，系统性优化 Claude Code 配置。

## 核心框架：六层架构

```
收集上下文 → 采取行动 → 验证结果 → [完成 or 回到收集]
      ↑              ↓
  CLAUDE.md      Hooks / 权限 / 沙箱
  Skills         Tools / MCP
  Memory
```

只强化一层会导致系统失衡。

## 输入参数

| 参数名 | 类型 | 必填 | 描述 |
|--------|------|------|------|
| project_path | string | 是 | 项目根目录路径 |
| check_scope | string | 否 | 检查范围：basic(快速检查) / deep(完整分析)，默认 deep |

## 执行步骤

### 1. 上下文成本审计

检查 200K 上下文的实际分配：

```
200K 总上下文
├── 固定开销 (~15-20K)
│   ├── 系统指令: ~2K
│   ├── 所有启用的 Skill 描述符: ~1-5K  ⚠️ 检查描述是否精简
│   ├── MCP Server 工具定义: ~10-20K   ⚠️ 最大隐形杀手
│   └── LSP 状态: ~2-5K
├── 半固定 (~5-10K)
│   ├── CLAUDE.md: ~2-5K               ⚠️ 是否 <200 行
│   └── Memory: ~1-2K
└── 动态可用 (~160-180K)
    ├── 对话历史
    ├── 文件内容
    └── 工具调用结果
```

**检查命令：**
```bash
# 查看 token 占用结构
/context

# 确认哪些 CLAUDE.md 被加载
/memory
```

### 2. 配置结构检查

**标准布局：**
```
Project/
├── CLAUDE.md                    # 项目契约（<200 行，2-5K tokens）
├── CLAUDE.local.md              # 个人覆盖（gitignored）
└── .claude/
    ├── settings.json            # 权限配置
    ├── settings.local.json      # 个人权限覆盖
    ├── rules/                   # 路径/语言特定规则
    │   ├── core.md
    │   ├── config.md
    │   └── release.md
    ├── skills/                  # 按需加载的工作流
    │   ├── runtime-diagnosis/
    │   ├── config-migration/
    │   └── release-check/
    ├── agents/                  # 子代理配置
    │   ├── reviewer.md
    │   └── explorer.md
    └── hooks/                   # Hook 配置（如需）
```

**检查清单：**
- [ ] CLAUDE.md 是否 <200 行
- [ ] 是否有 CLAUDE.local.md 做个人覆盖
- [ ] rules/ 是否按路径生效，而非全部塞进根 CLAUDE.md
- [ ] Skills 描述符是否精简（低效 45 tokens vs 高效 9 tokens）

### 3. Skills 设计审查

**好 Skill 的标准：**

| 检查项 | 要求 |
|--------|------|
| 描述 | 让模型知道"何时该用我"，不是"我是干什么的" |
| 完整性 | 有步骤、输入、输出、停止条件 |
| 结构 | 正文只放导航和核心约束，大资料拆到 supporting files |
| 副作用 | 有副作用的 Skill 设置 `disable-model-invocation: true` |

**三种典型类型：**

1. **检查清单型**（质量门禁）
   ```yaml
   name: release-check
   description: Use before cutting a release to verify build, version, and smoke test.
   ```

2. **工作流型**（标准化操作）
   ```yaml
   name: config-migration
   description: Migrate config schema. Run only when explicitly requested.
   disable-model-invocation: true  # 显式调用
   ```

3. **领域专家型**（封装决策框架）
   ```yaml
   name: runtime-diagnosis
   description: Use when app crashes, hangs, or behaves unexpectedly at runtime.
   ```

**反模式检查：**
- [ ] 描述过短（如 "help with backend"）
- [ ] 正文过长（几百行手册塞进 SKILL.md）
- [ ] 一个 Skill 覆盖 review/deploy/debug/docs/incident 五件事
- [ ] 有副作用的 Skill 允许模型自动调用

### 4. 工具设计审查

**好工具 vs 坏工具：**

| 好工具 | 坏工具 |
|--------|--------|
| 名称前缀按系统分层：github_pr_*、jira_issue_* | 命名随意，难以选择 |
| 支持 response_format: concise / detailed | 总是返回完整大响应 |
| 错误响应教模型如何修正 | 只抛 opaque error code |
| 合并成高层任务工具 | 暴露过多底层碎片工具 |

**检查：**
- [ ] 工具名称是否有清晰前缀
- [ ] 大响应是否支持简洁模式
- [ ] 是否避免 list_all_* 让模型自行筛选

### 5. Hooks 配置检查

**适合 Hooks：**
- 阻断修改受保护文件
- Edit 后自动格式化/lint/轻量校验
- SessionStart 后注入动态上下文
- 任务完成后推送通知

**不适合 Hooks：**
- 需要读大量上下文的复杂语义判断
- 长时间运行的业务流程
- 需要多步推理和权衡的决策

**示例配置：**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "pattern": "*.rs",
        "hooks": [
          {
            "type": "command",
            "command": "cargo check 2>&1 | head -30",
            "statusMessage": "Running cargo check..."
          }
        ]
      }
    ]
  }
}
```

### 6. 输出优化报告

## 输出格式

```json
{
  "summary": {
    "context_health": "needs_attention",
    "config_score": 7.5,
    "priority_issues": 3
  },
  "context_audit": {
    "fixed_overhead": "18K",
    "mcp_bloat": "5 servers = ~25K tokens (12.5%)",
    "recommendation": "Disconnect idle MCP servers"
  },
  "claUDE_md_check": {
    "line_count": 150,
    "token_estimate": "3.5K",
    "has_compact_instructions": true,
    "issues": []
  },
  "skills_review": {
    "total_skills": 5,
    "auto_invoke_optimized": 3,
    "issues": [
      {
        "skill": "security-review",
        "problem": "Description too long (~45 tokens)",
        "fix": "Shorten to: 'Use for PR reviews with focus on security.'"
      }
    ]
  },
  "hooks_config": {
    "has_rust_check": true,
    "has_lint_on_edit": true,
    "recommendations": []
  },
  "action_items": [
    "Disconnect 2 idle MCP servers to save ~10K tokens",
    "Optimize security-review skill description",
    "Add lua syntax check hook for *.lua files"
  ]
}
```

## CLAUDE.md 高质量模板

```markdown
# Project Contract

## Build And Test
- Install: `pnpm install`
- Dev: `pnpm dev`
- Test: `pnpm test`
- Typecheck: `pnpm typecheck`
- Lint: `pnpm lint`

## Architecture Boundaries
- HTTP handlers live in `src/http/handlers/`
- Domain logic lives in `src/domain/`
- Do not put persistence logic in handlers
- Shared types live in `src/contracts/`

## Coding Conventions
- Prefer pure functions in domain layer
- Do not introduce new global state without explicit justification
- Reuse existing error types from `src/errors/`

## Safety Rails

### NEVER
- Modify `.env`, lockfiles, or CI secrets without explicit approval
- Remove feature flags without searching all call sites
- Commit without running tests

### ALWAYS
- Show diff before committing
- Update CHANGELOG for user-facing changes

## Verification
- Backend changes: `make test` + `make lint`
- API changes: update contract tests under `tests/contracts/`
- UI changes: capture before/after screenshots

## Compact Instructions
When compressing, preserve in priority order:
1. Architecture decisions (NEVER summarize)
2. Modified files and their key changes
3. Current verification status (pass/fail)
4. Open risks, TODOs, rollback notes
```

## 关键原则

1. **CLAUDE.md 是协作契约** — 只放每次会话都得成立的事
2. **渐进式披露** — 偶尔用的东西不要每次都加载
3. **验证闭环** — 说不清「什么叫做完」的任务不适合丢给 Claude
4. **缓存友好** — 静态内容前置，动态内容后置，不打乱工具定义顺序

## 相关资源

- 原文：Tw93 - 你不知道的 Claude Code
- 健康检查 Skill：tw93/claude-health
- 官方文档：Anthropic Claude Code 文档

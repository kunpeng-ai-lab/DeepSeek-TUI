# DeepSeek-TUI 文件策略引擎 — 交接文档

**交接日期**: 2026-05-13
**交接人**: Claude Code (Agent A)
**接手人**: 下一任 Agent
**工作空间**: 请在独立目录执行 `git clone https://github.com/kunpeng-ai-lab/DeepSeek-TUI.git`，基于 **官方最新 main (v0.8.32)** 开展工作

---

## 1. 项目背景

为 DeepSeek-TUI 增加**文件访问策略引擎**（File Policy Engine），防止 AI agent 在不受控的情况下读取或修改敏感文件（如 `.env`、SSH 私钥、密钥文件等）。

该功能代号 **Track 2**，在内部又称 **harness**。当前实现基于官方 **v0.8.28**，但官方已迭代到 **v0.8.32**，核心任务是将 harness 代码从 v0.8.28 **迁移/重新实现**到 v0.8.32 上，并推送到 `kunpeng-ai-lab/DeepSeek-TUI` 的 `feature/file-policy-engine` 分支。

---

## 2. 已完成工作（v0.8.28 上全部验证通过）

### 2.1 配置解析层
- **文件**: `crates/tui/src/execpolicy/rules.rs`
- **内容**:
  - `ExecPolicyConfig` 新增 `file_rules: BTreeMap<String, FileRuleSet>` 字段
  - `FileRuleSet` 结构体（`allow: Vec<String>`, `deny: Vec<String>`）
  - `evaluate_file(category, path)` 方法：先查具体 category，再回退 `default`，`deny` 优先于 `allow`
  - glob 路径匹配（`*` 匹配单段，`**` 匹配任意层级）
  - `is_file_tool()` / `extract_file_path()` 辅助函数

### 2.2 引擎集成层
- **文件**: `crates/tui/src/core/engine.rs`
- **内容**:
  - `Engine` 新增 `file_policy: Option<Arc<ExecPolicyConfig>>` 字段
  - `Engine::new` 中条件加载：`if config.features.enabled(Feature::FilePolicy) { load_default_policy() } else { None }`
  - `check_file_policy()` 实例方法
  - `evaluate_file_policy()` 静态方法（供 `execute_tool_with_lock` 复用）

### 2.3 拦截路径（串行 + 并行）
- **文件**: `crates/tui/src/core/engine/turn_loop.rs`
- **内容**:
  - `handle_tool_calls_serial`：每个工具调用前调用 `check_file_policy()`，拒绝时 `emit_tool_audit`
  - `execute_parallel_tool_batch`：批量执行前逐个检查，拒绝时不提交到线程池

### 2.4 防御性检查（capacity / 子代理路径）
- **文件**: `crates/tui/src/core/engine/tool_execution.rs`
- **内容**:
  - `execute_tool_with_lock` 入口新增 `file_policy` 参数，在函数开头做防御性拦截
- **文件**: `crates/tui/src/core/engine/capacity_flow.rs`
- **内容**:
  - 适配 `execute_tool_with_lock` 新签名（传 `None`）

### 2.5 Feature Flag
- **文件**: `crates/tui/src/features.rs`
- **内容**:
  - `Feature` enum 新增 `FilePolicy`
  - `FEATURES` 数组新增条目：key `"file_policy"`，stage `Experimental`，default_enabled `true`

### 2.6 测试覆盖
- **文件**: `crates/tui/src/core/engine/tests.rs`
- **数量**: 14 个 file_policy 相关测试全部通过
- **关键测试**:
  - `file_policy_blocks_denied_write_file`
  - `file_policy_allows_permitted_write_file`
  - `file_policy_skips_non_file_tools`
  - `file_policy_falls_back_to_default_category`
  - `file_policy_feature_enabled_by_default`
  - `file_policy_not_loaded_when_feature_disabled`
  - `file_policy_feature_switch_controls_engine_loading`

### 2.7 文档与示例
- `docs/file-policy-design.md` — 架构、配置格式、拦截流程、测试覆盖、未来扩展
- `refs/execpolicy.example.toml` — 常见场景示例配置
- `config.example.toml` — `[features]` 表新增 `file_policy = true`

### 2.8 已打磨的 SKILL.md（Track 1 遗留）
- `skills/workspace-guard/` — 工作区守护
- `skills/cost-saver/` — 成本优化
- `skills/rule-enforcer/` — 破坏性操作审批

---

## 3. 待完成工作（你的任务）

### 3.1 核心任务：迁移到 v0.8.32

**原因**: 官方已从 v0.8.28 迭代到 v0.8.32，`engine.rs`、`turn_loop.rs`、`tool_execution.rs` 等核心文件可能存在结构变化。直接 cherry-pick 不可行。

**做法**: 在 `DeepSeek-TUI-scroll-fix`（或你 clone 的 `kunpeng-ai-lab/DeepSeek-TUI`）的 **最新 main** 上创建 `feature/file-policy-engine` 分支，逐个文件对照 v0.8.28 的实现重新 apply 逻辑。

### 3.2 迁移文件清单

| 文件 | v0.8.28 状态 | 迁移策略 |
|------|-------------|---------|
| `crates/tui/src/execpolicy/rules.rs` | 新增 | 完整复制，注意 `ExecPolicyConfig`/`RuleSet` 需加 `Clone` |
| `crates/tui/src/execpolicy/mod.rs` | 可能新增 | 确保 `pub mod rules;` 和 `load_default_policy()` 导出 |
| `crates/tui/src/core/engine.rs` | 修改 | 在 `Engine` struct 和 `Engine::new` 中插入 `file_policy` |
| `crates/tui/src/core/engine/turn_loop.rs` | 修改 | 在串行/并行工具调用前插入 `check_file_policy()` |
| `crates/tui/src/core/engine/tool_execution.rs` | 修改 | `execute_tool_with_lock` 新增 `file_policy` 参数 |
| `crates/tui/src/core/engine/capacity_flow.rs` | 修改 | 适配 `execute_tool_with_lock` 新签名 |
| `crates/tui/src/core/engine/tests.rs` | 修改 | 新增 14 个测试，注意 `use crate::features::{Feature, Features};` |
| `crates/tui/src/features.rs` | 修改 | `Feature` enum 新增 `FilePolicy` |
| `config.example.toml` | 修改 | `[features]` 表新增 `file_policy = true` |
| `docs/file-policy-design.md` | 新增 | 完整复制到项目 docs/ 目录 |
| `refs/execpolicy.example.toml` | 新增 | 完整复制到项目 refs/ 目录 |

### 3.3 版本差异注意事项

1. **代码版本差异**:
   - v0.8.28 的 `engine.rs` 中 `Engine` struct 的字段顺序、构造方式可能与 v0.8.32 不同
   - `turn_loop.rs` 的串行/并行工具调用函数签名可能已变化
   - `features.rs` 的 `FEATURES` 数组格式需核对

2. **API 兼容性**:
   - v0.8.32 可能已引入新的引擎事件类型或工具执行路径，需确认 `emit_tool_audit` 的调用方式是否一致
   - `Arc<ExecPolicyConfig>` 的赋值方式需适配新 `Engine::new` 的上下文

3. **测试框架**:
   - `engine/tests.rs` 中的测试辅助函数（如 `create_test_options`、`EngineConfig::default()`）在 v0.8.32 中可能已更名或 relocated
   - Windows 平台测试需保留 `#[cfg(windows)]` / `#[cfg(not(windows))]` 条件编译

### 3.4 验证清单

迁移完成后必须执行：

```bash
# 1. 编译检查
cargo check --workspace

# 2. 运行文件策略相关测试
cargo test -p deepseek-tui --bin deepseek-tui file_policy

# 3. 运行所有 engine 测试
cargo test -p deepseek-tui --bin deepseek-tui engine::

# 4. 运行 UI 相关测试（含 composer_arrows_scroll）
cargo test -p deepseek-tui --bin deepseek-tui composer_arrows_scroll
```

---

## 4. 设计文档索引

| 文档 | 路径 | 说明 |
|------|------|------|
| 架构设计 | `docs/file-policy-design.md` | 配置格式、拦截流程、审计事件、测试覆盖 |
| 示例配置 | `refs/execpolicy.example.toml` | `read_file`/`write_file`/`edit_file`/`default` 规则示例 |
| 项目进度 | `STATUS.md` | 所有 Track 的状态跟踪 |
| 迭代快照 | `CHECKPOINT.md` | 每次迭代的详细变更记录 |

---

## 5. 后续强化建议（提高工程能力与编码能力）

### 5.1 工程化改进

1. **按 workspace 隔离策略**
   - 当前策略文件全局统一（`~/.deepseek/execpolicy.toml`）
   - 建议支持项目级 `.deepseek/execpolicy.toml`，优先级高于全局配置
   - 实现：在 `load_default_policy()` 中先查当前 workspace，再查 home 目录

2. **实时重载（Hot Reload）**
   - 当前策略文件在 TUI 启动时加载一次
   - 建议通过文件监听（如 `notify` crate）或 `/reload policy` slash command 实现热更新
   - 避免每次修改策略后重启 TUI

3. **策略生效可视化**
   - 在 TUI footer 或状态栏显示当前文件策略状态（如 `FP: 3 rules active`）
   - 拦截时弹出 toast 提示用户，而非静默阻止

4. **CI 集成测试**
   - 在 GitHub Actions 中增加 `cargo test -p deepseek-tui file_policy` job
   - 确保每次 PR 不破坏策略引擎逻辑

### 5.2 编码能力提高

1. **增加模糊测试（Fuzzing）**
   - 对 `evaluate_file()` 和 `pattern_matches()` 使用 `cargo fuzz`
   - 输入：随机路径字符串 + 随机规则集
   - 验证：永不 panic，决策结果可预期

2. **性能基准测试**
   - 当前 glob 匹配是朴素遍历 O(n*m)
   - 对 1000+ 规则的场景增加 `criterion` benchmark
   - 若成为瓶颈，可引入 `globset` crate 的预编译优化

3. **规则冲突检测**
   - 在策略加载时检测 `allow` 和 `deny` 是否有重叠模式（如 `allow = ["*"]` + `deny = ["*.key"]` 是合理的，但 `allow = ["src/**/*.rs"]` + `deny = ["src/main.rs"]` 应给出 warning）
   - 帮助用户发现配置错误

4. **AskUser 弹窗实现**
   - 当前 `ExecPolicyDecision::AskUser` 被直接视为放行
   - 建议实现真正的确认对话框：当路径无匹配规则时，弹出 "Allow / Deny / Always allow this pattern" 选择
   - 提升用户体验和安全性

---

## 6. 资源与参考

- **官方仓库**: `https://github.com/Hmbown/DeepSeek-TUI`
- **Fork 仓库**: `https://github.com/kunpeng-ai-lab/DeepSeek-TUI`
- **参考实现代码**: `D:\Sherlock\workplace\kimi-workspace\DeepSeek-TUI\`（v0.8.28 + harness 完整代码，非 git 仓库）
- **当前修复分支**: `fix/win10-scroll-fix`（已完成，PR #1578）

---

**交接完成。请接手 Agent 基于 `kunpeng-ai-lab/DeepSeek-TUI` 最新 main 创建 `feature/file-policy-engine` 分支，按上述清单迁移并验证。**

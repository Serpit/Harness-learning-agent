# 外部 Agent 编码委派方案

> Claude Code 做指挥官 + 质检，实际编码通过 `/codex:dispatch` 直接派发给 Codex 执行
> 核心思路：Claude Code 是知识的"编译器"——把 context/ 里的规范编译成自包含指令，通过 `codex exec` 直接交付执行

---

## 一、架构

```
Claude Code（指挥官）                          Codex（执行者）
┌──────────────────────┐                     ┌──────────────────┐
│ /req-new             │                     │                  │
│  → 分析需求          │                     │ codex exec       │
│  → 生成 plan.md      │                     │  → 读 task 文件  │
│                      │   codex exec -      │  → 编码          │
│ /codex:dispatch      │────────────────────→│  → 提交          │
│  → 生成 task 文件    │                     │  → 返回摘要      │
│  → 直接调用 codex    │                     │                  │
│                      │                     └──────────────────┘
│ /codex:verify        │←──检查代码变更──┘
│  → 质检 + 更新进度   │
└──────────────────────┘
```

**用户只需要两步**：`/codex:dispatch` 派发 → `/codex:verify` 验收。不再需要手动复制 task 文件喂给 Codex。

---

## 二、职责分工

| 环节 | 谁做 | 为什么 |
|------|------|--------|
| 需求分析、计划拆解 | Claude Code | 需要理解项目全貌 + context/ |
| 生成自包含 task 文件 | Claude Code | 把 context/ 知识"编译"进 task |
| 调用 Codex 执行 | Claude Code（Bash） | `codex exec` 直接在终端调用 |
| 实际编码 | Codex | token 便宜，不需要项目全貌 |
| 质检验收 | Claude Code | 掌握规范，能做深度检查 |
| 状态管理 | Claude Code | plan.md / process.txt 更新 |
| 经验沉淀 | Claude Code | 写回 context/ |

---

## 三、任务文件格式 — Codex 的输入

task 文件必须**完全自包含**——Codex 不认识 `context/` 体系，所有规范、约束全部内联。

**`requirements/{id}/tasks/task-001.md`** 示例：

```markdown
# Task: 翻译 home/search 模块到意大利语

## 目标
在 res/values-it/strings.xml 中添加 home 和 search 模块的意大利语翻译。

## 要修改的文件
- res/values-it/strings.xml（主要）

## 参考文件
- res/values/strings.xml — 英文原文（source of truth）
- res/values-ja/strings.xml — 参考日语翻译的格式

## 具体要求
1. 翻译 strings.xml 中 name 以 `home_` 和 `search_` 开头的所有条目
2. 占位符规则：
   - %1$s、%d 等保持原样，不要翻译
   - <xliff:g> 标签保持原样
3. 翻译风格：
   - 使用非正式语气（tu，不用 Lei）
   - 食材单位按意大利本地习惯（如 tazza → tazza，不直译 cup）
4. app_name 保持 "CookPal"，不翻译

## 禁止
- 不要修改其他语言的 strings.xml
- 不要修改 .kt 文件
- 不要新增或删除 string key，只添加翻译

## 验收标准
- [ ] 所有 home_* 和 search_* key 都有意大利语翻译
- [ ] 占位符数量和顺序与英文原文一致
- [ ] 无遗漏的 xliff:g 标签
```

---

## 四、plan.md 格式改造

每个任务项包含结构化元数据，供 `/codex:dispatch` 自动生成 task 文件：

```markdown
# 意大利语支持 - 执行计划

## 任务拆解

- [ ] 1. 翻译 home/search 模块（82 条）
  - scope: res/values-it/strings.xml
  - ref: res/values/strings.xml (home_*, search_*)
  - context: i18n-experience.md, compose-patterns.md
  - depends: none

- [ ] 2. 翻译 detail/import 模块（64 条）
  - scope: res/values-it/strings.xml
  - ref: res/values/strings.xml (detail_*, import_*)
  - context: i18n-experience.md
  - depends: none

- [ ] 3. language.kt 添加意大利语选项
  - scope: util/language.kt
  - ref: 看现有语言的写法
  - context: compose-patterns.md
  - depends: none

## 决策记录
- app_name 保持英文 "CookPal"，不翻译
- 食材单位按意大利本地习惯翻译，不直译
- subscription 法律条款标记为 [待专业翻译]
```

字段说明：
- `scope:` — 要修改的文件
- `ref:` — 要参考的文件和范围
- `context:` — Claude Code 生成 task 时从哪些 context/ 文件提取规范内联
- `depends:` — 任务依赖，决定能否并行

---

## 五、命令定义

### /codex:dispatch — 生成任务 + 直接派发给 Codex

**`.claude/commands/codex/dispatch.md`**：

```markdown
为当前需求生成编码任务并直接派发给 Codex 执行。

## 第一步：生成 task 文件

1. 找到当前进行中的需求（扫描 requirements/*/meta.yaml，stage 为 coding）
2. 读取 plan.md，找出所有未完成任务（`- [ ]`）
3. 读取相关的 context/ 规范文件（根据任务项的 context: 字段）
4. 对每个未完成任务生成独立的 task 文件：

   requirements/{id}/tasks/task-{序号}.md

   每个 task 文件必须**完全自包含**，包括：
   - 目标（一句话）
   - 要修改的文件列表
   - 要参考的文件列表
   - 具体要求（从 context/ 提取相关规范，内联写入）
   - 禁止事项
   - 验收标准（checklist 格式）

   关键原则：Codex 只需要读这一个文件 + repo 代码就能干活。

## 第二步：调用 Codex 执行

5. 分析任务间的依赖关系（depends 字段）
6. 对无依赖的任务，可以并行派发（最多 3 个）
7. 对每个任务，执行：

   cat requirements/{id}/tasks/task-{序号}.md | codex exec --full-auto -

   使用 Bash 工具的 run_in_background 参数异步执行。

8. 向用户汇报已派发的任务列表，提示完成后运行 /codex:verify

如果指定了任务编号，只派发该任务。
如果指定了 "generate"，只生成 task 文件不执行。

参数（可选）: $ARGUMENTS
```

### /codex:verify — 验收 Codex 产出

**`.claude/commands/codex/verify.md`**：

```markdown
检查 Codex 完成的编码任务，验收质量并更新进度。

1. 找到当前进行中的需求
2. 读取 requirements/{id}/tasks/ 下的任务文件
3. 用 git diff 查看 Codex 做了哪些代码变更
4. 对每个任务，逐条检查验收标准：
   - 读取 task 文件中的验收标准 checklist
   - 逐条验证（读代码、grep 关键内容）
   - 同时参照 context/ 下的项目规范做额外检查（Codex 可能不知道的规范）
5. 输出验收报告：
   - ✅ task-001: 通过（摘要）
   - ❌ task-003: 占位符 %1$s 在第 42 行被翻译成了 %1$秒
6. 对通过的任务：
   - 更新 plan.md 勾选 [x]
   - 更新 process.txt 追加进度
7. 对未通过的任务：
   - 将修复要求追加到 task 文件末尾的 ## 修复要求 section
   - 询问用户是否立即重新派发给 Codex 修复（再次调用 codex exec）

8. 全部任务通过后：
   - 更新 meta.yaml: stage → review
   - 提示运行 /code-review 做最终审查

验证范围（可选，指定任务编号）: $ARGUMENTS
```

### /codex:fix — 将修复任务重新派发

**`.claude/commands/codex/fix.md`**：

```markdown
将验收未通过的任务重新派发给 Codex 修复。

1. 找到当前进行中的需求
2. 读取指定（或所有）task 文件，检查是否有 ## 修复要求 section
3. 对有修复要求的任务，执行：

   cat requirements/{id}/tasks/task-{序号}.md | codex exec --full-auto -

   Codex 会读到原始要求 + 修复要求，在现有代码基础上修复。

4. 派发完成后提示用户再次运行 /codex:verify

参数（可选，指定任务编号）: $ARGUMENTS
```

---

## 六、Codex 调用细节

### 调用方式

```bash
# 单任务派发
cat requirements/italian-support/tasks/task-001.md | codex exec --full-auto -

# 后台执行（Claude Code 的 Bash 工具 run_in_background=true）
# 可以并行派发多个无依赖的任务
```

### 为什么用 `--full-auto`

- Codex 在 full-auto 模式下不需要用户交互确认
- task 文件已经用"禁止"和"验收标准"做了约束，风险可控
- 质检由 Claude Code 的 `/codex:verify` 兜底

### 为什么用 stdin 而不是 inline prompt

```bash
# ❌ inline prompt — 长内容容易被 shell 截断/转义出问题
codex exec "很长的任务描述..."

# ✅ stdin pipe — 任意长度，无转义问题
cat task-001.md | codex exec --full-auto -
```

---

## 七、完整流程

```
你                    Claude Code                     Codex
│                         │                              │
│ /req-new 支持意大利语    │                              │
│────────────────────────→│                              │
│                         │ 创建 plan.md                 │
│                         │←──确认计划？                  │
│ 确认                    │                              │
│────────────────────────→│                              │
│                         │ stage → coding               │
│                         │                              │
│ /codex:dispatch         │                              │
│────────────────────────→│                              │
│                         │ 1. 读 context/ 规范          │
│                         │ 2. 生成 tasks/task-*.md      │
│                         │ 3. cat task | codex exec -   │
│                         │─────────────────────────────→│
│                         │    （并行派发无依赖任务）      │ 编码 + 提交
│                         │                              │←────┘
│                         │←── 完成通知                   │
│                         │                              │
│ /codex:verify           │                              │
│────────────────────────→│                              │
│                         │ 逐条验收                     │
│                         │ ✅ task-001, task-002 通过    │
│                         │ ❌ task-003 有问题            │
│                         │ → 修复要求写入 task-003.md   │
│                         │                              │
│ /codex:fix              │                              │
│────────────────────────→│                              │
│                         │ cat task-003 | codex exec -  │
│                         │─────────────────────────────→│
│                         │                              │ 修复 + 提交
│                         │←── 完成通知                   │
│                         │                              │
│ /codex:verify 3         │                              │
│────────────────────────→│                              │
│                         │ ✅ task-003 通过              │
│                         │ 更新 plan.md + process.txt   │
│                         │ stage → review               │
│                         │                              │
│ /code-review            │                              │
│────────────────────────→│                              │
│                         │ compose-checker 子代理       │
│                         │ memory-checker 子代理        │
│                         │                              │
│ /req-done               │                              │
│────────────────────────→│                              │
│                         │ 沉淀经验，收尾               │
```

### 对比旧方案

| 步骤 | 旧方案（手动） | 新方案（/codex:*） |
|------|---------------|-------------------|
| 生成 task | /req-dispatch → 手动 | /codex:dispatch → 自动 |
| 执行编码 | 手动复制 task 喂给 Codex | Claude Code 直接调用 codex exec |
| 修复问题 | 手动复制修复要求再喂一次 | /codex:fix 一键重派 |
| 用户操作数 | 每个任务 2-3 次手动操作 | **全程只输命令** |

---

## 八、Token 节省分析

| 环节 | 全在 Claude Code 做 | 委派给 Codex |
|------|---------------------|-------------|
| 读代码文件 | 消耗 Claude context | Codex 承担 |
| 编码试错 | 全部累积在 Claude 对话 | Codex 内部消化 |
| Claude Code 负担 | 编码细节 + 状态管理 | **只有状态管理 + 摘要** |
| 单任务 Claude 消耗 | ~5k tokens | ~500 tokens（dispatch + verify）|

Claude Code token 消耗降到原来的 **~1/10**，context window 保持干净，后期任务质量不退化。

---

## 九、状态流转

```
/req-new            → meta.yaml: stage=analysis       ← Claude Code
  用户确认计划      → meta.yaml: stage=coding          ← Claude Code
/codex:dispatch     → 生成 tasks/task-*.md             ← Claude Code（知识编译）
                    → codex exec 直接执行              ← Claude Code 调用 → Codex 执行
/codex:verify       → 逐条验收                         ← Claude Code（质检）
                    → plan.md: 逐个勾选 [x]            ← Claude Code
                    → process.txt: 追加进度             ← Claude Code
/codex:fix          → 重新派发未通过的任务              ← Claude Code 调用 → Codex 修复
/req-save           → process.txt: 写断点              ← Claude Code（中途离开时）
/req-continue       → 读取状态，继续流程                ← Claude Code
  编码全部通过      → meta.yaml: stage=review          ← Claude Code
/code-review        → compose-checker + memory-checker  ← Claude Code 子代理
/req-done           → meta.yaml: stage=done            ← Claude Code
                    → 沉淀经验到 context/               ← Claude Code
```

---

## 十、settings.json 权限配置

需要在 `.claude/settings.local.json` 的 `permissions.allow` 中添加 Codex 执行权限：

```json
{
  "permissions": {
    "allow": [
      "Bash(codex exec:*)",
      "Bash(cat requirements/*/tasks/*:*)"
    ]
  }
}
```

---

## 十一、目录结构变化

```
requirements/{id}/
├── meta.yaml
├── plan.md
├── process.txt
├── notes.md
├── test-cases.md
└── tasks/                    # 新增：Codex 任务文件目录
    ├── README.md             # 任务总览 + 依赖关系
    ├── task-001.md           # 自包含任务描述
    ├── task-002.md
    └── task-003.md

.claude/commands/
├── codex/                    # 新增：Codex 集成命令
│   ├── dispatch.md           # /codex:dispatch
│   ├── verify.md             # /codex:verify
│   └── fix.md                # /codex:fix
├── req-new.md
├── req-continue.md
├── req-save.md
├── req-done.md
├── code-review.md
├── new-page.md
├── i18n-check.md
└── note.md
```

# 基于 codex-plugin-cc 的外部 Agent 委派方案

> Claude Code 负责需求分析、任务编译、验收和状态管理。
> 实际编码委派给 `codex-plugin-cc` 的 `/codex:rescue`，后台任务通过 `/codex:status` 和 `/codex:result` 管理。

说明：本文按 `codex-plugin-cc` 当前公开能力改造，不再假设 Claude 侧直接用 shell 编排 `codex exec` 作为主路径。

参考：
- `codex-plugin-cc` README: <https://github.com/openai/codex-plugin-cc>
- Raw README: <https://raw.githubusercontent.com/openai/codex-plugin-cc/main/README.md>

---

## 一、架构

```text
Claude Code（指挥官）                         codex-plugin-cc + Codex
┌──────────────────────┐                    ┌─────────────────────────────┐
│ /req-new             │                    │                             │
│  → 分析需求          │                    │ /codex:rescue              │
│  → 生成 plan.md      │                    │  → 通过 codex-rescue 子代理 │
│                      │                    │     把任务交给 Codex        │
│ /codex:prepare       │                    │                             │
│  → 生成 task 文件    │                    │ /codex:status              │
│  → 编译 context 规范 │──────────────────→│  → 查看后台任务状态         │
│                      │ task 文件内容       │                             │
│ /codex:verify        │←──────────────────│ /codex:result              │
│  → 验收代码变更      │   任务结果 / job id │  → 读取最终结果 / session id │
│  → 更新 plan/process │                    │                             │
└──────────────────────┘                    └─────────────────────────────┘
```

核心原则：
- Claude Code 继续做“知识编译器”，把 `context/` 和项目规范压缩成自包含 task 文件。
- 真正的委派入口使用插件已有的 `/codex:rescue`，不自造一套后台调度协议。
- Claude 侧只补插件没有的能力：任务文件生成、验收、状态落盘。

---

## 二、职责分工

| 环节 | 谁做 | 为什么 |
|------|------|--------|
| 需求分析、计划拆解 | Claude Code | 需要项目全貌和长期上下文 |
| 生成自包含 task 文件 | Claude Code | 负责把 `context/` 规范编译进任务 |
| 后台委派与运行 | `codex-plugin-cc` | 插件已提供 `/codex:rescue`、`/codex:status`、`/codex:result` |
| 实际编码 / 调查 / 修复 | Codex | 适合承接实现型工作 |
| 结果验收 | Claude Code | Claude 持有项目规范与验收口径 |
| 状态管理 | Claude Code | 更新 `plan.md`、`process.txt`、`meta.yaml` |

边界要清楚：
- 插件负责“把任务交给 Codex 并跟踪 job”。
- 插件不负责你的 `requirements/` 体系，也不负责自动更新 `plan.md`。
- 插件 README 没承诺“自动提交 commit”，因此本文不把“提交代码”当成默认行为。

---

## 三、任务文件格式

task 文件仍然建议保留，而且必须自包含。

**`requirements/{id}/tasks/task-001.md`** 示例：

```markdown
# Task: 翻译 home/search 模块到意大利语

## 目标
在 `res/values-it/strings.xml` 中添加 `home_*` 和 `search_*` 的意大利语翻译。

## 要修改的文件
- `res/values-it/strings.xml`

## 参考文件
- `res/values/strings.xml`：英文原文，唯一 source of truth
- `res/values-ja/strings.xml`：仅参考格式，不参考措辞

## 具体要求
1. 翻译 `name` 以 `home_` 和 `search_` 开头的所有条目
2. 保留 `%1$s`、`%d` 等占位符，不得改顺序
3. 保留 `<xliff:g>` 标签
4. 使用非正式语气（tu）
5. `app_name` 保持 `CookPal`

## 禁止
- 不要修改其他语言文件
- 不要修改 `.kt` 文件
- 不要新增或删除 string key

## 验收标准
- [ ] 所有 `home_*` 和 `search_*` key 都已翻译
- [ ] 占位符数量和顺序与英文一致
- [ ] 没有遗漏 `xliff:g` 标签
```

关键点：
- task 文件是给 Codex 的，不是给人看的摘要。
- 必须写清“改哪些文件”“不准改什么”“如何验收”。
- 如果要复跑修复，直接在文件末尾追加 `## 修复要求` 即可。

---

## 四、plan.md 元数据约定

`plan.md` 继续保留结构化元数据，但这些字段服务于 Claude 的“任务编译”和“验收”，不是插件内建格式。

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
```

字段说明：
- `scope:` 预期改动文件
- `ref:` 参考文件和范围
- `context:` Claude 生成 task 时要内联的规范来源
- `depends:` 依赖关系，供人工决定是否并行委派

注意：
- 插件支持后台任务管理，但不会理解你的 `depends:`。
- 是否并行，仍由 Claude 或操作者根据任务冲突风险决定。
- 如果多个任务可能修改同一文件，默认不要并行。

---

## 五、命令设计

这里改成“两层命令”：
- 一层是插件现成命令：`/codex:rescue`、`/codex:status`、`/codex:result`、`/codex:cancel`
- 一层是你自己的 Claude 工作流命令：`/codex:prepare`、`/codex:verify`

### 1. `/codex:prepare`

用途：只负责生成 task 文件，不直接执行。

建议放在：
- `.claude/commands/codex/prepare.md`

职责：
1. 找到当前进行中的需求
2. 读取 `plan.md`
3. 找出未完成任务
4. 读取相关 `context/` 规范
5. 为每个任务生成自包含 `tasks/task-xxx.md`
6. 生成 `tasks/README.md`，写明任务编号、scope、依赖和推荐派发顺序

这样做的原因：
- 任务编译是 Claude 擅长的事
- 这一步可审阅、可重跑、可修改
- 避免把“编译任务”和“提交给 Codex”耦合在一个黑盒命令里

### 2. `/codex:rescue`

用途：真正把工作交给插件和 Codex。

这不是你自定义的命令，而是 `codex-plugin-cc` 自带命令。

推荐调用方式：

```text
/codex:rescue --background 请按 requirements/{id}/tasks/task-001.md 执行。先完整阅读任务文件，再查看相关代码，仅修改任务允许的文件。完成后总结：
1. 改了哪些文件
2. 如何满足验收标准
3. 剩余风险
```

说明：
- 使用 `--background` 跑长任务
- 任务正文尽量短，把细节放进 task 文件
- 如果是续跑修复，用 `--resume` 或在提示里明确引用上次任务

### 3. `/codex:status`

用途：查看后台 job 状态。

这是插件现成命令，不需要你自己发明“完成通知”协议。

使用场景：
- 看任务是否还在跑
- 查看最近 job id
- 决定是否需要取消或继续等待

### 4. `/codex:result`

用途：读取已完成 job 的最终输出。

这是验证前的标准入口，建议每次验收前先看一次结果，获取：
- job 最终摘要
- 可能的 session id
- Codex 自述的修改范围与风险

### 5. `/codex:verify`

用途：Claude 侧二次验收。

建议放在：
- `.claude/commands/codex/verify.md`

职责：
1. 找到当前进行中的需求
2. 读取 `tasks/task-*.md`
3. 读取 `/codex:result` 对应 job 的结果摘要
4. 用 `git diff` 和代码检查逐条验证验收标准
5. 通过则更新 `plan.md`、`process.txt`
6. 不通过则把修复要求追加到 task 文件

`/codex:verify` 不是插件自带能力，而是你工作流中的质量闸门。

### 6. `/codex:fix`

如果你还想保留这个命令，建议把它定义成 Claude 侧便捷命令，而不是假装插件内建。

职责改成：
1. 找出带 `## 修复要求` 的 task 文件
2. 调用插件 `/codex:rescue --resume` 或重新发起一次 `/codex:rescue`
3. 在提示词中明确：“基于原任务 + 修复要求继续处理”

不要把它写成固定 shell：

```bash
cat task-001.md | codex exec --full-auto -
```

这条命令当然可以存在，但在“基于 `codex-plugin-cc`”的方案里，它不该是主路径。

---

## 六、推荐工作流

### 标准流

```text
1. /req-new
2. Claude 生成 plan.md
3. /codex:prepare
4. 审阅 task 文件
5. /codex:rescue --background <引用 task 文件路径>
6. /codex:status
7. /codex:result
8. /codex:verify
9. 如失败，补充 ## 修复要求
10. /codex:rescue --resume 或再次发起 rescue
11. /codex:verify
12. 全部通过后 stage -> review
```

### 多任务流

只在下面条件成立时并行：
- 任务没有依赖
- `scope:` 不重叠
- 不会同时改同一个文件

否则默认串行。

原因很简单：
- 插件可以管理多个后台 job
- 但你的 `verify` 还是要把“哪个 diff 属于哪个 task”对应起来
- 如果共享文件并行修改，验收和复跑都会变脆

如果后面要上真正并行，建议补一层 `git worktree`。

---

## 七、验收与结果归因

这是旧版文档最容易失真的部分，需要明确。

不要只写：
- “用 git diff 查看 Codex 做了哪些代码变更”

更稳妥的做法是：
1. 每次派发前记录 task 编号和预期 `scope`
2. 每次拿到 `/codex:result` 后，记录对应 job id
3. 验收时优先检查 `scope` 内文件
4. 若发现超出 `scope` 的改动，直接判为异常或待人工确认

建议在 `tasks/README.md` 里新增：

```markdown
| task | scope | codex job | status | notes |
|------|-------|-----------|--------|-------|
| 001  | res/values-it/strings.xml | task-abc123 | running | first pass |
```

这样 `verify` 才能把 task、job、diff 三者对应起来。

---

## 八、安装与前置条件

按 `codex-plugin-cc` README，前置条件是：
- 已安装 Claude Code
- 已安装插件 marketplace
- 安装 `codex-plugin-cc`
- 本机有可用的 `codex` CLI
- 已通过 `codex login` 登录

参考安装流程：

```bash
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

如果你不想让插件帮装，也可以自己装：

```bash
npm install -g @openai/codex
!codex login
```

来源：`codex-plugin-cc` README。

---

## 九、权限配置

如果走插件主路径，重点不是允许 `Bash(codex exec:*)`，而是保证：
- Claude 能读写你的 `requirements/`、`context/`、`plan.md`
- 插件已安装且可运行
- 本地 `codex` 已认证

如果你仍保留少量 Bash 辅助命令，例如读 diff、生成目录、查看文件，可以在 `.claude/settings.local.json` 中按实际需要增量放行。

不建议把方案核心建立在这两条上：

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

因为这更像“自定义 shell 委派方案”的权限，而不是插件方案的核心配置。

---

## 十、目录结构

```text
requirements/{id}/
├── meta.yaml
├── plan.md
├── process.txt
├── notes.md
├── test-cases.md
└── tasks/
    ├── README.md
    ├── task-001.md
    ├── task-002.md
    └── task-003.md

.claude/commands/
├── codex/
│   ├── prepare.md
│   ├── verify.md
│   └── fix.md
├── req-new.md
├── req-continue.md
├── req-save.md
├── req-done.md
└── code-review.md
```

和旧版相比，最重要的变化是：
- 不再自定义 `/codex:dispatch`
- 使用插件已有的 `/codex:rescue`
- `prepare/verify/fix` 都是 Claude 工作流命令，不冒充插件能力

---

## 十一、最终建议

建议把这套方案表述为：

> Claude Code 负责任务编译与验收，`codex-plugin-cc` 负责把任务稳定地委派给 Codex 并管理后台 job。

不要表述为：

> Claude Code 通过 `/codex:dispatch` 直接调用 `codex exec`，这就是 `codex-plugin-cc` 方案。

这两者不是一回事。

如果后面你确认需要：
- 真正的一键派发
- 自动把 task 文件内容塞给 `/codex:rescue`
- 自动把 job id 记回 `tasks/README.md`

那可以再加一层你自己的 Claude 命令封装，但应当明确写成：
- “基于 `codex-plugin-cc` 的上层工作流封装”

而不是把插件本身写成 shell 调度器。

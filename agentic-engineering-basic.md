# Agentic Engineering 落地方案（基础版）

> 适用工具：Claude Code
> 适用项目：Android 客户端（以 CookPal 为例）
> 核心理念：知识积累 → 复利效应

---

## 一、整体目录结构

```
your-android-project/
├── AGENTS.md                          # AI 入职手册（已有，增强）
├── .claude/
│   ├── settings.json                  # 团队共享配置（hooks + 权限）
│   ├── settings.local.json            # 个人配置（不提交）
│   ├── commands/                      # 自定义命令
│   │   ├── code-review.md             # /code-review 多维度审查
│   │   ├── new-page.md                # /new-page 页面脚手架
│   │   ├── i18n-check.md              # /i18n-check 多语言检查
│   │   ├── note.md                    # /note 快速记录经验
│   │   ├── req-new.md                 # /req-new 创建需求
│   │   ├── req-continue.md            # /req-continue 恢复需求
│   │   ├── req-save.md                # /req-save 保存进度
│   │   ├── req-done.md                # /req-done 完成需求
│   │   ├── test-import.md             # /test-import 导入 QA 测试用例
│   │   └── test-verify.md             # /test-verify AI 静态走查 + 标记手测项
│   └── agents/                        # 专项子代理
│       ├── compose-checker.md         # Compose 规范检查
│       ├── memory-checker.md          # 内存安全检查
│       └── test-case-verifier.md      # 测试用例静态走查
├── context/                           # 知识库（核心资产）
│   └── project/
│       └── cookpal/
│           ├── INDEX.md               # 知识索引入口
│           ├── navigation.md          # PageStack 导航系统
│           ├── compose-patterns.md    # Compose 项目规范
│           ├── testing-guide.md       # 自测验证指南
│           └── experience/
│               ├── ai-mistakes.md     # AI 常犯错误
│               ├── i18n-experience.md # 多语言经验
│               ├── platform-pitfalls.md # 平台踩坑
│               └── testing-lessons.md # 测试经验
└── requirements/                      # 需求记录
    └── {requirement-id}/
        ├── meta.yaml                  # 元信息：阶段、涉及模块
        ├── plan.md                    # 计划：任务拆解
        ├── process.txt                # 进度：做到哪了
        ├── notes.md                   # 笔记：过程中的发现
        └── test-cases.md              # 测试用例 + 执行状态
```

---

## 二、知识库（context/）

### 作用

AI 的"长期记忆"。把项目中 AI 容易搞错的知识沉淀在这里，每次会话自动可用。

### 核心文件

#### context/project/cookpal/INDEX.md

```markdown
# CookPal 项目知识索引

## 架构
- MVVM，单模块 :app，100% Jetpack Compose
- 自定义导航 PageStack（非 Jetpack Navigation），详见 [navigation.md](./navigation.md)

## 核心模式
- Compose 项目特有模式，详见 [compose-patterns.md](./compose-patterns.md)
- 网络层：Server 单例（base/server.kt），返回 Result<T>
- 存储：PreferenceInterface + SharedPreferences
- 无 DI 框架，使用全局单例

## 多语言
- 支持：en(默认)、zh-rTW、ja、ko、it
- 所有文案必须走 R.string.xxx

## 自测验证
- 自测流程和静态走查规范，详见 [testing-guide.md](./testing-guide.md)

## 踩坑记录
- [experience/ai-mistakes.md](./experience/ai-mistakes.md)
- [experience/i18n-experience.md](./experience/i18n-experience.md)
- [experience/platform-pitfalls.md](./experience/platform-pitfalls.md)
- [experience/testing-lessons.md](./experience/testing-lessons.md)
```

#### context/project/cookpal/navigation.md

```markdown
# PageStack 导航系统

## 获取栈
val stack = LocalPageStack.current

## 导航操作
- 跳转：stack.goto(Page.Xxx(参数))
- 返回：stack.pop()
- 带结果：stack.gotoForResult(page)
- 清栈：stack.gotoWithClearStack(page)

## 新增页面步骤
1. 在 main.kt 的 Page sealed class 中添加子类
2. 在 NavDisplay 的 entryProvider 中注册渲染
3. 在 page/{feature}/ 下创建页面文件

## 禁止
- ❌ NavController / NavHost / NavGraph
- ❌ navigate("route") 字符串路由
```

#### context/project/cookpal/compose-patterns.md

```markdown
# CookPal Compose 规范

## 点击
- ✅ Modifier.noIndicationClickableSingle {} （通用点击）
- ✅ Modifier.clickableSingle {} （需要水波纹）
- ❌ Modifier.clickable {}

## 弹窗与 Loading
- val loading = rememberLoadingDialog()
- loading.show() / loading.hide()
- 全局弹窗：GlobalDialogHost / GlobalBottomSheetHost

## 图片
- ✅ UrlImage(url, ...)
- ❌ AsyncImage / Glide

## 颜色与字体
- 直接 Color(0xFFxxxxxx)，不走 MaterialTheme.colorScheme
- 字体大小直接 .sp

## 协程
- val scope = rememberCoroutineScope()
- scope.launch { Server.xxx() }
```

#### experience/ 目录

初始创建空文件占位，格式统一为：

```markdown
# [主题] 经验记录

<!-- 格式：日期 | 问题 | 原因 | 正确做法 -->
```

### 知识路由决策

遇到新信息时，判断该放哪里：

| 判断条件 | 放哪里 |
|---------|--------|
| 所有人都要知道的通用规范？ | AGENTS.md |
| 特定领域的专项知识？ | context/ 对应目录 |
| 当前需求的执行计划？ | requirements/{id}/plan.md |
| 当前需求的实时进度？ | requirements/{id}/process.txt |
| 过程中发现的经验（待沉淀）？ | requirements/{id}/notes.md |

---

## 三、自定义命令（.claude/commands/）

### /code-review — 多维度代码审查

**.claude/commands/code-review.md**：

```markdown
审查当前分支相对于 main 的增量代码变更。

开始前先读取 context/project/cookpal/INDEX.md 了解项目规范。

按以下维度逐一检查：
1. **架构一致性**：是否用了 Server 单例、PageStack 导航、ContextViewModel
2. **Compose 规范**：点击修饰符、UrlImage、Loading、颜色写法
   （参照 context/project/cookpal/compose-patterns.md）
3. **内存安全**：ViewModel 协程作用域、Compose remember/DisposableEffect
4. **多语言**：新增文案是否在 strings.xml 中，5 种语言是否补全
5. **安全**：硬编码密钥、日志敏感信息
6. **性能**：LazyColumn 使用、大图处理

同时参考 context/project/cookpal/experience/ 下的历史踩坑记录。

输出格式：先给总结（✅通过 / ⚠️需修改），再按维度展开具体问题。
```

### /new-page — 页面脚手架

**.claude/commands/new-page.md**：

```markdown
在 CookPal 项目中创建新页面。

先读取 context/project/cookpal/navigation.md 了解导航模式。

步骤：
1. 在 main.kt 的 Page sealed class 中添加新页面定义（含参数）
2. 在 page/ 下创建对应 feature 目录
3. 创建 {Feature}Page.kt（@Composable 函数）
4. 如需 ViewModel，创建 {Feature}VM.kt（继承 ContextViewModel）
5. 在 NavDisplay 的 entryProvider 中注册页面渲染
6. 遵循项目 Compose 规范（context/project/cookpal/compose-patterns.md）

先给页面结构方案确认，再动手写代码。

页面需求: $ARGUMENTS
```

### /i18n-check — 多语言完整性检查

**.claude/commands/i18n-check.md**：

```markdown
检查多语言翻译完整性。

1. 读取 values/strings.xml 中所有 <string name="xxx"> 的 name
2. 逐一对比以下语言文件，找出缺失项：
   - values-zh-rTW/strings.xml
   - values-ja/strings.xml
   - values-ko/strings.xml
   - values-it/strings.xml
3. 检查翻译中是否有丢失的占位符（%s %d %1$s 等）
4. 检查是否有丢失的 <xliff:g> 标签

输出：按语言分组的缺失清单 + 占位符异常清单。
```

### /note — 快速记录经验

**.claude/commands/note.md**：

```markdown
将经验记录到 context/project/cookpal/experience/ 目录。

根据内容判断写入哪个文件：
- AI 相关错误 → ai-mistakes.md
- 多语言相关 → i18n-experience.md
- 平台/系统相关 → platform-pitfalls.md
- 都不合适 → 创建新文件

追加格式：
- **日期**：今天
- **问题**：[发生了什么]
- **原因**：[为什么]
- **正确做法**：[应该怎么做]

记录内容: $ARGUMENTS
```

### /req-new — 创建需求

**.claude/commands/req-new.md**：

```markdown
创建新需求记录。

1. 根据需求描述生成需求 ID（英文短横线格式，如 italian-support）
2. 在 requirements/{id}/ 下创建：
   - meta.yaml（stage: analysis，created: 今天，modules: 待分析）
   - plan.md（空模板）
   - process.txt（记录创建时间）
   - notes.md（空模板）
3. 读取 context/project/cookpal/INDEX.md 了解项目背景
4. 和用户讨论需求范围，输出任务拆解到 plan.md
5. 不要直接开始编码，等用户确认计划

需求描述: $ARGUMENTS
```

### /req-continue — 恢复需求

**.claude/commands/req-continue.md**：

```markdown
恢复之前的需求进度。

1. 如果指定了需求 ID，读取 requirements/{id}/
   如果没指定，扫描 requirements/*/meta.yaml，找出 stage 不是 done 的需求
   - 如果只有一个进行中的需求，直接恢复
   - 如果有多个，列出让用户选择
2. 依次读取：
   - meta.yaml → 当前阶段
   - process.txt → 做到哪了（重点看最后几行）
   - notes.md → 过程中的发现
   - plan.md → 整体计划和完成情况
3. 向用户汇报当前状态，确认接下来做什么

需求 ID（可选）: $ARGUMENTS
```

### /req-save — 保存进度

**.claude/commands/req-save.md**：

```markdown
保存当前需求进度（存档）。

1. 更新 process.txt：追加 [日期 时间] + 当前进度描述
2. 更新 plan.md：标记已完成的任务项为 [x]
3. 更新 meta.yaml：如果阶段有变化，询问用户确认新阶段
   - analysis → coding：计划确认完毕，开始编码
   - coding → review：编码完成，进入审查
   - review → done：审查通过（此时建议使用 /req-done）
4. 检查 notes.md 中是否有标记为"待沉淀"的内容，提醒用户
5. 如果即将中断会话，在 process.txt 末尾写明"下次继续从哪里开始"
```

### /req-done — 完成需求

**.claude/commands/req-done.md**：

```markdown
完成当前需求，执行收尾流程。

1. 更新 meta.yaml：stage → done
2. 更新 process.txt：追加完成时间
3. 更新 plan.md：确认所有任务项已勾选，未勾选的询问用户是否放弃

4. 知识沉淀（核心步骤）：
   - 读取 notes.md，找出标记为"待沉淀"的条目
   - 逐条询问用户确认是否写入 context/experience/
   - 用户确认后，将经验追加到对应的 context/ 文件
   - 每条经验用 --- 分隔，包含日期

5. 代码合并提示：
   - 提示用户执行：
     a. git checkout main
     b. git merge feature/{id}
     c. git branch -d feature/{id}

6. 输出需求总结（做了什么、沉淀了哪些经验）
```

---

## 四、子代理（.claude/agents/）

子代理运行在独立上下文窗口，不污染主对话。适合专项检查任务。

### compose-checker — Compose 规范检查

**.claude/agents/compose-checker.md**：

```yaml
---
name: compose-checker
description: 检查 Compose 代码是否符合 CookPal 项目规范，在代码审查时自动调用
tools: Read, Grep, Glob, Bash
model: sonnet
---

你是 CookPal 项目的 Compose 规范检查专家。

先读取 context/project/cookpal/compose-patterns.md 了解规范。

然后用 `git diff main --name-only -- '*.kt'` 获取增量文件列表，逐一检查是否存在：
1. Modifier.clickable {} 而非 noIndicationClickableSingle/clickableSingle
2. AsyncImage 或 Glide 而非 UrlImage
3. NavController/NavHost 而非 PageStack
4. MaterialTheme.colorScheme 而非直接 Color(0xFF...)
5. 硬编码字符串未放入 strings.xml
6. 未使用 rememberLoadingDialog 的自定义 loading

输出：问题列表（文件:行号 + 问题 + 修复建议）。控制在 1500 token 以内。
```

### memory-checker — 内存安全检查

**.claude/agents/memory-checker.md**：

```yaml
---
name: memory-checker
description: 检查 Android 代码中的内存泄漏和协程安全问题，在代码审查时自动调用
tools: Read, Grep, Glob, Bash
model: sonnet
---

你是 Android 内存安全专家。

用 `git diff main --name-only -- '*.kt'` 获取增量文件列表，逐一检查：
1. ContextViewModel 中是否有不当的 Context 长期持有
2. LaunchedEffect/DisposableEffect 是否正确清理资源
3. rememberCoroutineScope 的作用域是否合理
4. 回调/监听器是否在适当时机移除
5. Bitmap 等大对象是否及时释放
6. Flow collect 是否在正确的生命周期范围内

输出：风险列表（文件:行号 + 风险等级 + 说明 + 修复建议）。控制在 1500 token 以内。
```

---

## 五、Hooks 配置

### .claude/settings.json（团队共享，提交到 git）

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "BRANCH=$(cd $CLAUDE_PROJECT_DIR && git branch --show-current 2>/dev/null); CMD=$(cat | jq -r '.tool_input.command // empty'); if [[ \"$BRANCH\" == \"main\" || \"$BRANCH\" == \"master\" ]] && echo \"$CMD\" | grep -qE 'git (push|commit)'; then echo '当前在受保护分支 '$BRANCH'，请先切到 feature 分支' >&2; exit 2; fi; exit 0"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(cat | jq -r '.tool_input.file_path // empty'); if echo \"$FILE\" | grep -qE '\\.kt$'; then HARDCODED=$(cd $CLAUDE_PROJECT_DIR && grep -nE '[\"]([\u4e00-\u9fff]|[A-Z][a-z]+ [a-z])' \"$FILE\" 2>/dev/null | grep -v 'import\\|package\\|//\\|R.string\\|const\\|Log\\.\\|@Preview\\|testTag\\|MediaType\\|SimpleDateFormat\\|no-i18n' | head -5); if [ -n \"$HARDCODED\" ]; then echo \"⚠️ 可能存在硬编码用户可见文案（含中文或英文短语），请确认是否需要放入 strings.xml。如果是技术常量，可在行尾加 // no-i18n 抑制此提醒\" >&2; exit 2; fi; fi; exit 0"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "MSG='📱 CookPal Android | 命令: /code-review /new-page /i18n-check /note /req-new /req-continue /req-save /req-done'; ACTIVE=''; for meta in $CLAUDE_PROJECT_DIR/requirements/*/meta.yaml; do [ -f \"$meta\" ] || continue; STAGE=$(grep '^stage:' \"$meta\" | awk '{print $2}'); if [ \"$STAGE\" != 'done' ]; then ID=$(grep '^id:' \"$meta\" | awk '{print $2}'); ACTIVE=\"$ACTIVE $ID($STAGE)\"; fi; done; if [ -n \"$ACTIVE\" ]; then MSG=\"$MSG | 📋 进行中需求:$ACTIVE，输入 /req-continue 恢复\"; fi; echo \"$MSG\""
          }
        ]
      }
    ]
  }
}
```

Hook 说明：

| Hook | 触发时机 | 作用 |
|------|---------|------|
| PreToolUse/Bash | AI 执行 shell 命令前 | 阻止在 main/master 分支直接 commit/push |
| PostToolUse/Write | AI 写入 .kt 文件后 | 检测疑似硬编码用户可见文案（中文或英文短语），支持 `// no-i18n` 抑制 |
| SessionStart | 每次会话启动 | 显示可用命令 + 提醒未完成的需求 |

### .claude/settings.local.json（个人，不提交）

> 注意：Claude Code 会自动将 `.claude/settings.local.json` 加入 gitignore。
> 如果你的项目 .gitignore 中没有，请手动添加 `.claude/settings.local.json`，避免误提交权限配置。

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git status:*)",
      "Bash(git branch:*)",
      "Bash(git add:*)",
      "Bash(git checkout:*)",
      "Bash(git commit:*)",
      "Bash(./gradlew:*)",
      "Bash(find:*)",
      "Bash(grep:*)",
      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(wc:*)",
      "Bash(mkdir:*)",
      "Write(requirements/**)",
      "Edit(requirements/**)",
      "Write(context/**)",
      "Edit(context/**)",
      "WebSearch"
    ],
    "deny": [
      "Read(.env)",
      "Read(./local.properties)",
      "Read(**/*keystore*)",
      "Read(**/*signing*)"
    ]
  }
}
```

---

## 六、AGENTS.md 增强

在现有 AGENTS.md 末尾追加：

```markdown
## 项目知识库 (Project Knowledge Base)
- 开始工作前，先读取 context/project/cookpal/INDEX.md 了解项目知识
- 踩坑记录在 context/project/cookpal/experience/，遇到新坑请提醒用户记录
- 单个 experience 文件超过 50 条时，将旧条目归档到 experience/archive/ 子目录
- 可用命令：
  - /code-review — 多维度代码审查
  - /new-page — 创建新页面
  - /i18n-check — 多语言完整性检查
  - /note — 快速记录经验
  - /req-new — 创建新需求
  - /req-continue — 恢复需求进度
  - /req-save — 保存当前进度
  - /req-done — 完成需求（沉淀经验 + 合并提示）
```

---

## 七、需求记录（requirements/）

### 文件说明

每个需求一个目录，4 个文件各管一件事：

| 文件 | 作用 | 类比 |
|------|------|------|
| meta.yaml | 元信息：阶段、涉及模块 | 需求卡片 |
| plan.md | 任务拆解 + 完成状态 | 待办清单 |
| process.txt | 实时进度日志 | 游戏存档 |
| notes.md | 过程中的发现和经验 | 复利原料 |

### meta.yaml 示例

```yaml
id: italian-support
title: CookPal 支持意大利语
created: 2026-04-03
stage: coding          # analysis → planning → coding → review → done
modules:
  - res/values-it/strings.xml
  - util/language.kt
related_context:
  - context/project/cookpal/experience/i18n-experience.md
```

### plan.md 示例

```markdown
# 意大利语支持 - 执行计划

## 任务拆解
- [x] 1. 盘点 strings.xml 缺失条目（共 236 条）
- [x] 2. 翻译 home/search 模块（82 条）
- [ ] 3. 翻译 detail/import 模块（64 条）
- [ ] 4. 翻译 login/pay/subscription 模块（55 条）
- [ ] 5. 翻译 guide/dialog/settings 模块（35 条）
- [ ] 6. language.kt 添加意大利语选项
- [ ] 7. 完整性检查（占位符、xliff 标签）
- [ ] 8. UI 溢出检查

## 决策记录
- app_name 保持英文 "CookPal"，不翻译
- 食材单位按意大利本地习惯翻译，不直译
- subscription 法律条款标记为 [待专业翻译]
```

### process.txt 示例

```
[2026-04-03 14:30] 开始需求，盘点缺失条目完成，共 236 条
[2026-04-03 15:20] home/search 模块 82 条翻译完成，已提交
[2026-04-03 16:00] 开始 detail/import 模块，发现占位符语序问题
[2026-04-03 16:30] ⚠️ 会话中断，detail/import 翻译到第 40 条（共 64 条）
--- 下次继续从 detail/import 第 41 条开始 ---
```

### notes.md 示例

```markdown
# 意大利语支持 - 过程笔记

## 发现
- 意大利语 "you" 有正式/非正式（Lei vs tu），统一用非正式 "tu"
- recipe_step_format 的 %1$s 在意大利语中语序要调整
- "Sign in with Google" 官方翻译是 "Accedi con Google"

## 待沉淀到 context/
- [ ] tu/Lei 选择 → i18n-experience.md
- [ ] 占位符语序问题 → i18n-experience.md
- [ ] 第三方登录按钮官方翻译 → i18n-experience.md
```

### 知识流转闭环

```
需求开发中 → notes.md 记录发现
                │
需求完成后 → 提取通用经验到 context/experience/
                │
下次需求   → AI 自动读取 context/，站在更高起点
```

---

## 八、典型工作流程

### 新需求

```
/req-new 支持意大利语
```
→ AI 创建 requirements/italian-support/，分析范围，输出计划让你确认

### 编码过程

```
确认计划，开始翻译 home 模块
```
→ AI 按计划执行，你确认每批产出

### AI 犯错时

```
/note AI 翻译时把 %1$s 占位符破坏了，翻译成了 %1$秒
```
→ 记录到 context/project/cookpal/experience/ai-mistakes.md

### 中途要走

```
/req-save
```
→ 进度写入 process.txt，下次可恢复

### 第二天继续

```
/req-continue
```
→ AI 读取存档，汇报进度，从断点继续

### 代码审查

```
/code-review
```
→ 主对话派出 compose-checker 和 memory-checker 并行检查，汇总报告

### 需求完成

```
/req-done
```
→ AI 沉淀 notes.md 中的经验到 context/，提示合并分支

---

## 九、知识库膨胀管理

随着经验积累，experience 文件会越来越大，消耗 AI 的上下文窗口。约定：

- 单个 experience 文件超过 **50 条**时，将 6 个月前的旧条目移到 `experience/archive/` 子目录
- archive/ 下的文件 AI 不会主动读取，但需要时可以搜索
- INDEX.md 中只引用活跃的 experience 文件，不引用 archive/

```
experience/
├── ai-mistakes.md              # 活跃（最近 50 条）
├── i18n-experience.md          # 活跃
├── platform-pitfalls.md        # 活跃
└── archive/                    # 归档（AI 不主动读取）
    ├── ai-mistakes-2026H1.md
    └── i18n-experience-2026H1.md
```

---

## 十、requirements/ 的 Git 策略

`requirements/` 目录跟随 feature 分支提交到 git：

- 需求开发中：requirements/{id}/ 在 feature 分支上，随代码一起提交
- 需求完成后：merge 到 main，成为历史存档
- `process.txt` 只在 `/req-save` 时提交，不要每次更新都 commit

这样做的好处：
- 团队成员可以通过 git log 查看需求的完整历史
- 需求记录和代码变更在同一个分支上，方便追溯
- main 分支上的 requirements/ 是所有已完成需求的归档

---

## 十一、落地节奏

| 阶段 | 时间 | 做什么 |
|------|------|--------|
| 第 1 天 | 30 分钟 | 建 context/ 目录，写 INDEX.md + navigation.md + compose-patterns.md |
| 第 1 天 | 10 分钟 | AGENTS.md 末尾追加知识库引用 |
| 第 2 天 | 20 分钟 | 创建 commands/（code-review、note、req-new、req-continue、req-save） |
| 第 2 天 | 10 分钟 | 配置 .claude/settings.json（hooks + 权限） |
| 第 3 天 | 15 分钟 | 创建 agents/（compose-checker、memory-checker） |
| 持续 | 每次 2 分钟 | AI 犯错时 /note 记录，需求结束时沉淀经验 |

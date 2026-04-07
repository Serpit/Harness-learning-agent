# Agentic Engineering 方案升级（融合 OpenSpec 编译器纪律）

> 基于 [agentic-engineering-basic.md](./agentic-engineering-basic.md)
> 借鉴对象：[OpenSpec](https://github.com/Fission-AI/OpenSpec)
> 核心思路：不引入 OpenSpec CLI，只**借格式 + 借机制**，把"和 AI 写需求"当成编译器在处理

---

## 一、为什么要升级

OpenSpec 的"魔力"不在于 spec 这个词，而在于它把以下模糊判断**降维成机器可校验的操作**：

| 模糊判断 | OpenSpec 的降维 |
|---|---|
| "这个需求清楚了吗？" | SHALL/MUST + ≥1 Scenario，validator 强制 |
| "现在该做哪一步？" | DAG status 查询 |
| "AI 该看哪些上下文？" | `openspec instructions` 显式列 dependencies |
| "这次改了什么？" | ADDED/MODIFIED/REMOVED 三段式 |
| "怎么合并回主文档？" | 程序按 Requirement name 替换 block |

基础版方案的短板正好对应：
1. `plan.md` 是任务清单，不是行为契约 → AI 做完无法自检
2. 没有"系统当前行为"的中心化文档 → 只能现场读代码
3. stage 是状态机式硬门禁 → AI 经常误判阶段
4. 提示词 context 和产物指令混在一起 → AI 经常把规范条文复述进产物

---

## 二、改造总览（按收益排序）

| 顺序 | 改造 | 投入 | 收益 |
|---|---|---|---|
| 1 | acceptance.md + validator + hook | 1 小时 | ⭐⭐⭐⭐⭐ |
| 2 | 两段式提示词改造 `/req-new` 等 | 30 分钟 | ⭐⭐⭐⭐ |
| 3 | 新增 `/req-verify` 命令 | 30 分钟 | ⭐⭐⭐⭐ |
| 4 | meta.yaml + req-status.sh | 1 小时 | ⭐⭐⭐ |
| 5 | behaviors/ + merge 脚本 | 半天 | ⭐⭐⭐⭐⭐ |
| 6 | compose-checker 升级 | 20 分钟 | ⭐⭐⭐ |

---

## 三、改造一：acceptance.md（可校验的验收契约）

### 目录变化

```
requirements/italian-support/
├── meta.yaml
├── plan.md          ← 任务（怎么做）
├── acceptance.md    ← 新增：验收契约（做完长什么样）
├── process.txt
└── notes.md
```

### acceptance.md 格式（严格锁死）

```markdown
# 意大利语支持 - 验收契约

## Requirements

### Requirement: 意大利语文案完整性
应用 MUST 在意大利语模式下显示所有用户可见文案，不得出现英文回退或 missing key。

#### Scenario: 切换到意大利语
- GIVEN 用户在设置中选择 "Italiano"
- WHEN 应用重启或语言生效
- THEN 所有页面的标题、按钮、提示文案均为意大利语
- AND 没有任何位置显示英文原文

#### Scenario: 占位符正确替换
- GIVEN recipe_step_format 中含 %1$s 占位符
- WHEN 渲染步骤 "Step 1: Mix"
- THEN 占位符被正确替换，语序符合意大利语习惯

### Requirement: 语言切换持久化
应用 MUST 在重启后保留用户选择的意大利语偏好。

#### Scenario: 重启后保持
- GIVEN 用户已切换到意大利语
- WHEN 杀掉进程并重新打开应用
- THEN 应用以意大利语启动
```

### scripts/validate-acceptance.sh

```bash
#!/usr/bin/env bash
set -e
FILE="$1"
[ -f "$FILE" ] || { echo "❌ acceptance.md 不存在"; exit 1; }

REQ_COUNT=$(grep -cE '^### Requirement:' "$FILE" || true)
[ "$REQ_COUNT" -ge 1 ] || { echo "❌ 至少需要一个 ### Requirement:"; exit 1; }

awk '
  /^### Requirement:/ {
    if (name) check();
    name=$0; has_must=0; scen=0; next
  }
  /SHALL|MUST/ { has_must=1 }
  /^#### Scenario:/ { scen++ }
  END { if (name) check() }
  function check() {
    if (!has_must) { print "❌ "name" 缺少 SHALL/MUST"; exit 2 }
    if (scen < 1)  { print "❌ "name" 至少需要一个 #### Scenario"; exit 2 }
  }
' "$FILE"

echo "✅ acceptance.md 校验通过 ($REQ_COUNT 条需求)"
```

### .claude/settings.json hook

```json
{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "command",
    "command": "FILE=$(cat | jq -r '.tool_input.file_path // empty'); if echo \"$FILE\" | grep -q 'requirements/.*/acceptance\\.md$'; then $CLAUDE_PROJECT_DIR/scripts/validate-acceptance.sh \"$FILE\" >&2 || exit 2; fi; exit 0"
  }]
}
```

**效果**：模糊需求进不了仓库。AI 写出不合格 acceptance（没 MUST、没 Scenario），hook 直接打回。

---

## 四、改造二：两段式提示词

### 原则
- `<context>`：给 AI 的约束，**禁止写进文件**
- `<template>`：产物骨架，AI 只填空

### .claude/commands/req-new.md（重写）

```markdown
创建新需求记录。

<context>
以下是给你的约束，不要写进任何文件：
- 项目背景见 context/project/cookpal/INDEX.md
- 需求 ID 用 kebab-case 英文短横线
- 阶段从 analysis 开始
- acceptance.md 必须满足：每个 Requirement 含 SHALL/MUST + ≥1 Scenario
- 不要在文件里复述这些约束本身
</context>

<steps>
1. 读 context/project/cookpal/INDEX.md（只读，不复述）
2. 和用户对话，确认需求范围
3. 生成 ID，创建 requirements/{id}/ 目录
4. 按 <template> 创建 4 个文件
5. 创建后自动跑 ./scripts/validate-acceptance.sh，失败必须修复
6. 等用户确认计划，不要直接编码
</steps>

<template name="meta.yaml">
id: {id}
title: {一句话标题}
created: {today}
artifacts:
  plan: missing
  acceptance: missing
  implementation: missing
  verification: missing
</template>

<template name="acceptance.md">
# {title} - 验收契约

## Requirements

### Requirement: {行为名}
系统 MUST {做什么}。

#### Scenario: {场景名}
- GIVEN ...
- WHEN ...
- THEN ...
</template>

需求描述: $ARGUMENTS
```

同样改造：`/new-page`、`/req-done`。

---

## 五、改造三：/req-verify 命令（新增）

### .claude/commands/req-verify.md

```markdown
对照 acceptance.md 自检当前需求实现完成度。

<steps>
1. 读 requirements/{当前id}/acceptance.md
2. 对每个 ### Requirement 和 #### Scenario，逐一判定：
   - ✅ 已满足：在代码/资源中能找到对应实现
   - ⚠️  部分满足：实现存在但有疑点（列出疑点）
   - ❌ 未满足：找不到对应实现
3. 对 ⚠️ 和 ❌ 项，定位到具体文件:行号
4. 输出表格：Scenario | 状态 | 证据/缺口
5. 全部 ✅ 时建议运行 /req-done
6. 有 ❌ 时**禁止**调用 /req-done
</steps>

<output_format>
| # | Scenario | 状态 | 证据/缺口 |
|---|----------|------|-----------|
| 1 | 切换到意大利语 | ✅ | values-it/strings.xml:1-236 |
| 2 | 占位符语序 | ⚠️ | recipe_step_format 已翻译，未人工校对 |
| 3 | 重启后保持 | ❌ | language.kt 未实现持久化 |
</output_format>
```

并在 `/req-done` 开头加：

```markdown
0. 先调用 /req-verify，如有 ❌ 项，中止流程
```

---

## 六、改造四：meta.yaml + req-status.sh

### meta.yaml 新格式

```yaml
id: italian-support
title: CookPal 支持意大利语
created: 2026-04-03
artifacts:
  plan: done           # done | ready | blocked | missing
  acceptance: done
  implementation: in_progress
  verification: blocked
notes_to_distill: 3
```

### scripts/req-status.sh

让 AI 不用猜阶段，直接查询：

```
$ ./scripts/req-status.sh italian-support
✅ plan          (10 tasks, 6 done)
✅ acceptance    (3 requirements, 7 scenarios)
🔄 implementation (in progress)
⏸  verification  (blocked: implementation not done)
📝 待沉淀经验: 3 条
建议下一步: 继续 implementation
```

`/req-continue` 改成先调这个脚本。

---

## 七、改造五：behaviors/（行为复利层，最大价值）

### 目录变化

```
context/project/cookpal/
├── INDEX.md
├── navigation.md
├── compose-patterns.md
├── behaviors/                    ← 新增
│   ├── auth.md                   ← 登录系统当前行为
│   ├── i18n.md                   ← 多语言系统当前行为
│   ├── recipe.md
│   └── ...
└── experience/
```

每个 `behaviors/{feature}.md` 用和 acceptance.md 完全一样的格式（Requirement + Scenario）。**这就是项目的"主 spec"**。

### /req-done 增加合并步骤

```markdown
4.5 更新 behaviors/（关键步骤）
   - 读 acceptance.md 中所有 Requirement
   - 询问：这些需求对应 behaviors/ 中的哪个文件？
   - 对每条 Requirement 生成三段式补丁：
     ## ADDED Requirements
     ## MODIFIED Requirements
     ## REMOVED Requirements
   - 用户确认后，机械合并到 behaviors/{feature}.md
   - 合并方式：按 ### Requirement: <名> 整块替换/追加/删除
```

### scripts/merge-behaviors.sh

50 行 awk/python 即可，按 Requirement name 做 block 级 CRUD。

### INDEX.md 引用

```markdown
## 系统当前行为
- 登录与会话：[behaviors/auth.md](./behaviors/auth.md)
- 多语言：[behaviors/i18n.md](./behaviors/i18n.md)
```

**效果**：下次 AI 接到"修改登录逻辑"，不需要 grep 代码猜，直接读 `behaviors/auth.md` 就有完整契约。这是基础版最缺的复利层。

---

## 八、改造六：compose-checker 升级

在 `.claude/agents/compose-checker.md` 末尾追加：

```markdown
额外检查（语义+回归）：
1. 读取当前 feature 分支对应的 requirements/*/acceptance.md
2. 对每个 Scenario，在增量代码中找到对应实现位置
3. 读取 context/project/cookpal/behaviors/ 中可能受影响的文件
4. 报告：本次改动是否破坏了 behaviors 中已有的 Requirement
```

子代理从"语法检查器"升级为"语义+回归检查器"。

---

## 九、改造后的目录全貌

```
your-android-project/
├── AGENTS.md
├── .claude/
│   ├── settings.json                  ← 增加 acceptance hook
│   ├── commands/
│   │   ├── req-new.md                 ← 两段式
│   │   ├── req-verify.md              ← 新增
│   │   ├── req-done.md                ← 增加 behaviors 合并
│   │   └── ...
│   └── agents/
│       └── compose-checker.md         ← 增加 spec 对照
├── scripts/                           ← 新增
│   ├── validate-acceptance.sh
│   ├── req-status.sh
│   └── merge-behaviors.sh
├── context/project/cookpal/
│   ├── INDEX.md                       ← 引用 behaviors
│   ├── behaviors/                     ← 新增
│   │   ├── auth.md
│   │   ├── i18n.md
│   │   └── ...
│   └── experience/
└── requirements/
    └── {id}/
        ├── meta.yaml                  ← artifacts 字段
        ├── plan.md
        ├── acceptance.md              ← 新增
        ├── process.txt
        └── notes.md
```

---

## 十、改造前后差异

| 维度 | 改造前 | 改造后 |
|---|---|---|
| 需求是否可机器校验 | ❌ 自由 markdown | ✅ validator 强制 |
| AI 能否自检完成度 | ❌ 只能手测 | ✅ /req-verify 对账 |
| 提示词是否区分约束/产物 | ❌ 混在一起 | ✅ context vs template |
| 系统当前行为是否沉淀 | ❌ 只在代码里 | ✅ behaviors/ 中心化 |
| 阶段判断 | ❌ AI 猜 | ✅ 脚本算 |
| 经验复利 | ✅ | ✅ |
| 行为复利 | ❌ | ✅ |
| 工程护栏 | ✅ | ✅ |

最终组合：**经验复利 + 行为复利 + 工程护栏 + 编译器纪律**——这是 OpenSpec 自己达不到、而升级版方案独占的完整度。

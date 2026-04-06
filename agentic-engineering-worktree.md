# Agentic Engineering 落地方案（Worktree 并行版）

> 基于基础版的升级，新增 git worktree 并行开发能力
> 适用场景：同时推进多个需求、需要隔离工作空间、团队协作
> 前置条件：已阅读并理解基础版方案

---

## 与基础版的差异总览

| 维度 | 基础版 | Worktree 并行版 |
|------|--------|-----------------|
| 工作方式 | 单分支串行，一次做一个需求 | 多 worktree 并行，同时推进多个需求 |
| 分支策略 | 手动切分支 | 每个需求自动创建独立 worktree + 分支 |
| 上下文隔离 | 共用一个工作目录 | 每个需求独立目录，互不干扰 |
| 知识共享 | 直接读取 context/ | 所有 worktree 继承 main 的 context/ |
| 适合场景 | 个人开发、需求不多 | 多需求并行、团队协作、需要隔离 |

---

## 一、核心概念

### 单仓模式：共享知识 + 隔离执行

```
cookpal-android/  (main 分支 — 知识中枢)
├── AGENTS.md                    ← 所有 worktree 共享
├── context/                     ← 所有 worktree 共享
├── .claude/                     ← 所有 worktree 共享
│
├── .worktrees/
│   ├── italian-support/         ← 终端 1：做意大利语
│   │   └── (feature/italian-support 分支)
│   ├── fix-login-bug/           ← 终端 2：修登录 bug
│   │   └── (feature/fix-login-bug 分支)
│   └── home-performance/        ← 终端 3：首页性能优化
│       └── (feature/home-performance 分支)
```

每个 worktree 是独立的工作目录、独立的分支，但共享同一个 .git 仓库。
创建 worktree 时会继承 main 分支的所有文件（包括 context/、.claude/、AGENTS.md）。

### 知识流向

```
main (知识中枢)
  │
  ├──→ worktree A 继承知识，开发需求 A
  │     │
  │     └── 发现新经验 ──→ 提交到 main 的 context/
  │                              │
  ├──→ worktree B 继承知识 ←─────┘ (merge main 获取最新)
  │     │
  │     └── 发现新经验 ──→ 提交到 main 的 context/
  │
  └──→ 未来的 worktree C 自动拥有 A、B 积累的所有经验
```

---

## 二、目录结构（在基础版上新增）

```
cookpal-android/
├── (基础版所有内容不变)
├── .worktrees/                    # 🆕 worktree 工作目录（gitignore）
│   ├── italian-support/
│   │   ├── (继承 main 的所有文件)
│   │   └── requirements/
│   │       └── italian-support/   # 需求记录跟着分支走
│   └── fix-login-bug/
│       ├── (继承 main 的所有文件)
│       └── requirements/
│           └── fix-login-bug/
```

### .gitignore 新增

```gitignore
# worktree 目录不提交（它是本地工作空间）
.worktrees/
```

---

## 三、升级的命令

### /req-new 升级版（支持 worktree）

**.claude/commands/req-new.md** 替换为：

```markdown
创建新需求记录，可选创建独立 worktree。

1. 根据需求描述生成需求 ID（英文短横线格式，如 italian-support）

2. 询问用户是否创建独立 worktree：
   - 如果是（推荐并行开发时使用）：
     执行 git worktree add .worktrees/{id} -b feature/{id}
     提示用户：请在新终端执行 cd .worktrees/{id} && claude
   - 如果否：
     执行 git checkout -b feature/{id}

3. 在 requirements/{id}/ 下创建：
   - meta.yaml（stage: analysis，created: 今天）
   - plan.md（空模板）
   - process.txt（记录创建时间 + worktree 路径）
   - notes.md（空模板）

4. 读取 context/project/cookpal/INDEX.md 了解项目背景
5. 和用户讨论需求范围，输出任务拆解到 plan.md
6. 不要直接开始编码，等用户确认计划

需求描述: $ARGUMENTS
```

### /req-done 新增（需求完成 + 清理）

**.claude/commands/req-done.md**：

```markdown
完成当前需求，执行收尾流程。

1. 更新 meta.yaml：stage → done
2. 更新 process.txt：追加完成时间
3. 更新 plan.md：确认所有任务项已勾选

4. 知识沉淀：
   - 读取 notes.md，找出标记为"待沉淀"的条目
   - 提醒用户确认哪些要写入 context/experience/
   - 用户确认后，将经验写入对应的 context/ 文件

5. 代码合并提示：
   - 如果在 worktree 中，提示用户：
     a. cd 回主目录
     b. git merge feature/{id}
     c. git worktree remove .worktrees/{id}
     d. git branch -d feature/{id}
   - 如果不在 worktree 中，提示用户：
     a. git checkout main
     b. git merge feature/{id}
     c. git branch -d feature/{id}

6. 输出需求总结（做了什么、耗时、沉淀了哪些经验）
```

### /req-sync 新增（同步知识到 worktree）

**.claude/commands/req-sync.md**：

```markdown
将 main 分支的最新知识同步到当前 worktree。

适用场景：其他 worktree 沉淀了新经验到 main 的 context/，
当前 worktree 需要获取这些最新知识。

执行：
1. 检查工作区是否干净（git status --porcelain），如果有未提交的变更，
   提示用户先执行 /req-save 提交进度，或 git stash 暂存
2. git fetch origin main（如果有远程）
3. git merge main --no-edit
4. 如果有冲突：
   - context/experience/ 下的文件：保留双方内容（两边都是 append-only 的经验记录，合并即可）
   - 其他 context/ 文件（INDEX.md 等）：取 main 版本
   - 非 context/ 文件：提示用户手动处理
4. 输出同步了哪些新知识文件

> 为减少冲突，experience 文件约定为 append-only 格式，每条经验用 --- 分隔。
> 不同 worktree 尽量写入不同的 experience 文件（如 i18n-experience.md vs performance-experience.md）。
```

---

## 四、升级的 Hooks

### .claude/settings.json 新增/修改

在基础版的 hooks 基础上，新增 worktree 相关的 hook：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "BRANCH=$(cd $CLAUDE_PROJECT_DIR && git branch --show-current 2>/dev/null); CMD=$(cat | jq -r '.tool_input.command // empty'); if [[ \"$BRANCH\" == \"main\" || \"$BRANCH\" == \"master\" ]] && echo \"$CMD\" | grep -qE 'git (push|commit)'; then echo '当前在受保护分支 '$BRANCH'，请先切到 feature 分支或创建 worktree（/req-new）' >&2; exit 2; fi; exit 0"
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
            "command": "FILE=$(cat | jq -r '.tool_input.file_path // empty'); if echo \"$FILE\" | grep -qE '\\.kt$'; then HARDCODED=$(cd $CLAUDE_PROJECT_DIR && grep -nE '[\"]([\u4e00-\u9fff]|[A-Z][a-z]+ [a-z])' \"$FILE\" 2>/dev/null | grep -v 'import\\|package\\|//\\|R.string\\|const\\|Log\\.\\|@Preview\\|testTag\\|MediaType\\|SimpleDateFormat\\|no-i18n' | head -5); if [ -n \"$HARDCODED\" ]; then echo \"⚠️ 可能存在硬编码用户可见文案，请确认是否需要放入 strings.xml。行尾加 // no-i18n 可抑制此提醒\" >&2; exit 2; fi; fi; exit 0"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "BRANCH=$(cd $CLAUDE_PROJECT_DIR && git branch --show-current 2>/dev/null); IS_WT=$(cd $CLAUDE_PROJECT_DIR && git worktree list 2>/dev/null | wc -l); MSG=\"📱 CookPal Android | 分支: $BRANCH\"; if [ \"$IS_WT\" -gt 1 ] 2>/dev/null; then MSG=\"$MSG (worktree)\"; fi; MSG=\"$MSG | 命令: /req-new /req-continue /req-save /req-done /req-sync /code-review /new-page /i18n-check /note\"; ACTIVE=''; for meta in $CLAUDE_PROJECT_DIR/requirements/*/meta.yaml; do [ -f \"$meta\" ] || continue; STAGE=$(grep '^stage:' \"$meta\" | awk '{print $2}'); if [ \"$STAGE\" != 'done' ]; then ID=$(grep '^id:' \"$meta\" | awk '{print $2}'); ACTIVE=\"$ACTIVE $ID($STAGE)\"; fi; done; if [ -n \"$ACTIVE\" ]; then MSG=\"$MSG | 📋 进行中:$ACTIVE\"; fi; echo \"$MSG\""
          }
        ]
      }
    ]
  }
}
```

### .claude/settings.local.json 补充说明

沿用基础版的权限配置，额外新增 worktree 相关权限：

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
      "Bash(git merge:*)",
      "Bash(git worktree:*)",
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

相比基础版新增了：`Bash(git merge:*)`、`Bash(git worktree:*)`。

---

## 五、并行开发工作流

### 场景：同时做两个需求

**需求 A**：支持意大利语（你主导）
**需求 B**：首页性能优化（AI 跑着，你偶尔看一眼）

#### Step 1：创建两个 worktree

```bash
# 终端 1
cd /Users/guansiping/workspace/cookpal-android
claude
> /req-new 支持意大利语
# AI 询问是否创建 worktree → 选是
# AI 执行：git worktree add .worktrees/italian-support -b feature/italian-support
# AI 提示：请在新终端执行 cd .worktrees/italian-support && claude
```

```bash
# 终端 2
cd /Users/guansiping/workspace/cookpal-android
claude
> /req-new 首页性能优化-列表滑动卡顿
# 同样创建 worktree
# cd .worktrees/home-performance && claude
```

#### Step 2：各自独立工作

```bash
# 终端 1（意大利语）
cd .worktrees/italian-support
claude
> /req-continue
> 开始翻译 home 模块
> ...
> /req-save   # 存档

# 终端 2（性能优化）
cd .worktrees/home-performance
claude
> /req-continue
> 分析首页 LazyColumn 的性能瓶颈
> ...
> /req-save   # 存档
```

两个终端互不干扰，各自在自己的分支上提交代码。

#### Step 3：知识同步

意大利语 worktree 里发现了一个 i18n 的坑，想让性能优化 worktree 也知道：

```bash
# 终端 1（意大利语 worktree）
> /note 意大利语文案比英语长 25%，PayPage 按钮会溢出

# 这条经验写入了 context/，但只在当前分支
# 需要同步到 main，让其他 worktree 也能拿到
```

```bash
# 回到主目录，把 context/ 更新合到 main
cd /Users/guansiping/workspace/cookpal-android
git checkout main
git checkout feature/italian-support -- context/
git commit -m "a-意大利语翻译经验沉淀"
git checkout -    # 切回之前的分支
```

```bash
# 终端 2（性能优化 worktree）
> /req-sync
# AI 执行 git merge main，拿到最新的 context/
```

#### Step 4：需求完成，合并清理

```bash
# 终端 1（意大利语做完了）
> /req-done
# AI 执行收尾：沉淀经验、提示合并步骤

# 回到主目录合并
cd /Users/guansiping/workspace/cookpal-android
git merge feature/italian-support
git worktree remove .worktrees/italian-support
git branch -d feature/italian-support
```

---

## 六、知识同步策略

并行开发时，知识同步是最关键的问题。三种策略按场景选择：

### 策略 A：即时同步（推荐）

发现重要经验时立刻同步到 main：

```bash
# 在 worktree 中，先提交 context/ 的变更
git add context/
git commit -m "a-xxx经验沉淀"

# 回主目录，从 feature 分支拉取 context/ 更新
cd /Users/guansiping/workspace/cookpal-android
git checkout main
git checkout feature/xxx -- context/
git commit -m "a-同步xxx经验到main"
git checkout -    # 切回之前的分支
```

> ⚠️ 注意：`git cherry-pick` 不支持 `-- path` 语法，不要用 `git cherry-pick <hash> -- context/`。
> 正确做法是用 `git checkout <branch> -- context/` 拉取指定分支的文件。

适合：发现了影响其他需求的关键经验（比如平台 bug、API 行为变更）。

### 策略 B：需求完成时批量同步

需求做完后，通过 /req-done 一次性把 notes.md 中的经验沉淀到 context/，
然后合并分支时 context/ 的更新自然进入 main。

适合：大部分场景。经验不紧急，等需求做完再同步即可。

### 策略 C：定期同步

约定每天下班前，各 worktree 执行一次 /req-sync 拉取 main 最新知识。

适合：团队多人协作，每个人在自己的 worktree 里工作。

---

## 七、团队协作模式

当团队多人使用时，单仓模式的优势最大化：

```
共享仓库 (main)
├── AGENTS.md          ← 团队统一的 AI 规范
├── context/           ← 团队共享的知识库
├── .claude/           ← 团队统一的工具配置
│
├── 开发者 A 的 worktree (feature/italian-support)
├── 开发者 B 的 worktree (feature/home-performance)
└── 开发者 C 的 worktree (bugfix/login-crash)
```

### 知识积累的复利效应

```
第 1 周：开发者 A 做意大利语，踩了 3 个坑 → 沉淀到 context/
第 2 周：开发者 B 做法语，AI 自动避开这 3 个坑 → 只踩了 1 个新坑 → 沉淀
第 3 周：开发者 C 做德语，AI 自动避开 4 个坑 → 0 个新坑
```

每个人的经验通过 context/ + git merge 自动传递给所有人的 AI。
这就是文章说的"一个人踩坑，全团队免疫"。

### 团队 PR 流程

建议 context/ 的更新也走 PR review：

```
feature/italian-support
  └── 包含 context/experience/i18n-experience.md 的更新
      └── PR review 时，团队确认经验描述是否准确
          └── 合入 main 后，所有人的 AI 自动获得这条经验
```

---

## 八、完整命令速查

| 命令 | 作用 | 基础版 | Worktree 版 |
|------|------|--------|-------------|
| `/req-new` | 创建需求 | ✅ 创建分支 | ✅ 创建分支 + 可选 worktree |
| `/req-continue` | 恢复需求 | ✅ | ✅ |
| `/req-save` | 保存进度 | ✅ | ✅ |
| `/req-done` | 完成需求 | ✅ 沉淀经验 + 合并提示 | ✅ 沉淀经验 + 合并清理 + worktree 移除 |
| `/req-sync` | 同步知识 | — | ✅ merge main 到当前 worktree |
| `/code-review` | 代码审查 | ✅ | ✅ |
| `/new-page` | 创建页面 | ✅ | ✅ |
| `/i18n-check` | 多语言检查 | ✅ | ✅ |
| `/note` | 记录经验 | ✅ | ✅ |

---

## 九、从基础版升级的步骤

如果你已经按基础版搭建完成，升级到 worktree 版只需：

1. **.gitignore** 添加 `.worktrees/`
2. **替换** `.claude/commands/req-new.md` 为升级版（支持 worktree 选项）
3. **新增** `.claude/commands/req-done.md`
4. **新增** `.claude/commands/req-sync.md`
5. **替换** `.claude/settings.json` 中的 hooks（SessionStart 增加分支/worktree 信息）
6. **AGENTS.md** 追加以下内容：

```markdown
## 项目知识库 - Worktree 并行版补充
- /req-done — 完成需求（沉淀经验 + 合并清理提示）
- /req-sync — 同步 main 最新知识到当前 worktree
- 并行开发时，experience 文件为 append-only 格式，每条用 --- 分隔
- 单个 experience 文件超过 50 条时归档到 experience/archive/
```

总共改 4 个文件，新增 2 个文件，10 分钟搞定。

---

## 十、常见问题与错误恢复

### Worktree 创建失败

**分支已存在**：
```bash
# 报错：fatal: a branch named 'feature/xxx' already exists
# 说明之前创建过同名分支但没清理
git branch -d feature/xxx          # 删除旧分支（已合并的）
git branch -D feature/xxx          # 强制删除（未合并的，确认不需要后）
# 然后重新 /req-new
```

**worktree 目录已存在**：
```bash
# 报错：fatal: '.worktrees/xxx' already exists
# 说明之前的 worktree 没清理干净
git worktree remove .worktrees/xxx --force
# 然后重新 /req-new
```

### /req-continue 找不到存档

可能原因：
- requirements/ 目录在另一个分支上（切分支后看不到）
- 需求 ID 拼写错误

解决：
```bash
# 查看所有分支上的 requirements
git log --all --name-only -- 'requirements/*/meta.yaml'
# 找到后切到对应分支
git checkout feature/xxx
```

### 知识同步冲突

```bash
# /req-sync 报冲突
# 1. 查看冲突文件
git status

# 2. experience 文件冲突：保留双方内容
git checkout --theirs context/project/cookpal/experience/xxx.md  # 先取 main 版本
# 然后手动把自己的新增条目追加到文件末尾

# 3. 其他文件冲突：按需处理
git mergetool  # 或手动编辑

# 4. 完成合并
git add . && git commit --no-edit
```

### Worktree 中 .claude/ 配置不生效

worktree 创建时继承的是 main 分支当时的文件。如果之后 main 上更新了 .claude/ 配置：
```bash
# 在 worktree 中同步最新配置
git checkout main -- .claude/
```

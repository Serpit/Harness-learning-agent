# /req-fix 方案：Fix 提交与行为契约同步

> 基于 [agentic-engineering-basic.md](./agentic-engineering-basic.md) 和 [agentic-engineering-openspec-upgrade.md](./agentic-engineering-openspec-upgrade.md)
> 解决问题：需求 done 后的 fix 提交如何与 behaviors/ 保持一致

---

## 一、问题背景

升级版方案引入了 `behaviors/` 作为系统行为契约的中心化文档。流程是：

```
需求开发 → acceptance.md → /req-done → 合并到 behaviors/ → 提测 → done
```

但 done 之后会出现 fix（QA 打回、线上问题、用户反馈）。fix 可能改变系统实际行为，导致 behaviors/ 过时。

---

## 二、Fix 的两种类型

| 类型 | 例子 | 改 behaviors？ |
|------|------|----------------|
| 实现 bug | 占位符没替换、空指针、样式错位 | 否，Scenario 本来就要求正确行为 |
| 行为变更 | 边界遗漏需补 Scenario、需求理解偏差需改 Requirement | 是 |

**判断标准**：fix 是在修"没做到"（实现 bug）还是"做错了/少了"（行为变更）。

---

## 三、/req-fix 命令设计

用 AI 语义级判断替代文件路径匹配。fix 完成后，AI 直接对照 behaviors/ 判断是否需要更新。

### .claude/commands/req-fix.md

```markdown
修复 bug 并检查行为契约一致性。

<context>
以下是给你的约束，不要写进任何文件：
- 项目背景见 context/project/cookpal/INDEX.md
- fix 分两类：实现 bug（不改行为）和行为变更（需更新契约）
- 判断标准：这个 fix 是在修"没做到"还是"做错了/少了"
- behaviors/ 中的 Requirement + Scenario 是行为契约，必须与系统实际行为一致
</context>

<steps>
1. 和用户确认 bug 现象，分析根因
2. 完成修复
3. 读取 behaviors/ 下可能相关的文件
4. 对照本次改动，判断 fix 类型：
   - **实现 bug**（behaviors/ 已正确描述预期行为，只是代码没做到）
     → 不更新 behaviors/
     → 提交 fix: {描述}
   - **行为变更**（发现 behaviors/ 缺少 Scenario 或 Requirement 描述不准确）
     → 先更新 requirements/{id}/acceptance.md
     → 再同步更新 behaviors/{feature}.md
     → 提交 fix: {描述} + 更新契约
5. 判断是否有通用经验价值，有则记录到 experience/
</steps>

<output_format>
修复完成后输出：
| 项目 | 内容 |
|------|------|
| Bug 现象 | {现象} |
| 根因 | {原因} |
| Fix 类型 | 实现 bug / 行为变更 |
| 改动文件 | {文件列表} |
| behaviors 更新 | 无 / {具体更新内容} |
| 经验沉淀 | 无 / 已记录到 {文件} |
</output_format>

Bug 描述: $ARGUMENTS
```

---

## 四、与现有流程的衔接

### 正常流程（不变）

```
/req-new → 编码 → /req-verify → /req-done（合并 behaviors）→ 提测 → done
```

### Fix 流程（新增）

```
用户描述 bug → /req-fix → 修复 + 判断类型 → 提交
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
           实现 bug                        行为变更
           直接提交                    更新 acceptance
           记 experience（可选）       更新 behaviors
                                      记 experience（可选）
```

### 关键设计决策

1. **done 不回退**：fix 不改变需求的 done 状态，需求确实完成了，fix 是独立事件
2. **不新建 requirement**：fix 不值得走完整的 req-new 流程，/req-fix 是轻量版
3. **behaviors 更新是原子的**：acceptance.md 和 behaviors/ 在同一次提交中同步更新

---

## 五、改造前后对比

| 场景 | 改造前 | 改造后 |
|------|--------|--------|
| 实现 bug fix | 修代码，提交，behaviors 不受影响 | 同左（/req-fix 确认后跳过 behaviors 更新） |
| 行为变更 fix | 修代码，提交，behaviors 悄悄过时 | /req-fix 自动检测并同步 behaviors |
| fix 经验沉淀 | 依赖开发者主动 /note | /req-fix 流程内置判断和提醒 |

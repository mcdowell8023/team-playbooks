# Dogfooding 报告：team-playbooks v1.0.0 自用 github-release.md 发布

## 时间
2026-04-26

## 用户
鲁班（按万三派单执行）

## 流程
按 github-release.md Step 0-9 走（万三派单已预整理步骤，与 Playbook 对照执行）

## 反馈分类

### ✅ 流畅项
1. **Step 2-3 gh 鉴权**：`gh auth status` 一条命令确认状态，已登录直接跳过，非常顺畅
2. **Step 5 gh repo create**：`--source=. --remote=origin --push` 一条命令搞定创建+推送，命令完整可直接 copy
3. **Step 7 gh release create**：`--notes-file` 参数清晰，release notes 单独文件管理是好实践
4. **Step 8 topics 设置**：`--add-topic` 多个串联，命令可直接复制
5. **Step 9 验证清单**：验证命令全面，覆盖 remote/heads/tags/release/topics，一跑就知道全不全

### ⚠️ 有歧义项
1. **git config user.email**：Playbook 假设全局 gitconfig 已设，但新机器/新目录可能没有。本次实际遇到 `git commit` 因无 email 失败，需要先 `git config user.email`
2. **安全扫描误报**：Playbook 教用户 `grep "ghp_"` 扫敏感信息，但 Playbook 自身就包含这些关键字作为教程示例，导致大量误报。需要决定：教程类仓库的安全扫描怎么处理？
3. **Step 0 没有明确的「source 不是 git 仓库」场景**：Playbook 假设你已经有一个 git 仓库要推，但本次需要先 mkdir + cp + git init，这一步完全不在 Playbook 覆盖范围

### ❌ 缺漏项
1. **「从零建仓」场景缺失**：github-release.md 主要覆盖「已有 git 仓库 → 推远端」，但「散落文件 → 初始化 git → 推远端」这个场景没有独立路径
2. **RELEASE-NOTES 文件的 git 管理**：release notes 文件创建后需要 commit + push，但 Playbook Step 7 只说了 `gh release create`，没提 release notes 文件本身要入库
3. **默认分支名**：git init 默认是 master，但 GitHub 新仓库默认是 main。本次用 `--source=.` 自动推了 master，没问题，但如果用户手动推可能会遇到分支名不匹配

### 🔧 优化建议
1. **增加 Step -1「Source 准备」**：在 Step 0 安全扫描之前，加一个判断：source 是否已经是 git 仓库？不是的话，给出 `git init` + `git config` + `git add` + `git commit` 模板
2. **安全扫描加白名单机制**：对于教程/文档类仓库，grep 规则可以加 `--exclude` 或用 `.gitattributes` 标记哪些文件是示例
3. **git config 前置检查**：在 Step 4 git init 后加一步 `git config user.email 2>/dev/null || echo "⚠️ 需要设置 git user.email"`
4. **Release notes 入库提醒**：Step 7 末尾加一行 `git add RELEASE-NOTES-*.md && git commit -m "docs: release notes" && git push`

## 改进 PR 建议

针对 github-release.md 的具体修改：

1. **新增 Step -1（约第 15-40 行之间插入）**：
   ```markdown
   ## Step -1: Source 准备（如果还不是 git 仓库）
   如果你的文件还不在 git 仓库里：
   mkdir -p /path/to/project && cd /path/to/project
   cp /source/files/* .
   git init
   git config user.name "yourname"
   git config user.email "you@example.com"
   git add . && git commit -m "feat: initial commit"
   ```

2. **Step 0 安全扫描（约第 200 行）加注释**：
   ```markdown
   > 💡 如果仓库本身是教程/文档类（包含命令示例），grep 会误报。
   > 确认是示例文本而非真实凭证即可跳过。
   ```

3. **Step 7 末尾（约第 500 行后）加入库步骤**：
   ```markdown
   ### 别忘了把 release notes 入库
   git add RELEASE-NOTES-*.md && git commit -m "docs: add release notes" && git push
   ```

## 总体评价

github-release.md 作为「已有 git 项目推 GitHub」的 Playbook **质量很高**，命令可直接复制执行，验证清单完整。主要短板在「从零建仓」和「教程类仓库安全扫描」两个边界场景。建议下一版补上 Step -1 和安全扫描白名单即可。

**可用性评分：8/10**（扣分项：从零建仓缺失 -1，安全扫描误报 -0.5，release notes 入库遗漏 -0.5）

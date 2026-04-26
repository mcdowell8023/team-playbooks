# Playbook: 发版前就绪审查（Release Readiness Review）

> **类型：** How-to Guide / 发版总闸执行手册
> **最后更新：** 2026-04-26
> **维护人：** 鲁班（起草）+ 万三（审批）

---

## 1. 适用场景

- 所有正式发版前（GitHub Release / npm publish / Skill 发布）
- 从 prerelease（alpha/beta/rc）到 stable 的晋级
- 任何面向外部用户可见的版本变更
- **必须在 `github-release.md` / `npm-publish.md` / `skill-publish.md` 之前执行**

## 2. 不适用场景

- 内部草稿版本、PoC、实验性分支
- 纯文档更新（不改代码、不改版本号）
- 日常 `git push` 不涉及 tag / release
- 开发期间的中间 commit 推送

## 3. 核心理念

> **单轨审查不够。代码面看起来没问题，不代表用户拿到手能跑。**
>
> 2026-04-26 实证：包拯（代码审查）判定 alpha.1 为 0 Blocker 可发，商鞅（端到端实跑）立刻跑出 3 个 Blocker（B4/B5/B6）。**如果只有一轨，这三个 Blocker 会直接到用户手上。**

本 Playbook 定义 **八大检查门（Gate）** + **三轨审查机制**，确保发版条件成熟后才进入实际发版流程。

---

## 4. 前置条件

### 4.1 凭证

| 凭证 | 用途 | 存放 | 检查方式 |
|------|------|------|---------|
| GitHub PAT | gh CLI / API | `~/code_key` | `gh auth status` |
| npm token | npm publish | `~/.npmrc` | `npm whoami` |
| Skill 发布凭证 | Claw Hub | 按项目 | `openclaw skills check` |

**硬规则：凭证先查记忆（`memory_search`），不要让用户重复登录。**

### 4.2 工具

- `gh` CLI（`~/bin/gh`）
- `npm`（Node.js 内置）
- `git`（系统）
- `grep` / `rg`（关键字扫描）

### 4.3 已知路径

| 资源 | 路径 |
|------|------|
| Playbook 模板 | `~/open-claw-output/doc/playbooks/playbook-template.md` |
| GitHub 发布手册 | `~/open-claw-output/doc/playbooks/github-release.md` |
| 产出根目录 | `~/open-claw-output/` |

---

## 5. 角色分工

### 万三（主会话 — 总控 + 架构视角）

- 发版决策拍板（版本号、prerelease 标记、发布范围）
- 架构变更影响评估（兼容性、升级路径）
- Gate 7 最终签字
- **凭证类问题先查记忆**

### 包拯（代码视角审查）

- grep 关键字清查 + 测试实跑 + 元信息核验
- 输出：验证矩阵 + Blocker/Major/Suggestion 分级
- 推荐模型：`gpt-5.4`（元信息交叉核验最强）
- 工时上限：10 分钟
- **必须返回 grep 证据，不接受纯声明**

### 商鞅（用户视角审查）

- mktemp 沙箱 + 端到端安装/运行 + 文档对照
- 模拟真实用户从零开始的体验
- 输出：验证矩阵 + Blocker/Major/Suggestion 分级
- 推荐模型：`gpt-5.4`
- 工时上限：10 分钟
- **沙箱必须 `mktemp -d` + `trap` 清理，禁止污染本机**

### 图灵（架构视角审查 — 可选）

- 仅在架构变更较大时介入
- 评估：breaking change / 依赖兼容 / 数据迁移 / 升级路径
- 推荐模型：`claude-opus-4.6`

### 鲁班 / 移公（修复执行）

- 审查发现 Blocker → 鲁班修代码 → 重新提审
- 移公处理环境 / 工具 / 凭证问题

---

## 6. 八大检查门（Pre-Release Gates）

### Gate 1: 代码冻结 🧊

**目标：** 确认发版分支代码状态确定，不会再有意外变更。

**检查项：**

```bash
# 1. 工作区干净
git status --short
# 预期: 空输出

# 2. 当前分支正确
git branch --show-current
# 预期: main（或约定的发版分支）

# 3. 无未推 commits
git log origin/main..HEAD --oneline
# 预期: 空（ahead 0）

# 4. 无未拉 commits
git log HEAD..origin/main --oneline
# 预期: 空（behind 0）
```

**异常处理：**
- 工作区不干净 → 补 commit 或 stash，不要带脏发版
- 分支不对 → `git checkout main` + `git merge <dev-branch>`
- ahead > 0 → `git push origin main`
- behind > 0 → `git pull --rebase origin main`

---

### Gate 2: 测试基线 🧪

**目标：** 所有自动化测试全绿，覆盖率达标，无已知 flaky test。

**检查项：**

```bash
# 1. 单元测试
npm test
# 预期: All tests passed（exit 0）

# 2. 覆盖率（如配置了）
npm run test:coverage
# 预期: ≥ 项目红线（通常 80%）

# 3. E2E 测试（如有）
npm run test:e2e
# 预期: All passed

# 4. Flaky test 检查
# 连跑 3 次，结果一致
for i in 1 2 3; do npm test 2>&1 | tail -1; done
```

**异常处理：**
- 测试失败 → **停止发版**，先修代码
- 覆盖率不达标 → 补测试或调整红线（需万三拍板）
- Flaky test → 隔离 + `test.skip` 标记 + 记录到已知问题

**典型耗时：** 2-5 分钟（Skill 类项目 ~400 tests ≈ 30s）

---

### Gate 3: 构建产物验证 📦

**目标：** 构建产物干净、完整、体积合理。

**检查项：**

```bash
# 1. 干净重建
rm -rf dist
npm run build
# 预期: exit 0，无 warning

# 2. 产物内容检查
ls -la dist/
# 预期: 含 types（.d.ts）+ JS（.js/.mjs/.cjs）+ 资源

# 3. 包体积
du -sh dist/
# 对比上版：不应有异常膨胀（>2x 需排查）

# 4. 产物未入库（关键！alpha.1 翻车点）
git ls-files | grep -E "^dist/" | head
# 预期: 空输出（dist/ 不应被 git 跟踪）

# 5. node_modules 未入库
git ls-files | grep -E "^node_modules/" | head
# 预期: 空输出
```

**异常处理：**
- `dist/` 或 `node_modules/` 被跟踪 → `git rm -r --cached dist/ node_modules/` + 修 `.gitignore` + 重新 commit
- 构建失败 → **停止发版**，先修构建
- 体积异常 → 检查是否误包含了测试文件、源码、大资源

**历史教训：** 2026-04-25 alpha.1，`dist/` + `node_modules/` 入库，导致仓库膨胀 + 安装体验差。

---

### Gate 4: 文档同步 📄

**目标：** 用户可见文档与代码版本一致，无死链。

**检查项：**

```bash
# 1. README 版本/徽章
grep -n "version\|badge\|shield" README.md

# 2. CHANGELOG 最新条目匹配当前版本
head -20 CHANGELOG.md
# 预期: 顶部条目 = 即将发布的版本号

# 3. QUICKSTART / 安装指引路径与代码一致
grep -rn "require\|import\|install" README.md QUICKSTART.md 2>/dev/null

# 4. 死链扫描
grep -oE '\[.*?\]\(.*?\)' README.md | grep -oE '\(.*?\)' | tr -d '()' | while read url; do
  if [[ "$url" == http* ]]; then
    status=$(curl -sI -o /dev/null -w "%{http_code}" "$url" 2>/dev/null || echo "ERR")
    [ "$status" != "200" ] && echo "❌ $url → $status"
  elif [[ "$url" != \#* ]]; then
    [ ! -f "$url" ] && echo "❌ $url → 文件不存在"
  fi
done
# 预期: 无输出（全部链接有效）

# 5. 文件树引用（如 README 中列出项目结构）
# 对照实际文件树，确认无残留/缺失
```

**异常处理：**
- 死链 → 修复或移除引用
- 版本不一致 → 更新文档
- 文件树残留 → 删除过时引用（alpha.1 包拯发现 `cli-reference.md` 死链）

---

### Gate 5: 配置与依赖 ⚙️

**目标：** package.json 及项目配置正确、完整。

**检查项：**

```bash
# 1. 版本号正确
node -e "console.log(require('./package.json').version)"
# 预期: 与即将发布的版本号一致

# 2. engines / peerDeps 合理
node -e "const p=require('./package.json'); console.log('engines:', p.engines); console.log('peerDeps:', p.peerDependencies)"

# 3. .gitignore 覆盖
cat .gitignore
# 必须包含: node_modules/ dist/ .env *.local .cache

# 4. LICENSE 存在（Public 仓库必须）
ls LICENSE
# 预期: 存在

# 5. main / module / types / exports 字段
node -e "const p=require('./package.json'); console.log('main:', p.main, 'module:', p.module, 'types:', p.types, 'exports:', JSON.stringify(p.exports))"
# 预期: 指向 dist/ 下的正确文件

# 6. files 字段（npm publish 范围）
node -e "console.log(require('./package.json').files)"
# 预期: 只包含需要发布的文件/目录
```

**异常处理：**
- 版本号不对 → `npm version <new-version> --no-git-tag-version` 修正
- `.gitignore` 遗漏 → 补齐 + `git rm -r --cached` 清理已跟踪的文件
- LICENSE 缺失 → 补 MIT/Apache-2.0（需万三确认）
- exports 字段错误 → 修正后重新 build 验证

---

### Gate 6: 凭证与权限 🔑

**目标：** 发版所需的所有凭证已就位、未过期、权限充足。

> **今日教训（2026-04-26）：** 鲁班发布 alpha.2 时卡在 gh auth 未登录，实际 `~/code_key` 早有 PAT。**先查记忆，不要让用户重新做已做过的事。**

**检查项：**

```bash
# 1. 先查记忆！
# memory_search "GitHub PAT token"
# memory_search "npm token"

# 2. GitHub 凭证
gh auth status
# 预期: ✓ Logged in to github.com account mcdowell8023

# 3. GitHub token scope 足够
curl -sI -H "Authorization: token $(cat ~/code_key)" https://api.github.com | grep x-oauth-scopes
# 预期: 含 repo, workflow, delete_repo

# 4. npm 凭证（如需 npm publish）
npm whoami
# 预期: 你的 npm 用户名

# 5. 网络连通
curl -sI https://api.github.com | head -1
# 预期: HTTP/2 200
```

**异常处理：**
- gh 未登录 → `cat ~/code_key | gh auth login --with-token`
- Token 过期 → 见 `github-release.md` Step 3 方案 E
- npm 未登录 → `npm login`（交互式）或配 `.npmrc`
- 网络不通 → 检查代理配置

---

### Gate 7: 三轨审查通过 ✅

**目标：** 代码视角 + 用户视角 + 架构视角三轨独立审查，全部 0 Blocker。

**这是本 Playbook 的核心章节，详见 §7 三轨审查机制。**

**检查项：**
- [ ] 包拯审查报告：0 Blocker
- [ ] 商鞅端到端报告：0 Blocker
- [ ] 万三架构评估签字（或图灵审查报告：0 Blocker）
- [ ] 所有 Major 已评估（可接受则记录为已知问题）

---

### Gate 8: 应急预案 🚨

**目标：** 发版失败或发版后发现严重问题时，有明确的回滚方案。

**检查项：**

```bash
# 1. 回滚命令已准备（按发版类型选）

# GitHub Release 回滚
echo "gh release delete vX.Y.Z --yes"
echo "git push origin :refs/tags/vX.Y.Z"
echo "git tag -d vX.Y.Z"

# npm unpublish（72 小时内）
echo "npm unpublish <package>@X.Y.Z"

# 2. 监控/告警（如有线上服务）
# 确认 healthcheck endpoint 可用

# 3. 用户通知渠道
# CHANGELOG 公开 / GitHub Discussions / 飞书群
```

**异常处理：**
- npm unpublish 超 72 小时 → 只能 deprecate
- GitHub Release 已被用户下载 → 发补丁版本 + 公告

---

## 7. 三轨审查机制（核心章节）

### 7.1 为什么需要三轨

| 视角 | 能发现 | 不能发现 |
|------|--------|---------|
| **代码（包拯）** | 语法错误、类型问题、元信息不一致、死链、测试覆盖 | 安装流程断裂、用户体验卡点、目录语义混乱 |
| **用户（商鞅）** | 安装失败、运行报错、文档与实际不符、CLI 选项不一致 | 深层代码缺陷、类型问题、依赖兼容 |
| **架构（图灵/万三）** | Breaking change、升级路径缺失、依赖冲突、性能退化 | 表面细节问题 |

**2026-04-26 实证：**
- 包拯审 alpha.1 → 0 Blocker → 判定可发
- 商鞅端到端跑 alpha.1 → **3 Blocker**：
  - B4: `uninstall` 选项 CLI 声明 vs 实际行为不一致
  - B5: `uninstall` 不完整，残留文件
  - B6: 目录语义混乱（`learn` vs `learning-loop`）
- **结论：单轨审查会放过 50% 以上的用户可感知 Blocker**

### 7.2 三轨报告模板

每轨审查必须输出以下格式（可直接作为子代理 prompt）：

```markdown
# [角色名] 发版前审查报告

## 审查对象
- 项目：<项目名>
- 版本：<版本号>
- 分支/commit：<branch> @ <sha>

## 验证矩阵

| # | 维度 | 状态 | 证据 |
|---|------|------|------|
| 1 | 代码 build | ✅/❌ | `npm run build` exit 0 |
| 2 | 单元测试 | ✅/❌ | 416/416 passed |
| 3 | E2E 测试 | ✅/❌ | 25/25 passed |
| 4 | 元信息一致 | ✅/❌ | package.json version = tag = CHANGELOG |
| 5 | 死链扫描 | ✅/❌ | grep 结果附后 |
| 6 | 安全扫描 | ✅/❌ | 无 ghp_ / sk- / 个人路径 |
| 7 | .gitignore | ✅/❌ | 无 dist/node_modules 入库 |
| 8 | 安装流程 | ✅/❌ | mktemp 沙箱实跑通过 |

## 发现问题

### Blocker（阻塞发版）
- **B1:** [描述] — 证据：[grep 输出 / 错误日志]
- **B2:** ...

### Major（建议修复，不阻塞）
- **M1:** [描述]

### Suggestion（改善建议）
- **S1:** [描述]

## 是否阻塞发版: YES / NO

## grep 证据附录
（关键扫描命令及输出）
```

### 7.3 审查并行策略

```
万三派发审查任务
├─ 包拯（代码视角）──┐
│                     ├─ 并行执行（不浪费时间）
├─ 商鞅（用户视角）──┘
│
├─ 两份报告返回
│   ├─ 双绿灯（0 Blocker + 0 Blocker）
│   │   └─ 万三做架构评估签字 → Gate 7 通过 ✅
│   │
│   ├─ 一方绿灯 + 一方有 Blocker
│   │   └─ 修复 Blocker → 仅重审有 Blocker 的那一轨
│   │       （绿灯方不需要重审，除非修复涉及其审查范围）
│   │
│   └─ 双方都有 Blocker
│       └─ 修复全部 → 双轨重审
│
└─ 图灵（架构视角）—— 仅在以下情况介入：
    - 有 breaking change
    - 依赖大版本升级
    - 数据模型变更
    - 万三判断需要
```

### 7.4 审查决策树

```
收到审查报告
├─ 0 Blocker + 0 Major → 直接通过
├─ 0 Blocker + N Major → 评估 Major：
│   ├─ 可接受 → 记录为已知问题，通过
│   └─ 应修复 → 修复后仅重审 Major 涉及的维度
├─ N Blocker → 必须修复
│   ├─ 修复后重审
│   │   ├─ 通过 → Gate 7 ✅
│   │   └─ 又发现新 Blocker → 再修再审
│   │       └─ 连续两轮同一角色过报（声称修好实际没修）
│   │           → **换角色**（gpt-5.4 ↔ opus-4.6）
│   └─ 修复代价过大 + 时间紧迫
│       → 标 prerelease + 列已知问题 + 显超签字
└─ 审查超时（>10 分钟）→ 检查模型状态 → 必要时换模型重派
```

### 7.5 包拯审查 Prompt 模板

```
你是包拯（reviewer）。对 [项目名] v[版本号] 做发版前代码视角审查。

## 审查范围
项目路径：[路径]
分支：main @ [commit sha]

## 必做检查
1. `npm run build` 干净构建
2. `npm test` 全绿
3. `git ls-files | grep -E "(dist/|node_modules/)"` → 空
4. `grep -rn "ghp_\|sk-\|/home/mcdowell" --include="*.{md,ts,js,json}" .` → 空
5. package.json version = 即将打的 tag
6. README / CHANGELOG 版本号一致
7. 死链扫描（README 中的所有链接）
8. .gitignore 覆盖完整

## 输出格式
按验证矩阵模板输出（见 release-readiness-review.md §7.2）

## 硬规则
- 所有「已通过」必须附 grep/命令输出证据
- 0 Blocker 不等于不用写报告
- 信用比速度重要，不确定就标 Major，别过报
```

### 7.6 商鞅端到端 Prompt 模板

```
你是商鞅（autotest）。对 [项目名] v[版本号] 做发版前用户视角端到端测试。

## 测试环境
- 必须使用 `mktemp -d` 创建隔离沙箱
- 必须 `trap "rm -rf $SANDBOX" EXIT` 清理
- 禁止在项目源码目录直接操作

## 必做测试
1. 全新安装：从 README 指引出发，模拟用户首次安装
2. 基本功能：每个 CLI 命令 / API 至少跑一次
3. 卸载流程：`--uninstall` 或卸载指引，确认清理干净
4. 文档对照：README 每条指令实际执行，输出与文档描述一致
5. 边界情况：空输入、错误参数、重复安装

## 输出格式
按验证矩阵模板输出（见 release-readiness-review.md §7.2）

## 硬规则
- 沙箱必须隔离，`trap` 清理
- 发现问题必须附完整错误日志
- 「能跑」≠「对」—— 对照文档逐条验证
```

---

## 8. 决策树：是否可以发版？

```
所有 Gate 1-8 全绿？
│
├─ 是 → ✅ 可以发版
│   └─ 进入 github-release.md / npm-publish.md / skill-publish.md
│
└─ 否 → 哪些 Gate 红？
    │
    ├─ Gate 1（代码冻结）红
    │   └─ 补 commit / push / merge → 重检 Gate 1
    │
    ├─ Gate 2（测试）红
    │   └─ 修代码 / 修测试 → 重跑 → 重检 Gate 2
    │
    ├─ Gate 3（构建产物）红
    │   └─ 修 build / 修 .gitignore → rm -rf dist → 重建 → 重检 Gate 3
    │
    ├─ Gate 4（文档）红
    │   └─ 修文档 → 重检 Gate 4
    │       └─ 死链太多？→ 派纪昀批量修
    │
    ├─ Gate 5（配置）红
    │   └─ 修 package.json / .gitignore / LICENSE → 重检 Gate 5
    │
    ├─ Gate 6（凭证）红
    │   └─ 先查记忆！→ 复用 / 续期 / 重新生成 → 重检 Gate 6
    │
    ├─ Gate 7（三轨审查）红
    │   └─ 有 Blocker → 修复 → 重审（见 §7.4 审查决策树）
    │
    ├─ Gate 8（应急预案）红
    │   └─ 准备回滚命令 → 确认监控 → 重检 Gate 8
    │
    └─ 时间紧迫但必须发版？
        └─ 列「已知问题」清单
            + 标 prerelease（alpha/beta/rc）
            + 显超亲自签字确认
            + Release Notes 明确标注风险
```

---

## 9. 时间盒（Time-box）

### 9.1 典型耗时基线（Skill 类项目，~400 tests，~4000 行）

基于 2026-04-26 v1.1.0-alpha.2 实战数据：

| 阶段 | 耗时 | 说明 |
|------|------|------|
| Gate 1-6 自检 | ~5 分钟 | 万三/鲁班执行，大部分是一行命令 |
| Gate 7: 包拯审查 | ~3 分钟 | 代码视角，grep + 测试实跑 |
| Gate 7: 商鞅端到端 | ~5 分钟 | 用户视角，mktemp 沙箱实跑 |
| Gate 7: 万三架构签字 | ~2 分钟 | 基于两份报告快速评估 |
| Gate 8 应急预案 | ~1 分钟 | 复制回滚命令模板 |
| **审查总时间（无修复）** | **~15-20 分钟** | |
| 单轮修复（1-3 Blocker） | ~10 分钟 | 鲁班修 + 重新 commit |
| 修复后重审 | ~5 分钟 | 仅重审有 Blocker 的轨 |
| **含一轮修复总时间** | **~30-35 分钟** | |

### 9.2 时间盒硬限

| 阶段 | 上限 | 超时动作 |
|------|------|---------|
| 单轨审查 | 10 分钟 | 检查模型状态 → 换模型重派 |
| 单轮修复 | 15 分钟 | 拆分修复项 → 并行或降级 |
| 整体审查流程 | 45 分钟 | 万三介入评估是否先发 prerelease |
| 修复轮数 | 3 轮 | 第 3 轮仍有 Blocker → 暂停发版，全面排查 |

---

## 10. 卡点 & 兜底

| # | 卡点 | 第一兜底 | 第二兜底 |
|---|------|---------|---------|
| 1 | 包拯连续过报（声称修好实际没修） | 换模型（gpt-5.4 ↔ opus-4.6） | 换角色（派图灵做代码审查） |
| 2 | 商鞅 mktemp 污染本机 | `mktemp -d` + `trap "rm -rf $dir" EXIT` | 用 Docker 容器隔离 |
| 3 | 文档死链太多（>5 条） | 派纪昀批量修复 | 移除死链章节，发后补 |
| 4 | 测试 flaky（间歇性失败） | 隔离 + `test.skip` + 连跑 3 次 | 标记为已知问题 + 记录 |
| 5 | 凭证过期 | 先查记忆 → `~/code_key` → 续 token | 让用户手动重新生成（最后手段） |
| 6 | 构建产物入库（dist/node_modules） | `git rm -r --cached` + 修 `.gitignore` | 重新初始化仓库（极端情况） |
| 7 | 审查模型不可用 | 按模型降级链切换 | 万三手动审查（仅紧急情况） |
| 8 | 版本号冲突（tag 已存在） | 删远端 tag → 重推 | 升 patch 版本号 |
| 9 | 包拯/商鞅审查结论矛盾 | 以更严格的结论为准 | 万三仲裁（附证据） |

---

## 11. 验证清单（Pre-Release Checklist）

> **可直接 copy 给子代理作为自检 prompt。**

```markdown
## Pre-Release Checklist — [项目名] v[版本号]

### Gate 1: 代码冻结
- [ ] `git status --short` → 空
- [ ] 当前分支 = `main`
- [ ] `git log origin/main..HEAD` → 空（ahead 0）
- [ ] `git log HEAD..origin/main` → 空（behind 0）

### Gate 2: 测试基线
- [ ] `npm test` → 全绿（exit 0）
- [ ] 测试数量 = [预期数]（如 416/416）
- [ ] 覆盖率 ≥ [红线]%
- [ ] 无 flaky test

### Gate 3: 构建产物
- [ ] `rm -rf dist && npm run build` → exit 0
- [ ] `ls dist/` 含预期文件（types + JS）
- [ ] `git ls-files | grep "^dist/"` → 空
- [ ] `git ls-files | grep "^node_modules/"` → 空
- [ ] 包体积合理

### Gate 4: 文档同步
- [ ] README 版本号/徽章 正确
- [ ] CHANGELOG 顶部条目 = 当前版本
- [ ] 无死链（grep + curl 扫描通过）
- [ ] 文件树引用与实际一致

### Gate 5: 配置与依赖
- [ ] `package.json` version = [版本号]
- [ ] engines / peerDeps 合理
- [ ] `.gitignore` 含 node_modules/ dist/ .env
- [ ] LICENSE 存在
- [ ] main/module/types/exports 指向正确

### Gate 6: 凭证与权限
- [ ] `gh auth status` → ✓ Logged in
- [ ] Token scopes 含 repo + workflow
- [ ] 网络连通（`curl -sI https://api.github.com`）
- [ ] npm token 有效（如需 npm publish）

### Gate 7: 三轨审查
- [ ] 包拯审查：0 Blocker
- [ ] 商鞅端到端：0 Blocker
- [ ] 万三架构签字
- [ ] 所有 Major 已评估

### Gate 8: 应急预案
- [ ] 回滚命令已准备
- [ ] Release Notes 已写
- [ ] prerelease 标记正确（alpha/beta/rc 必须标）
```

---

## 12. 历史教训

> 同一个坑踩两次是流程问题。每次踩坑必须更新此表。

| 日期 | 事件 | 教训 | 影响 Gate |
|------|------|------|----------|
| **2026-04-26** | alpha.2 双轨审查：包拯判 0 Blocker，商鞅跑出 B4/B5/B6 三个 Blocker | **单轨审查不够，必须双轨并行** | Gate 7 |
| **2026-04-26** | 鲁班发版卡在 gh auth 未登录，实际 `~/code_key` 早有 PAT | **凭证先查记忆，不要让用户重做** | Gate 6 |
| **2026-04-25** | alpha.1 `dist/` + `node_modules/` 入库，仓库膨胀 | **发版前必须检查 `.gitignore` 覆盖 + git ls-files** | Gate 3, Gate 5 |
| **2026-04-19** | v5.0.x 连续两轮 CHANGELOG 过报（图灵声称全闭合但正文残留） | **CHANGELOG 必须可定位到正文；连续过报换角色** | Gate 7 |
| **2026-03-19** | 第一次推 GitHub：私有/oc-前缀/main 分支不规范 | **发版前先查历史规范，命名需万三拍板** | Gate 4, Gate 5 |
| **2026-03-19** | 敏感信息差点入库（PAT / 个人路径） | **发版前必须跑安全扫描** | Gate 3 |

---

## 13. 与其他 Playbook 关系

```
                    ┌─────────────────────────────┐
                    │  release-readiness-review.md │  ← 你在这里（总闸）
                    │     （本 Playbook）           │
                    └──────────┬──────────────────┘
                               │ 全部 Gate 通过后
                    ┌──────────┴──────────────────┐
                    │                              │
            ┌───────▼────────┐          ┌─────────▼──────────┐
            │ github-release │          │   npm-publish.md   │
            │     .md        │          │      （待）         │
            └───────┬────────┘          └─────────┬──────────┘
                    │                              │
                    └──────────┬───────────────────┘
                               │
                    ┌──────────▼──────────────────┐
                    │   skill-publish.md （待）    │
                    │   Phase 3 = 双轨审查展开版   │
                    └─────────────────────────────┘
```

| Playbook | 关系 | 说明 |
|----------|------|------|
| `github-release.md` | **后续** | 本 Playbook Gate 全通过后，按 github-release.md 执行实际发版 |
| `npm-publish.md`（待） | **后续** | npm 包发布，同样需先过本闸 |
| `skill-publish.md`（待） | **后续 + 协同** | Skill 发布 Phase 3 即本 Playbook §7 三轨审查的展开版 |
| `playbook-template.md` | **模板** | 本 Playbook 结构基于模板 |

---

## 14. 快速执行流程（TL;DR）

给急性子的万三一个 30 秒概览：

```
1. Gate 1-6: 跑命令，5 分钟搞定
   git status → npm test → npm run build → 检查文档 → 检查配置 → gh auth status

2. Gate 7: 并行派包拯 + 商鞅，8 分钟返回
   ├─ 双绿灯 → 万三签字 → Gate 7 ✅
   └─ 有 Blocker → 鲁班修 → 重审受影响的轨

3. Gate 8: 复制回滚命令模板，1 分钟

4. 全绿 → 进入 github-release.md / npm-publish.md
```

---

## 15. 维护说明

- 本 Playbook 只写稳定的审查流程，不写项目特定的命令参数
- 新项目首次发版时，用 §11 Checklist 做模板，填入项目特定参数
- 有新翻车案例时，优先更新「§10 卡点 & 兜底」「§11 验证清单」「§12 历史教训」
- Gate 数量变更需万三审批（不轻易加减）
- 三轨审查 Prompt 模板由包拯/商鞅使用反馈迭代

---

## 修订记录

| 版本 | 日期 | 作者 | 变更 |
|------|------|------|------|
| v1.0 | 2026-04-26 | 鲁班 | 初版，基于 v1.1.0-alpha.2 发版实战经验 |

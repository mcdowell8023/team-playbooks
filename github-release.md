# Playbook: GitHub Release

> **类型：** How-to Guide / 对外发布执行手册
> **最后更新：** 2026-04-26
> **维护人：** 范蠡（流程框架）+ 移公（技术深度）

---

## 1. 适用场景

- 第一次把本地项目发布到 GitHub，创建新仓库并完成首发
- 已有 GitHub 仓库，需要执行版本发布（push branch + push tag + create release）
- 需要同时处理 RELEASE NOTES、topics、pre-release 标记等对外发布元信息

## 2. 不适用场景

- 仅做普通 `git push`，不打 tag、不创建 release
- 私有仓库日常内部协作，不涉及正式 release 页面
- 只是修 README / 小改动同步，不需要版本化发布

## 3. 前置条件

> **为什么先检查前置条件：** 多数发布失败（尤其首次）根因都是环境没就绪——gh 没装、token 过期、git 没配。提前确认比中途排障省 10 倍时间。

### 3.1 凭证

- GitHub PAT 路径：`~/code_key`
- **硬规则：先查记忆，再决定是否让用户登录**
- 推荐检索：`memory_search "GitHub PAT token"`
- 兜底检查：确认 `~/code_key` 存在、权限合理、内容为 `ghp_*` PAT

```bash
# 快速确认凭证可用
ls -la ~/code_key
# 预期: -rw------- 1 mcdowell mcdowell 41 ... ~/code_key

head -c 4 ~/code_key
# 预期: ghp_（classic PAT 前缀）

# API 直接验证
curl -s -H "Authorization: token $(cat ~/code_key)" https://api.github.com/user | jq .login
# 预期: "mcdowell8023"
```

### 3.2 工具：gh CLI

优先使用 `~/bin/gh`（用户目录安装，不需要 sudo）。

```bash
# 检查是否已安装
which gh && gh --version
# 预期: ~/bin/gh → gh version 2.x.x
```

**未安装时——用户目录安装脚本（无需 sudo）：**

```bash
#!/usr/bin/env bash
set -euo pipefail

# === 配置 ===
GH_VERSION="${GH_VERSION:-2.91.0}"
INSTALL_DIR="${HOME}/bin"
TMP_DIR="/tmp/gh-install-$$"

# === 代理（可选，按需取消注释）===
# export HTTPS_PROXY="http://127.0.0.1:7890"
# export HTTP_PROXY="http://127.0.0.1:7890"

TARBALL="gh_${GH_VERSION}_linux_amd64.tar.gz"
URL="https://github.com/cli/cli/releases/download/v${GH_VERSION}/${TARBALL}"

mkdir -p "${INSTALL_DIR}" "${TMP_DIR}"
cd "${TMP_DIR}"

echo "下载 gh ${GH_VERSION}..."
wget -q --show-progress "${URL}" -O "${TARBALL}"

# SHA256 校验（推荐）
CHECKSUM_URL="https://github.com/cli/cli/releases/download/v${GH_VERSION}/gh_${GH_VERSION}_checksums.txt"
wget -q "${CHECKSUM_URL}" -O checksums.txt
grep "${TARBALL}" checksums.txt | sha256sum -c -
# 预期: gh_2.91.0_linux_amd64.tar.gz: OK

tar xzf "${TARBALL}"
cp "gh_${GH_VERSION}_linux_amd64/bin/gh" "${INSTALL_DIR}/"
chmod +x "${INSTALL_DIR}/gh"

# PATH（幂等写入）
grep -q 'export PATH="$HOME/bin:$PATH"' ~/.bashrc || \
  echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
export PATH="${INSTALL_DIR}:${PATH}"

# 清理
rm -rf "${TMP_DIR}"

# 验证
gh --version
echo "✅ gh 安装完成: $(which gh)"
```

**卸载：**

```bash
rm -f ~/bin/gh
sed -i '/export PATH="\$HOME\/bin:\$PATH"/d' ~/.bashrc
rm -rf ~/.config/gh
```

**代理环境兜底：**

```bash
export HTTPS_PROXY="http://proxy:port"
export HTTP_PROXY="http://proxy:port"
export NO_PROXY="localhost,127.0.0.1"
# gh CLI 遵循这些环境变量，无需额外配置
curl -sI https://api.github.com | head -1
# 预期: HTTP/2 200
```

### 3.3 git 配置

```bash
# 检查当前配置
git config --global user.name
# 预期: mcdowell8023（空 = 需要设置）

git config --global user.email
# 预期: 你的 GitHub 关联邮箱（空 = 需要设置）

# 设置（如果为空）
git config --global user.name "mcdowell8023"
git config --global user.email "your-email@example.com"

# 用 gh 作为 credential helper（推荐，自动管理）
gh auth setup-git
# 预期: ✓ Configured git to use GitHub CLI for authentication

# 查看 GitHub 关联邮箱
gh api /user/emails --jq '.[].email'
```

**⚠️ 常见坑：**
- email 必须和 GitHub 账号关联的邮箱一致，否则 commit 不会关联到你的 profile
- push 报 `Author identity unknown` = user.name/email 没设

### 3.4 已知路径

| 资源 | 路径 |
|------|------|
| GitHub PAT | `~/code_key` |
| gh CLI | `~/bin/gh` |
| 项目仓库 | 按具体项目路径 |
| 产出文档 | 项目内 `./docs/` 或根目录 |

### 3.5 发布前文件要求

- Public 仓库：**必须有 `LICENSE`**
- 版本发布：建议有 `CHANGELOG.md` 或独立 `RELEASE_NOTES*.md`
- 预发布（alpha/beta/rc）：notes 中应明确标注范围与风险

## 4. 角色分工

### 万三（主会话）
- 决策仓库名、可见性（public/private）、description、topics、是否 prerelease
- **先做记忆检索**：涉及 token / 账号 / 历史仓库规范时，先 `memory_search`
- 判断是否满足对外发布门槛：测试、LICENSE、说明文档、版本号、tag 命名
- 对关键元信息拍板，不把决策责任下放给执行角色

### 鲁班（执行）
- 负责 commit / tag / push / release / topics 的具体执行
- 工时上限：30 分钟内完成单次发布动作；超时需报告卡点
- 推荐模型：`claude-opus-4.7` 或 `gpt-5.4`
- 安全边界：只在目标项目目录操作；禁止误推错仓库、错 tag、错 remote

### 移公（技术补深）
- 补 gh 安装、PATH、鉴权、网络/权限问题处理
- 对 shell 命令、安装方式、跨环境差异给技术注释
- 当 `gh` / `git` / 网络鉴权出问题时作为第一技术兜底

### 包拯 / 商鞅（可选审查）
- 发布前独立检查：工作区干净、`.gitignore` 正确、无 dist/node_modules 入库、tag 与 notes 对齐
- 对外发布前做最后一次可信度校验时介入

## 5. 步骤

### Step 0. Pre-publish 安全扫描

> **为什么放最前面：** 2026-03-19 事件——敏感信息差点入库。安全问题一旦推上 public 仓库，撤回也来不及（git history 永留痕）。

**一键扫描脚本：**

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "${1:-.}"  # 参数1=仓库路径，默认当前目录

echo "=== 敏感信息扫描 ==="
FOUND=0

echo "--- PAT / Token ---"
if grep -rn "ghp_" --include="*.{md,yaml,json,sh,ts,js,mjs,cjs}" . 2>/dev/null; then
  echo "⚠️  发现 GitHub PAT！"
  FOUND=1
fi

echo "--- API Key ---"
if grep -rn "sk-" --include="*.{md,yaml,json,sh,ts,js,mjs,cjs}" . 2>/dev/null; then
  echo "⚠️  发现疑似 API Key！"
  FOUND=1
fi

echo "--- 个人路径 ---"
if grep -rn "/home/mcdowell" --include="*.{md,yaml,json,sh,ts,js,mjs,cjs}" . 2>/dev/null; then
  echo "⚠️  发现个人绝对路径！"
  FOUND=1
fi

echo "--- 内网 IP ---"
if grep -rn "192\.168\." --include="*.{md,yaml,json,sh,ts,js,mjs,cjs}" . 2>/dev/null; then
  echo "⚠️  发现内网 IP！"
  FOUND=1
fi

echo ""
echo "=== .gitignore 覆盖检查 ==="
LEAKED=$(git ls-files | grep -E "(node_modules|dist/|\.env|\.local|\.cache)" || true)
if [ -n "$LEAKED" ]; then
  echo "⚠️  以下文件不应入库:"
  echo "$LEAKED"
  FOUND=1
else
  echo "✅ .gitignore 覆盖正常"
fi

echo ""
echo "=== 大文件检查（Top 10）==="
git ls-files -z | xargs -0 du -b 2>/dev/null | sort -n | tail -10
echo "---"
du -sh .git
# 预期: .git 目录 < 50MB 为宜

echo ""
if [ $FOUND -eq 0 ]; then
  echo "✅ 安全扫描通过"
else
  echo "❌ 发现问题，修复后再发布！"
  exit 1
fi
```

**快速手动版（一行命令）：**

```bash
grep -rn "ghp_\|sk-\|/home/mcdowell\|192\.168\." --include="*.{md,yaml,json,sh,ts,js}" . 2>/dev/null && echo "⚠️ 发现敏感内容" || echo "✅ 清洁"
```

### Step 1. 本地状态检查

> **目标：** 确认本地代码已达到可发布状态，避免把脏工作区或未验证产物推上 GitHub。

```bash
git status --short
git branch --show-current
git log --oneline -n 5
npm run verify  # 或项目对应的测试/构建命令
```

**预期输出：**
- 工作区干净，或仅剩明确可接受的发布文件
- 当前分支正确（通常是 `main`）
- 最近提交能对应本次版本内容
- 测试/构建校验通过

**异常处理：**
- 工作区不干净：先整理、补 commit 或回退无关文件，再继续
- 测试失败：**停止发布，先修复；不要带病出仓**
- 发现 `dist/`、`node_modules/` 等产物被跟踪：先修 `.gitignore`，清理索引后再发

### Step 2. 凭证查记忆

> **目标：** 避免重复让用户手动登录，先复用已知凭证路径。
> **硬规则：凭证先查记忆，不要一上来就让用户重新登录。**

```bash
# 1. 先查记忆
memory_search "GitHub PAT token"

# 2. 确认本地文件
ls -la ~/code_key
head -c 4 ~/code_key
# 预期: ghp_

# 3. 验证 token 有效性
curl -s -H "Authorization: token $(cat ~/code_key)" https://api.github.com/user | jq .login
# 预期: "mcdowell8023"
```

**异常处理：**
- `memory_search` 无结果：直接检查 `~/code_key`，再看 `MEMORY.md` / 历史日记
- `~/code_key` 不存在：再让用户补充，不要一上来就让用户重新登录整套流程
- Token 无效（401）：见 [Step 3 方案 E](#方案-etoken-过期--失效处理)

### Step 3. gh 鉴权

> **目标：** 用现成 PAT 完成 CLI 认证，避免网页交互打断流程。

#### 方案 A：复用现有 PAT（首选）

```bash
cat ~/code_key | gh auth login --with-token
# 预期: 无输出 = 成功

gh auth status
# 预期:
# github.com
#   ✓ Logged in to github.com account mcdowell8023 (keyring)
#   - Active account: true
#   - Git operations protocol: https
#   - Token: ghp_****
#   - Token scopes: 'delete_repo', 'gist', 'read:org', 'repo', 'workflow', 'write:packages'
```

#### 方案 B：环境变量（CI / 一次性场景）

```bash
# 临时（单条命令）
GH_TOKEN=$(cat ~/code_key) gh repo list

# 当前 shell 全局
export GH_TOKEN=$(cat ~/code_key)
gh repo list
# 注意: GH_TOKEN 优先级高于 gh auth login 存储的凭据
```

#### 方案 C：交互式登录（兜底）

```bash
gh auth login
# 交互选择:
#   ? What account do you want to log into? GitHub.com
#   ? What is your preferred protocol for Git operations? HTTPS
#   ? Authenticate Git with your GitHub credentials? Yes
#   ? How would you like to authenticate GitHub CLI? Paste an authentication token
#   Paste your authentication token: ****
#   ✓ Logged in as mcdowell8023
```

#### 方案 D：Token Scopes 需求速查

| 操作 | 最低 Scope | 当前 ~/code_key 覆盖？ |
|------|-----------|----------------------|
| `gh repo create` | `repo` | ✅ |
| `gh release create` | `repo` | ✅ |
| `gh repo edit --add-topic` | `repo` | ✅ |
| `gh repo delete` | `delete_repo` | ✅ |
| `gh workflow run` | `workflow` | ✅ |
| 读私有仓库 | `repo` | ✅ |
| 读组织信息 | `read:org` | ✅ |
| 发 npm 包 | `write:packages` | ✅ |

**查看当前 scopes：**

```bash
gh auth status
# 看 "Token scopes:" 行

# 或 API 直接查
curl -sI -H "Authorization: token $(cat ~/code_key)" https://api.github.com | grep x-oauth-scopes
```

#### 方案 E：Token 过期 / 失效处理

**症状：** `gh auth status` → `✗ Token invalid or expired`，或命令报 HTTP 401

```bash
# 1. 确认 token 格式
wc -c ~/code_key   # 应 40-41 字节（classic PAT）
head -c 4 ~/code_key  # 应 ghp_

# 2. API 测试
curl -s -H "Authorization: token $(cat ~/code_key)" https://api.github.com/user | jq .login
# 失败: { "message": "Bad credentials" }

# 3. 续 / 重新生成
# 浏览器: https://github.com/settings/tokens → Regenerate

# 4. 更新本地
echo -n "ghp_新token内容" > ~/code_key
chmod 600 ~/code_key
cat ~/code_key | gh auth login --with-token
gh auth status  # 验证
```

### Step 4. License / Notes 检查

> **目标：** 确认对外发布材料完整，避免 public 仓库元信息缺失。

```bash
ls LICENSE CHANGELOG.md RELEASE_NOTES.md 2>/dev/null
```

**异常处理：**
- Public 仓库无 LICENSE：**先补齐再发布**
- 没有 release notes：至少生成一份最小版，说明版本变化与风险

### Step 5. 创建 GitHub 仓库（首次发布）

> **目标：** 把本地仓库正确绑定到 GitHub 远端，并一次带上 description / 可见性等核心信息。
> **决策点：** 仓库名、可见性必须万三拍板后再执行。

**确认当前分支（重要！当前分支决定远端默认分支）：**

```bash
git branch
# 预期: * main
# 如果不是 main，先切：git checkout main
```

**标准创建：**

```bash
gh repo create mcdowell8023/<repo-name> \
  --source . \
  --push \
  --public \
  --description "<description>"
# 预期:
# ✓ Created repository mcdowell8023/<repo-name> on GitHub
#   https://github.com/mcdowell8023/<repo-name>
# ✓ Added remote https://github.com/mcdowell8023/<repo-name>.git
# ✓ Pushed commits to https://github.com/mcdowell8023/<repo-name>.git
```

**附加选项：**

```bash
--homepage "https://my-project.dev"   # 主页链接
--license MIT                          # 远端创建 LICENSE（如果本地没有）
--private                              # 私有仓库
--disable-wiki --disable-issues        # 禁用 wiki / issues
```

**⚠️ 陷阱：**

| 陷阱 | 症状 | 解决 |
|------|------|------|
| `--push` 只推当前分支，不推 tag | 创建完后 `git tag` 列出的 tag 还在本地 | `git push origin --tags` 手动推 |
| 当前分支决定远端默认分支 | 在 `v1.1-dev` 上执行，远端 default = `v1.1-dev` | 先 `git checkout main` 再创建 |
| remote origin 已存在 | `fatal: remote origin already exists` | `git remote remove origin` 后重试 |

**异常处理：**
- 远端同名仓库已存在：`gh repo view <user>/<repo>` 检查 → `git remote add origin https://github.com/<user>/<repo>.git && git push -u origin main`
- 仓库名未拍板：暂停执行，先由万三确定命名与可见性

### Step 6. 推送 Tag

> **目标：** 确保版本 tag 真正到远端。**`gh repo create --push` 不会自动推 tag，必须单独 push。**

```bash
# 创建 annotated tag（推荐，有元信息）
git tag -a vX.Y.Z -m "vX.Y.Z"

# 推送单个 tag
git push origin vX.Y.Z
# 预期: * [new tag]   vX.Y.Z -> vX.Y.Z

# 验证远端 tag
git ls-remote --tags origin
# 预期: abc1234  refs/tags/vX.Y.Z
```

**Annotated vs Lightweight tag：**

```bash
# annotated（推荐）— 有作者、日期、消息
git tag -a v1.2.0 -m "v1.2.0 release"

# lightweight — 只是指针
git tag v1.2.0

# 查看 tag 类型
git cat-file -t v1.2.0
# 预期: tag（annotated）或 commit（lightweight）
```

**⚠️ 陷阱：**
- `gh release create <tag>` 会自动推那一个 tag（如果只在本地），但多 tag 发版时先 `git push --tags` 再逐个 `gh release create`
- Tag 名冲突（远端已有同名）：`git push origin :refs/tags/<tag>` 删远端再重推（**会影响已下载用户！**）
- `git push origin --tags` 会推**所有**本地 tag，包括实验性的——慎用

### Step 7. 创建 Release

> **目标：** 在 GitHub Releases 页面生成正式版本页，附上 notes 与 prerelease 状态。

**基础发布：**

```bash
gh release create vX.Y.Z \
  --title "vX.Y.Z — <主题>" \
  --notes-file RELEASE_NOTES.md \
  --prerelease
# 预期: https://github.com/mcdowell8023/<repo>/releases/tag/vX.Y.Z
```

**附带二进制资源：**

```bash
gh release create vX.Y.Z \
  --title "vX.Y.Z" \
  --notes "Stable release" \
  ./dist.tar.gz#"Source bundle" \
  ./my-cli-linux-amd64#"Linux binary"
# # 号后面是 GitHub 上显示的标签名
```

**自动生成 Release Notes（基于 commits）：**

```bash
gh release create vX.Y.Z --generate-notes
# 基于上一个 tag 到当前 tag 的 commits 自动生成
```

**草稿模式（先预览再发布）：**

```bash
gh release create vX.Y.Z --draft --title "vX.Y.Z" --notes "WIP"
# 确认后发布
gh release edit vX.Y.Z --draft=false
```

**编辑已有 Release：**

```bash
gh release edit vX.Y.Z --title "新标题" --notes "新描述"

# 追加资源文件
gh release upload vX.Y.Z ./new-file.zip --clobber
# --clobber 覆盖同名文件
```

**异常处理：**
- notes 文件路径错误：先 `ls -la RELEASE-NOTES*.md` 确认路径
- prerelease 标记漏掉：对 alpha/beta/rc **必须**补上，避免误导用户为正式稳定版

### Step 8. 设置 Topics

> **目标：** 补齐仓库发现性元信息，方便后续社区传播与搜索。
> **决策点：** topics 必须万三确认后再执行，避免来回改。

```bash
# 添加 topics（一次性设齐）
gh repo edit mcdowell8023/<repo> \
  --add-topic ai-agents \
  --add-topic openclaw \
  --add-topic claude-code

# 查看当前 topics
gh repo view mcdowell8023/<repo> --json repositoryTopics --jq '.repositoryTopics[].name'

# 删除单个 topic
gh repo edit mcdowell8023/<repo> --remove-topic <topic>
```

**注意：**
- topic 只能小写字母、数字、连字符
- 最多 20 个 topic
- topic 影响 GitHub 搜索发现

### Step 9. 最终验证

> **目标：** 确认 repo、tag、release、topics 四类结果全部落地。

```bash
# 1. 远端连通
git remote -v
# 预期: origin  https://github.com/mcdowell8023/<repo>.git (fetch/push)

# 2. 分支推送确认
git ls-remote --heads origin main
# 预期: <sha>  refs/heads/main

# 3. Tag 推送确认
git ls-remote --tags origin | grep vX.Y.Z
# 预期: <sha>  refs/tags/vX.Y.Z

# 4. Release 存在
gh release view vX.Y.Z --json url --jq .url
# 预期: https://github.com/mcdowell8023/<repo>/releases/tag/vX.Y.Z

# 5. 仓库元信息
gh repo view mcdowell8023/<repo> --json url,visibility,description,repositoryTopics
# 预期: JSON 含正确的 visibility/description/topics

# 6. 浏览器打开确认
gh repo view --web       # 打开仓库页
gh release view vX.Y.Z --web  # 打开 release 页
```

**异常处理：**
- Release 查不到：检查 tag 是否已推、tag 名与 release 名是否一致
- topics 缺失：重新执行 `gh repo edit`
- URL 能返回但网页打不开：再做浏览器人工验证，排除网络或可见性问题

## 6. 卡点 & 兜底（决策树）

> **为什么用决策树：** 发布过程中 80% 的卡点是可预见的，事先准备好排查路径比现场 Google 快 5 倍。

| # | 错误信息 / 卡点 | 排查命令 | 第一兜底 | 第二兜底 |
|---|----------------|---------|---------|---------|
| 1 | `gh: command not found` | `which gh; echo $PATH` | `export PATH="$HOME/bin:$PATH"` | 重装 §3.2 |
| 2 | `HTTP 401` / `Bad credentials` | `gh auth status` | `cat ~/code_key \| gh auth login --with-token` | 重新生成 PAT (§Step 3E) |
| 3 | `HTTP 403` / `Resource not accessible` | `gh auth status` 看 scopes | 缺权限，重新生成 PAT 加 scope | 用 Web UI 操作 |
| 4 | sudo 不可用装 gh | `which sudo` | 用 `~/bin/gh` 用户目录安装 (§3.2) | 让用户手动 sudo 安装 |
| 5 | `gh repo create` 报 `already exists` | `gh repo view <user>/<repo>` | `git remote add origin <url> && git push -u origin main` | 改仓库名或清理旧仓库 |
| 6 | Tag push `! [rejected]` (already exists) | `git ls-remote --tags origin` | 删远端: `git push origin :refs/tags/<tag>` 再推 | 谨慎 force |
| 7 | `gh release create` 报 tag not found | `git tag -l` 确认本地有 | `git push origin <tag>` 先推 tag | — |
| 8 | `--notes-file` 报 no such file | `ls -la RELEASE-NOTES*.md` | 确认文件路径相对于 cwd | 改用 `--notes "inline text"` |
| 9 | Release upload 超时 / 大文件 | `ls -lh <file>` | `gh release upload <tag> <file> --clobber` 单独上传 | 压缩后重试 |
| 10 | `git push` 报 `refusing to merge unrelated histories` | 新仓库有默认 README | `git pull origin main --allow-unrelated-histories` | `--force` push |
| 11 | `Permission denied (publickey)` | `git remote -v` 检查协议 | 用 HTTPS 而非 SSH | 配 SSH key |
| 12 | 工作区有发布污染（dist/node_modules 入库） | `git ls-files \| grep -E "(node_modules\|dist/)"` | 修 `.gitignore` + `git rm -r --cached` + 重新确认 | 暂停发布，先做清仓整理 |
| 13 | topics 漏项 | `gh repo view --json repositoryTopics` | 一次性重新设齐所有 topics | 发后补改，记录变更 |

## 7. 验证清单

发布后逐条确认：

```bash
# 快速全量验证（可直接复制执行）
echo "=== 验证清单 ==="

echo "1. Remote"
git remote -v | grep origin

echo "2. Branch"
git ls-remote --heads origin main

echo "3. Tag"
git ls-remote --tags origin | grep vX.Y.Z

echo "4. Release"
gh release view vX.Y.Z --json url --jq .url

echo "5. Repo meta"
gh repo view mcdowell8023/<repo> --json url,visibility,repositoryTopics

echo "6. Git status clean"
git status --short

echo "=== Done ==="
```

**Checklist：**

- [ ] `git remote -v` 含 `origin` 指向正确 GitHub URL
- [ ] `git ls-remote --tags origin` 含目标 tag
- [ ] `gh release view <tag> --json url` 有结果
- [ ] `gh repo view --json repositoryTopics` 含预期 topics
- [ ] 浏览器打开 repo URL / release URL 可访问
- [ ] Public 仓库已确认 LICENSE 存在
- [ ] `git status` 确认无误入库产物（dist / node_modules / 本地缓存）
- [ ] 安全扫描通过（Step 0）

## 8. 应急处理

> **为什么需要：** 发布后发现严重问题（安全漏洞、敏感信息泄露、错误版本），需要快速撤回。

### 8.1 删除 Release（保留 tag）

```bash
gh release delete vX.Y.Z --yes
# 预期: ✓ Deleted release vX.Y.Z
# tag 仍在：git ls-remote --tags origin | grep vX.Y.Z → 有结果
```

### 8.2 删除远端 Tag

```bash
git push origin :refs/tags/vX.Y.Z
# 或
git push --delete origin vX.Y.Z
# 预期: - [deleted]  vX.Y.Z
```

### 8.3 删除本地 Tag

```bash
git tag -d vX.Y.Z
# 预期: Deleted tag 'vX.Y.Z'
```

### 8.4 完全撤回（Release + Tag 一起删）

```bash
gh release delete vX.Y.Z --yes 2>/dev/null; true
git push origin :refs/tags/vX.Y.Z
git tag -d vX.Y.Z
echo "✅ Release + Tag 已完全撤回"
```

### 8.5 删除仓库（不可逆！）

```bash
# 需要 delete_repo scope（当前 ~/code_key 有）
gh repo delete mcdowell8023/<repo> --yes
# 预期: ✓ Deleted repository mcdowell8023/<repo>
# ⚠️ 30 天内可联系 GitHub 支持恢复，之后永久删除
```

### 8.6 改可见性（公转私 / 私转公）

```bash
gh repo edit mcdowell8023/<repo> --visibility private
# 仓库变私有，所有 fork/star 用户失去访问

gh repo edit mcdowell8023/<repo> --visibility public
# 仓库重新公开
```

## 9. 常用变体

### 9.1 二次发布（已有仓库，新版本）

覆盖 80% 后续发版场景的简化流程：

```bash
cd /path/to/repo

# 1. 确认状态
git status
git branch
# 预期: * main, nothing to commit

# 2. 提交变更
git add -A
git commit -m "feat: vX.Y.Z changes"

# 3. 打 annotated tag
git tag -a vX.Y.Z -m "vX.Y.Z release"

# 4. Push 分支 + tag（两条命令，明确可控）
git push origin main
git push origin vX.Y.Z

# 5. 创建 Release
gh release create vX.Y.Z \
  --title "vX.Y.Z — <主题>" \
  --notes-file RELEASE-NOTES-vX.Y.Z.md

# 6. 验证
gh release view vX.Y.Z
```

**快捷版（分支 + tag 一次推）：**

```bash
git push origin main --follow-tags
# --follow-tags 推送 annotated tag（不推 lightweight tag）
```

## 10. 历史教训

> **为什么记录教训：** 同一个坑踩两次是流程问题，不是人的问题。每次踩坑必须更新此表。

| 日期 | 事件 | 教训 | 影响步骤 |
|------|------|------|---------|
| 2026-04-26 | 万三让显超手动 `gh auth login`，实际 `~/code_key` 早有 PAT | **凭证类问题先查记忆**，不要让用户重复做已做过的事 | Step 2, §3.1 |
| 2026-03-19 | 第一次推 GitHub 没遵守规范（私有 / `oc-` 前缀 / `main` 分支），后续才补写 TOOLS.md | 发布前先查历史 GitHub 规范 | Step 1 |
| 2026-03-19 | 敏感信息差点入库 | **发布前必须跑安全扫描** | Step 0 |
| alpha.1 翻车 | `dist/` 入库 + `node_modules/` 入库 | 发布前必须过 `.gitignore` 检查 + `git status` 二次确认 | Step 0, Step 1 |
| 2026-04-26 alpha.2 | `gh repo create` 完成后 tag 仍需单独推 | **repo 创建成功 ≠ 版本发布完整闭环**；`--push` 不推 tag | Step 6 |
| 2026-04-26 alpha.2 | 双轨审查价值验证，代码审查通过后仍被端到端实跑抓出 Blocker | 发布前验证不能只看代码面 | Step 9 |

## 11. 相关 Playbook

- `skill-publish.md`（待）— 当 GitHub 发布之外还要同步 Skill 分发时使用
- `npm-publish.md`（待）— 当需要同时发布 npm 包时串联使用
- `release-readiness-review.md`（待）— 当需要正式发布前独立审查时使用

## 12. 维护说明

- 本文只固化稳定发布流程；项目特定命名、版本号、topics 由万三在执行前拍板
- 如果未来形成更完整的发布体系，应把"预发布 / 正式发布 / npm 发布 / Skill 发布"拆成独立 Playbook，再用此文作为 GitHub 部分的公共底座
- 技术命令部分由移公维护，流程框架由范蠡维护

## 修订记录

- 2026-04-26 v1.0 范蠡（流程框架）+ 移公（技术深度）合并定稿

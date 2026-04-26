# Playbook: npm 包发布

> **类型：** How-to Guide / npm 包发布执行手册
> **最后更新：** 2026-04-26
> **维护人：** 移公（techops）

---

## 1. 适用场景

- 将 TypeScript / JavaScript 项目作为 npm 包发布到 npmjs.com（公开或私有 scope）
- 已发布包的版本迭代（patch / minor / major / prerelease）
- 需要同时发布到 npm + GitHub Release 的双渠道发布
- scope 包（如 `@openclaw/xxx`）的首次发布和后续更新
- CI/CD 自动化发布（GitHub Actions / 其他 CI）

## 2. 不适用场景

- 纯应用项目（不是包）——没有 `main` / `exports` 字段、不被别人 `npm install`
- 仅本地使用的脚本，不需要分发
- 私有 npm registry（如 Verdaccio、GitHub Packages）——流程类似但鉴权不同，本文只覆盖 npmjs.com
- 只做 GitHub Release 不发 npm → 用 `github-release.md`

## 3. 前置条件

> **为什么先检查前置条件：** npm 发布失败 90% 是鉴权问题和 package.json 配置不全。提前确认比发布时排障省 10 倍时间。

### 3.1 凭证

- **硬规则：先查记忆，再决定是否让用户登录**
- 推荐检索：`memory_search "npm token publish 账号"`
- npm token 存放路径：`~/.npmrc`（格式 `//registry.npmjs.org/:_authToken=npm_xxxxx`）
- CI 场景：环境变量 `NPM_TOKEN` 或 GitHub Secrets `secrets.NPM_TOKEN`

```bash
# 快速确认当前鉴权状态
npm whoami
# 预期: 你的 npm 用户名
# 失败: ENEEDAUTH / 403 → 需要登录或配置 token

# 检查 .npmrc 是否有 token
cat ~/.npmrc 2>/dev/null | grep -o '//registry.npmjs.org/:_authToken=npm_.*' | head -c 40
# 预期: //registry.npmjs.org/:_authToken=npm_xxxx（截断显示，不要泄漏完整 token）

# 检查 token 权限（API 验证）
npm profile get
# 预期: 显示 name / email / 2FA 状态等
```

**没有 npm 账号时：**

1. 注册：https://www.npmjs.com/signup
2. 验证邮箱（必须，否则无法发布）
3. **强烈推荐开启 2FA**（npmjs.com → Account Settings → Two-Factor Authentication）
   - 类型选 `Authorization and Publishing`（最安全）
   - 或至少 `Authorization only`

### 3.2 工具

```bash
# Node.js（>=20.0.0）
node -v
# 预期: v20.x.x 或 v22.x.x

# npm（>=10.0.0，随 Node 自带）
npm -v
# 预期: 10.x.x

# 可选：npm-check-updates（检查依赖版本）
npx ncu
```

### 3.3 已知路径

| 路径 | 说明 |
|------|------|
| `~/.npmrc` | npm 鉴权 token |
| `~/open-claw-output/code/` | 项目产出目录 |
| `~/open-claw-output/doc/playbooks/github-release.md` | GitHub Release 协同 Playbook |

## 4. 角色分工

### 万三（主会话）
- 决策：发布版本号、是否公开、发布渠道（npm-only / GitHub-only / 双发）
- **发布前必须先查记忆**确认 npm 凭证状态
- 协调 npm publish 与 GitHub Release 的执行顺序
- 最终验证（npm view + 安装测试）

### 移公（techops）
- 执行：鉴权配置、构建、打包验证、发布命令、发布后验证
- 工时上限：30 分钟（单次发布）
- 推荐模型：`claude-opus-4.6`
- 安全边界：只操作项目目录和 `~/.npmrc`，不修改系统级配置

### 商鞅（autotest）/ 包拯（reviewer）
- 发布前：检查 `npm pack --dry-run` 打包内容、`npm publish --dry-run` 模拟
- 发布后：安装测试、import 测试、类型可用性测试

## 5. 步骤

### 步骤 1：确认发布渠道和版本号

**目标**
- 明确本次发布的版本号、dist-tag、发布渠道

**操作**
```bash
# 查看当前版本
cat package.json | jq -r '.version'

# 查看已发布版本（如果不是首发）
npm view @scope/package-name versions --json 2>/dev/null || echo "首次发布，无历史版本"

# 查看 dist-tags
npm view @scope/package-name dist-tags --json 2>/dev/null || echo "首次发布"
```

**决策点**
- 版本号：`npm version patch/minor/major/prerelease` 还是自定义？
- dist-tag：`latest`（默认）/ `beta` / `alpha` / `next`？
- 渠道：npm-only / GitHub-only / 双发？

**预期输出**
- 明确的版本号（如 `1.0.0-alpha.1`）
- 明确的发布渠道

---

### 步骤 2：package.json 检查与修正

**目标**
- 确保 package.json 所有必备字段齐全，避免发布后元信息残缺

**命令**
```bash
# 一次性检查所有关键字段
node -e "
const pkg = require('./package.json');
const checks = {
  name:        pkg.name,
  version:     pkg.version,
  description: pkg.description,
  main:        pkg.main,
  types:       pkg.types || pkg.typings,
  files:       pkg.files,
  license:     pkg.license,
  repository:  pkg.repository,
  keywords:    pkg.keywords,
  engines:     pkg.engines,
  publishConfig: pkg.publishConfig,
};
const missing = Object.entries(checks).filter(([k,v]) => !v);
if (missing.length) {
  console.log('❌ 缺失字段:');
  missing.forEach(([k]) => console.log('  -', k));
  process.exit(1);
} else {
  console.log('✅ 所有关键字段齐全');
  console.log(JSON.stringify(checks, null, 2));
}
"
```

**必备字段参考（完整模板）：**

```jsonc
{
  "name": "@openclaw/self-learning-loop",
  "version": "1.0.0-alpha.1",
  "description": "一句话描述包的功能",
  "main": "dist/index.js",
  "module": "dist/index.mjs",           // ESM 入口（可选但推荐）
  "types": "dist/index.d.ts",
  "exports": {                           // 现代 Node.js 入口（>=16.x）
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "engines": {
    "node": ">=20.0.0"
  },
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/mcdowell8023/package-name.git"
  },
  "bugs": {
    "url": "https://github.com/mcdowell8023/package-name/issues"
  },
  "homepage": "https://github.com/mcdowell8023/package-name#readme",
  "keywords": ["openclaw", "ai", "learning"],
  "license": "MIT",
  "scripts": {
    "build": "tsdown",
    "test": "vitest run",
    "prepublishOnly": "npm run build && npm test"
  }
}
```

**⚠️ 关键陷阱：**

| 陷阱 | 后果 | 预防 |
|------|------|------|
| scope 包没设 `publishConfig.access: "public"` | `npm publish` 报 402 Payment Required | 加 `publishConfig` 或用 `--access public` |
| `files` 字段漏了 | 整个项目被打包（含 node_modules、.git 等） | 必填 `files`，用 `npm pack --dry-run` 验证 |
| `main` 指向不存在的文件 | 安装后 `require()` 报错 | 先 `npm run build` 再检查 dist/ |
| 没有 `types` 字段 | TypeScript 项目无法获得类型提示 | 确保 tsconfig 输出 `.d.ts`，`types` 指向它 |
| `engines` 不写 | 用户低版本 Node 安装后运行时报错 | 写明最低 Node 版本 |
| `prepublishOnly` 没配 | 可能发布未构建的代码 | 加上 `npm run build && npm test` |

**预期输出**
- 所有关键字段检查通过
- `publishConfig.access` 对 scope 包为 `"public"`

**异常情况**
- 字段缺失：按模板补齐，重新检查
- scope 名被占用：`npm view @scope/name` 确认，必要时改名

---

### 步骤 3：干净构建

**目标**
- 从零构建，确保 dist/ 是最新代码产物

**命令**
```bash
# 干净构建（强制从 lockfile 安装）
rm -rf dist node_modules
npm ci
npm run build

# 如果没有 lockfile（首次）
rm -rf dist node_modules
npm install
npm run build
```

**预期输出**
```bash
# 验证 dist 内容
ls -la dist/
# 预期: index.js, index.d.ts, index.mjs（取决于构建配置）

file dist/index.js
# 预期: ASCII text / UTF-8 Unicode text

file dist/index.d.ts
# 预期: ASCII text / UTF-8 Unicode text（类型声明文件）

# 快速语法检查
node -e "require('./dist/index.js')"
# 预期: 无报错退出
```

**异常情况**
- `npm ci` 失败 → `package-lock.json` 和 `package.json` 不一致 → 先 `npm install` 再 `npm ci`
- `npm run build` 失败 → 看报错，通常是 TypeScript 编译错误
- dist/ 为空 → 检查 build 脚本配置（tsconfig.json / tsdown.config.ts）

---

### 步骤 4：测试

**目标**
- 确保代码质量，发布前最后一道门

**命令**
```bash
# 运行测试
npm test

# 如果有 lint
npm run lint 2>/dev/null || echo "无 lint 脚本"

# 如果有 typecheck
npm run typecheck 2>/dev/null || npx tsc --noEmit 2>/dev/null || echo "无 typecheck"
```

**预期输出**
- 所有测试通过
- 无 lint 错误

**异常情况**
- 测试失败 → 修代码，不要带着失败测试发布
- 没有测试脚本 → 至少做步骤 3 的 `node -e "require('./dist')"` 冒烟测试

---

### 步骤 5：打包预览（强制步骤）

> ⚠️ **这步不能跳过。** 发布前必须确认打包内容和体积。

**目标**
- 确认将要发布的文件列表、体积、无敏感信息泄漏

**命令**
```bash
# 方法 1：dry-run 预览
npm pack --dry-run 2>&1
# 输出: 文件列表 + 总体积

# 方法 2：实际打包后检查
npm pack
ls -la *.tgz
# 看体积（通常 <100KB 对于工具库）

# 查看包内文件清单
tar -tzf *.tgz | head -80
# 预期: package/dist/... package/README.md package/LICENSE package/package.json

# 查看包内是否有不该发的文件
tar -tzf *.tgz | grep -E '\.(env|key|pem|secret|npmrc)' && echo "⚠️ 发现敏感文件！" || echo "✅ 无敏感文件"

# 查看包内是否有大文件
tar -tzf *.tgz -v | sort -k3 -n -r | head -10
# 看最大的 10 个文件

# 发布 dry-run（最终确认）
npm publish --dry-run 2>&1
# 检查 version, files, 体积
```

**预期输出**
- 文件列表只含 `dist/`、`README.md`、`LICENSE`、`package.json`
- 体积合理（工具库通常 <100KB，大型库 <1MB）
- 无 `.env`、`.npmrc`、`node_modules`、`.git`、`src/`（源码看你是否想包含）

**异常情况**
- 包含了不该发的文件 → 修改 `files` 字段或添加 `.npmignore`
- 体积太大（>1MB） → 检查是否误包含了测试文件、文档图片、node_modules
- 没有 README.md → npmjs.com 包页面会是空白的，**必须有**

**清理打包产物**
```bash
rm -f *.tgz
```

---

### 步骤 6：鉴权

**目标**
- 确保当前终端有 npm 发布权限

**方案 A：交互式登录（手动发布）**
```bash
npm login
# 按提示输入：
#   Username: 你的 npm 用户名
#   Password: 你的 npm 密码
#   Email: 你的注册邮箱
#   OTP: 6位码（如果开启了 2FA）

# 验证
npm whoami
# 预期: 你的 npm 用户名
```

**方案 B：~/.npmrc + Granular Token（推荐）**
```bash
# 1. 在 npmjs.com 生成 token
#    → 登录 npmjs.com → 头像 → Access Tokens → Generate New Token
#    → 类型选 Granular Access Token
#    → 权限：Read and Write（至少 publish 需要 write）
#    → Packages：选 "Only select packages and scopes" 或 "All packages"
#    → 过期时间：推荐 90 天（定期轮换）

# 2. 写入 .npmrc
echo "//registry.npmjs.org/:_authToken=npm_xxxxxxxxxxxxxxxx" > ~/.npmrc
chmod 600 ~/.npmrc

# 3. 验证
npm whoami
# 预期: 你的 npm 用户名

# ⚠️ 安全提醒：
# - .npmrc 权限必须 600（仅本人可读写）
# - 不要把 .npmrc 提交到 git（确保 .gitignore 包含 .npmrc）
# - token 不要出现在命令行参数里（会被 shell history 记录）
```

**方案 C：环境变量（CI 专用）**
```bash
# GitHub Actions 中
env:
  NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

# 本地临时使用（不推荐，仅调试）
NPM_TOKEN=npm_xxx npm publish
```

**方案 D：scope 绑定 registry**
```bash
# 让 @openclaw scope 指向 npm 官方 registry
npm config set @openclaw:registry https://registry.npmjs.org/

# 验证
npm config get @openclaw:registry
# 预期: https://registry.npmjs.org/
```

**预期输出**
- `npm whoami` 返回正确用户名
- `~/.npmrc` 权限为 `600`

**异常情况**
- `npm whoami` 报 `ENEEDAUTH` → token 无效或不存在，重新生成
- `npm whoami` 报 `E403` → token 权限不够，重新生成 Granular Token 并勾选 write
- 2FA 认证失败 → 确认 OTP 应用时间同步，重新输入

---

### 步骤 7：版本号管理

**目标**
- 确认/设置正确的版本号，处理 git tag 协同

**自动升版（推荐，会自动 git commit + tag）**
```bash
# 小版本修复
npm version patch
# 1.0.0 → 1.0.1
# 自动: git commit -m "1.0.1" + git tag v1.0.1

# 新功能
npm version minor
# 1.0.0 → 1.1.0

# 破坏性变更
npm version major
# 1.0.0 → 2.0.0

# 预发布版本
npm version prerelease --preid=alpha
# 1.0.0 → 1.0.1-alpha.0
# 再次执行: 1.0.1-alpha.0 → 1.0.1-alpha.1

npm version prerelease --preid=beta
# 从 alpha 切到 beta: 需要先手动设版本
npm version 1.0.1-beta.0

# 自定义版本号
npm version 2.0.0-rc.1

# 自定义 commit message
npm version patch -m "release: v%s"
# %s 会被替换为版本号
```

**⚠️ `npm version` 与 git 的交互：**

| 行为 | 默认 | 控制方式 |
|------|------|----------|
| 修改 package.json | ✅ 始终 | — |
| git commit | ✅ 在 git 仓库内 | `--no-git-tag-version` 跳过 |
| git tag | ✅ 在 git 仓库内 | `--no-git-tag-version` 跳过 |
| tag 前缀 | `v` | `tag-version-prefix` 配置 |
| commit message | `版本号` | `-m "模板 %s"` |

**与 github-release.md 协同时：**
```bash
# 如果你打算用 github-release.md 的流程管理 tag：
# 方案 1：npm version 管理 tag，github-release.md 只做 release
npm version patch
git push origin main --tags
# 然后用 gh release create 创建 release

# 方案 2：npm version 不管 tag，手动管理
npm version patch --no-git-tag-version
git add package.json package-lock.json
git commit -m "chore: bump version to $(jq -r .version package.json)"
# tag 和 release 完全交给 github-release.md
```

**预期输出**
- `package.json` 中 version 已更新
- git log 中有对应 commit（如果没用 `--no-git-tag-version`）
- git tag 中有对应 tag（如果没用 `--no-git-tag-version`）

**异常情况**
- `npm version` 报 `Git working directory not clean` → 先 commit 或 stash 未提交的改动
- tag 已存在 → `git tag -d vX.Y.Z` 删除本地 tag，重新执行
- 版本号已在 npm 发布过 → 必须用新版本号，npm 不允许覆盖已发布版本

---

### 步骤 8：发布

**目标**
- 执行 npm publish，将包推送到 npmjs.com

**命令**
```bash
# ========================================
# ⚠️ 强制：发布前先 dry-run
# ========================================
npm publish --dry-run
# 仔细检查输出：
#   - name 和 version 正确
#   - 文件列表符合预期
#   - 体积合理

# ========================================
# 正式发布
# ========================================

# scope 包首次发布（必须 --access public，否则 402）
npm publish --access public

# 非 scope 包 / 已配置 publishConfig 的 scope 包
npm publish

# 预发布版本（不更新 latest tag）
npm publish --tag alpha
npm publish --tag beta
npm publish --tag next

# 2FA 场景（手动输入 OTP）
npm publish --access public --otp=123456

# 完整命令示例（scope 包 + beta + OTP）
npm publish --access public --tag beta --otp=123456
```

**dist-tag 说明：**

| tag | 用途 | `npm install` 行为 |
|-----|------|-------------------|
| `latest` | 默认，正式版 | `npm install pkg` 安装这个 |
| `beta` | 测试版 | `npm install pkg@beta` |
| `alpha` | 早期测试 | `npm install pkg@alpha` |
| `next` | 下一个大版本预览 | `npm install pkg@next` |

**管理 dist-tag：**
```bash
# 查看当前 tags
npm dist-tag ls @scope/package

# 添加/移动 tag
npm dist-tag add @scope/package@1.0.0-beta.1 beta

# 删除 tag
npm dist-tag rm @scope/package beta

# 把 beta 提升为 latest
npm dist-tag add @scope/package@1.0.0 latest
```

**预期输出**
```
npm notice
npm notice 📦  @openclaw/self-learning-loop@1.0.0-alpha.1
npm notice === Tarball Contents ===
npm notice 1.2kB dist/index.js
npm notice 0.8kB dist/index.d.ts
npm notice 2.1kB README.md
npm notice 1.1kB LICENSE
npm notice 0.9kB package.json
npm notice === Tarball Details ===
npm notice name:          @openclaw/self-learning-loop
npm notice version:       1.0.0-alpha.1
npm notice filename:      openclaw-self-learning-loop-1.0.0-alpha.1.tgz
npm notice package size:  2.3 kB
npm notice unpacked size: 6.1 kB
npm notice total files:   5
npm notice
+ @openclaw/self-learning-loop@1.0.0-alpha.1
```

**异常情况**
- `402 Payment Required` → scope 包没 `--access public`，重新执行
- `403 Forbidden` → 鉴权问题，回步骤 6
- `E409 Conflict` → 版本号已存在，升版本后重试
- `EOTP` → 需要 OTP，加 `--otp=xxxxxx`
- 网络超时 → 检查 registry 配置：`npm config get registry`
- `EPUBLISHCONFLICT` → 包名被其他人占用，需要改名

---

### 步骤 9：发布后验证

> ⚠️ **这步不能跳过。** 发布成功不等于能用。

**目标**
- 确认包在 npm 可见、可安装、可使用

**命令**
```bash
# 1. 查看包元信息
npm view @scope/package-name
# 预期: 显示 name, version, description, dist-tags 等

# 2. 查看所有版本
npm view @scope/package-name versions --json
# 预期: 包含刚发布的版本号

# 3. 查看特定版本详情
npm view @scope/package-name@1.0.0-alpha.1
# 预期: 显示该版本的完整元信息

# 4. 安装测试（隔离环境）
TMP_TEST=$(mktemp -d)
cd "$TMP_TEST"
npm init -y
npm install @scope/package-name@1.0.0-alpha.1

# 5. CJS 导入测试
node -e "const pkg = require('@scope/package-name'); console.log('CJS OK:', typeof pkg);"
# 预期: CJS OK: object（或 function，取决于导出）

# 6. ESM 导入测试（如果支持）
node --input-type=module -e "import pkg from '@scope/package-name'; console.log('ESM OK:', typeof pkg);" 2>/dev/null || echo "ESM 不支持或需要 type:module"

# 7. TypeScript 类型检查（如果有 types）
ls node_modules/@scope/package-name/dist/*.d.ts
# 预期: 存在 .d.ts 文件

# 8. 清理测试目录
rm -rf "$TMP_TEST"

# 9. 网页确认
echo "https://www.npmjs.com/package/@scope/package-name"
# 在浏览器打开确认包页面正常
```

**预期输出**
- `npm view` 返回正确元信息
- 安装测试成功
- require / import 无报错
- npmjs.com 包页面可访问

**异常情况**
- `npm view` 报 404 → npm 有传播延迟（通常 <5 分钟），等一下再试
- 安装后 require 报错 → `main` 字段指向错误，需要发补丁版本
- 类型文件缺失 → `types` 字段或 `files` 字段配置问题

---

### 步骤 10：撤回 / 应急

**目标**
- 发布了有问题的版本，需要撤回或标记弃用

**24 小时内——unpublish（完全删除）**
```bash
# 删除特定版本
npm unpublish @scope/package-name@1.0.0 --otp=123456
# ⚠️ 24h 后此操作不可用

# 删除整个包（慎重！需要 --force）
npm unpublish @scope/package-name --force --otp=123456
# ⚠️ 删除后 24h 内不能重新发布同名包
```

**24 小时后——deprecate（标记弃用，不删除）**
```bash
# 弃用特定版本
npm deprecate @scope/package-name@1.0.0 "Critical bug, please upgrade to 1.0.1"

# 弃用版本范围
npm deprecate @scope/package-name@"< 1.0.1" "Deprecated, upgrade to >=1.0.1"

# 撤销弃用（传空字符串）
npm deprecate @scope/package-name@1.0.0 ""
```

**紧急修复流程：**
```bash
# 1. 立即 deprecate 问题版本
npm deprecate @scope/package-name@1.0.0 "CRITICAL: security issue, use 1.0.1"

# 2. 快速修复代码
git checkout -b hotfix/1.0.1
# ... 修复 ...

# 3. 发布修复版本
npm version patch
npm publish --access public

# 4. 确认 latest 指向修复版本
npm view @scope/package-name dist-tags
# 预期: latest: 1.0.1
```

---

### 步骤 11：CI 自动化发布

**目标**
- 配置 GitHub Actions 在 push tag 时自动发布到 npm

**GitHub Actions 工作流（`.github/workflows/npm-publish.yml`）：**

```yaml
name: Publish to npm

on:
  push:
    tags:
      - 'v*'  # 匹配 v1.0.0, v1.0.0-alpha.1 等

# 防止并发发布
concurrency:
  group: npm-publish
  cancel-in-progress: false

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # npm provenance 需要

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          registry-url: 'https://registry.npmjs.org'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Test
        run: npm test

      - name: Verify package contents
        run: npm pack --dry-run

      - name: Publish
        run: npm publish --access public --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

      - name: Verify publication
        run: |
          sleep 10
          npm view $(jq -r .name package.json)@$(jq -r .version package.json)
```

**GitHub Secrets 配置：**
```
仓库 → Settings → Secrets and variables → Actions → New repository secret
  Name: NPM_TOKEN
  Value: npm_xxxxxxxxxxxxxxxx（Granular Token，write 权限）
```

**npm provenance（来源证明，推荐）：**
- `--provenance` 标志让 npm 记录包是从哪个 CI 构建发布的
- 需要 `id-token: write` 权限
- 发布后在 npmjs.com 包页面会显示 "Provenance" 徽章

**预发布版本的 CI：**
```yaml
# 在 publish step 中判断 tag 类型
- name: Publish
  run: |
    VERSION=$(jq -r .version package.json)
    if [[ "$VERSION" == *"-alpha"* ]] || [[ "$VERSION" == *"-beta"* ]] || [[ "$VERSION" == *"-rc"* ]]; then
      TAG=$(echo "$VERSION" | grep -oP '(?<=-)[a-z]+' | head -1)
      npm publish --access public --tag "$TAG" --provenance
    else
      npm publish --access public --provenance
    fi
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

### 步骤 12：与 GitHub Release 的协同

**目标**
- 协调 npm publish 和 GitHub Release 的执行顺序，避免版本不一致

**推荐顺序（npm 先发）：**

```
1. 代码准备完毕，所有测试通过
2. npm version patch/minor/major（自动 commit + tag）
3. npm publish --access public（先发 npm）
4. npm publish 成功 → git push origin main --tags
5. gh release create vX.Y.Z（创建 GitHub Release）
   → 按 github-release.md 流程执行
```

**为什么 npm 先发：**
- npm publish 失败（鉴权/网络/冲突）比 GitHub Release 更常见
- 如果 npm 失败，git tag 还没 push，可以 `git tag -d vX.Y.Z` 删掉重来
- 如果 GitHub Release 先发了但 npm 失败，用户看到 Release 却装不到包 → 更糟糕

**备选顺序（GitHub 先发，适合 GitHub-only 项目暂不发 npm）：**

```
1. 按 github-release.md 完成 GitHub Release
2. 后续决定要发 npm 时，从对应 tag checkout
   git checkout vX.Y.Z
3. 确保 package.json version 与 tag 一致
4. npm publish --access public
```

**`npm version` 与手动 tag 的协调：**

```bash
# 场景 A：用 npm version 管理一切（推荐）
npm version patch                    # 改 package.json + commit + tag
npm publish --access public          # 发 npm
git push origin main --tags          # push commit + tag
gh release create v1.0.1 --generate-notes  # 创建 GitHub Release

# 场景 B：手动管理 tag（与 github-release.md 分工明确）
npm version patch --no-git-tag-version  # 只改 package.json，不 commit 不 tag
git add package.json package-lock.json
git commit -m "chore: release v1.0.1"
git tag v1.0.1
npm publish --access public
git push origin main --tags
gh release create v1.0.1 --generate-notes

# 场景 C：CI 自动化（tag push 触发）
npm version patch                    # 本地 commit + tag
git push origin main --tags          # push 触发 CI
# CI 自动 npm publish（步骤 11 的 workflow）
# CI 或手动创建 GitHub Release
```

**双渠道发布检查清单：**
```bash
# 发布后确认两个渠道版本一致
NPM_VER=$(npm view @scope/package-name version)
GH_TAG=$(gh release view --repo owner/repo --json tagName -q .tagName | sed 's/^v//')
if [ "$NPM_VER" = "$GH_TAG" ]; then
  echo "✅ 版本一致: $NPM_VER"
else
  echo "❌ 版本不一致! npm=$NPM_VER github=$GH_TAG"
fi
```

---

## 6. 卡点 & 兜底（决策树）

| # | 卡点 | 症状 | 排查命令 | 第一兜底 | 第二兜底 |
|---|------|------|----------|---------|---------|
| 1 | `403 Forbidden` | 发布被拒 | `npm whoami` | 重新 `npm login` 或重生成 token | 检查 token scope 是否含 write/publish |
| 2 | `402 Payment Required` | scope 包发布报错 | `npm publish --dry-run` 看是否 scope | `npm publish --access public` | `publishConfig.access = "public"` |
| 3 | 包名被占用 | `E403` 或 `EPUBLISHCONFLICT` | `npm view <name>` | 加 scope（`@org/name`） | 改名 |
| 4 | 版本号已存在 | `E409 Conflict` | `npm view pkg versions` | `npm version patch` 升版本 | 手动改 package.json |
| 5 | `EOTP` 2FA 失败 | OTP 错误 | 检查认证器时间 | `--otp=新6位码` | 关闭 2FA 后重试（不推荐） |
| 6 | 包内容不对 | 发布后 require 报错 | `npm pack --dry-run` | 修改 `files` 字段 | 添加 `.npmignore` |
| 7 | 体积太大 | 包 >10MB | `tar -tzf *.tgz` 排查 | 精简 `files` 字段 | 排查是否包含了 test/docs/assets |
| 8 | 类型文件缺失 | TS 项目无类型提示 | `ls dist/*.d.ts` | 检查 tsconfig `declaration: true` | 确认 `types` 字段指向正确 |
| 9 | 网络超时 | `ETIMEDOUT` | `npm config get registry` | 设置代理 `HTTPS_PROXY` | 切官方源 `npm config set registry https://registry.npmjs.org/` |
| 10 | Token 过期 | `npm whoami` 报 401 | `npm whoami` | 重新生成 Granular Token | `npm login` 交互式登录 |
| 11 | `ENEEDAUTH` | 未登录 | `cat ~/.npmrc` | 配置 `.npmrc` token | `npm login` |
| 12 | `ERR! publish Failed PUT 404` | registry URL 错误 | `npm config get registry` | `npm config set registry https://registry.npmjs.org/` | 删除 `.npmrc` 中错误的 registry 行 |
| 13 | git working dir not clean | `npm version` 拒绝执行 | `git status` | 先 commit 或 stash | `npm version --no-git-tag-version` |
| 14 | `prepublishOnly` 脚本失败 | build/test 报错阻止发布 | 看脚本报错 | 修复 build/test | 临时 `--ignore-scripts`（不推荐） |

## 7. 验证清单

- [ ] `npm whoami` 返回正确用户名
- [ ] `package.json` 所有必备字段齐全（name/version/main/types/files/license/publishConfig）
- [ ] scope 包已配置 `publishConfig.access: "public"`
- [ ] `npm run build` 成功，`dist/` 内容正确
- [ ] `npm test` 通过
- [ ] `npm pack --dry-run` 文件列表符合预期
- [ ] `npm pack --dry-run` 无敏感文件（.env / .npmrc / 密钥）
- [ ] `npm publish --dry-run` 版本号和名称正确
- [ ] `npm publish` 成功（正式发布）
- [ ] `npm view @scope/package@version` 返回正确元信息
- [ ] 隔离环境安装测试通过（`npm install` + `require()`）
- [ ] TypeScript 类型文件可用（`.d.ts` 存在）
- [ ] npmjs.com 包页面可访问且信息正确
- [ ] 如双渠道发布：npm 版本与 GitHub Release tag 一致

## 8. 历史教训

- **2026-04-26 self-learning-loop alpha.1/alpha.2**：GitHub-only 发布，npm 未发。教训：发布前明确发布渠道（GitHub/npm/双发），写入 release checklist。
- **2026-03-19 oc-skill-memory-system**：首次 GitHub 发布没规范，npm 也没发。教训：每个项目在首次发布前必须确认发布渠道和 package.json 完整性。
- **scope 包默认私有**：这是 npm 新手最常见翻车点。`@scope/xxx` 默认 restricted（需要付费），必须 `--access public` 或 `publishConfig.access = "public"`。
- **`npm version` 自动 tag 与 GitHub Release 冲突**：如果两边都在管 tag，容易出现 tag 不一致。统一用一种方式管理（推荐 `npm version` 管 tag，GitHub Release 只做 release notes）。
- **发布未构建的代码**：忘记 `npm run build` 就发布 → dist/ 是旧代码。加 `prepublishOnly` 脚本自动构建。
- **忘记 dry-run**：发布了包含 `.env` 的版本 → 凭证泄漏。必须先 `npm pack --dry-run` 检查。

## 9. 相关 Playbook

- `github-release.md` — GitHub Release 发布流程。双渠道发布时先完成 npm publish 再跳转此 Playbook
- `skill-publish.md`（待创建）— OpenClaw Skill 发布流程。如果包同时是 Skill，发布后还需更新 Skill Hub

## 10. 快速参考卡片

> 给熟手看的速查，不看正文直接用。

```bash
# === npm 包发布速查 ===

# 0. 鉴权
npm whoami || npm login

# 1. 构建
rm -rf dist && npm ci && npm run build && npm test

# 2. 预检
npm pack --dry-run                    # 看文件列表
npm publish --dry-run                 # 看发布预览

# 3. 升版本
npm version patch                     # 或 minor / major / prerelease --preid=alpha

# 4. 发布
npm publish --access public           # scope 包首次必须
npm publish --tag beta                # 预发布
npm publish --otp=123456              # 2FA

# 5. 验证
npm view @scope/pkg@version
TMP=$(mktemp -d) && cd $TMP && npm init -y && npm i @scope/pkg && node -e "require('@scope/pkg')" && rm -rf $TMP

# 6. 应急
npm unpublish @scope/pkg@version      # 24h 内
npm deprecate @scope/pkg@version "msg" # 24h 后
```

## 11. 维护说明

- 本 Playbook 只写稳定流程，不写一次性背景噪音。
- npm CLI 大版本升级时（如 npm 11.x），需要验证命令兼容性并更新。
- 有新翻车案例时，优先更新「卡点 & 兜底」「验证清单」「历史教训」。
- CI workflow 模板需要根据实际项目构建工具（tsdown / tsup / tsc）调整 build 命令。

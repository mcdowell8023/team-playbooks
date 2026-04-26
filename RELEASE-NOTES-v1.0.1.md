# v1.0.1 — Dogfooding Patch

基于 v1.0.0 自用反馈（DOGFOODING-v1.0.0.md）的小补丁，主要优化 github-release.md 的 4 个细节。

## 改动

### github-release.md
- **新增 Step 0.5**：从散落文件初始化仓库（针对「无 git 仓库的散落文件」场景）
- **Step 0 优化**：安全扫描的 grep 命令加排除规则与误报处理指南，避免对教程类文档误报
- **Step 1 补充**：git 全局配置（user.name / user.email）前置检查
- **Step 7 补充**：明确 RELEASE-NOTES 文件建议先 commit + push 入库便于追溯

### 不变
- npm-publish.md
- skill-publish.md
- release-readiness-review.md
- playbook-template.md

## 这次发版的元意义

v1.0.1 本身就是 dogfooding 闭环的产物：用 Playbook 自己 → 发现问题 → 改进 Playbook → 再发版。
未来每次重大使用都鼓励产出 DOGFOODING-vX.X.X.md 反馈报告。

## 反馈来源

- DOGFOODING-v1.0.0.md（鲁班，2026-04-26）

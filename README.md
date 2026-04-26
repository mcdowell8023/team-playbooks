# Team Playbooks

万三虚拟团队的可复用工作流手册。每个 Playbook 是「触发场景 → 步骤 → 卡点 → 验证」的执行手册，沉淀自真实项目操作经验。

## Playbook 索引

| Playbook | 用途 | 触发场景 |
|---|---|---|
| [github-release.md](./github-release.md) | GitHub 项目发布 | 推新仓库 / 已有仓库版本发布 |
| [npm-publish.md](./npm-publish.md) | npm 包发布 | scope 包 / 公开包 / 私有包 |
| [skill-publish.md](./skill-publish.md) | Skill 完整发布 | 设计→实装→审查→发版全流程 |
| [release-readiness-review.md](./release-readiness-review.md) | 发版前总闸 | 任何正式发版前必过 |
| [playbook-template.md](./playbook-template.md) | 元模板 | 创建新 Playbook 时套这个壳 |

## 使用方式

1. 找到目标 Playbook
2. 检查「适用场景」是否匹配
3. 准备「前置条件」（凭证 / 工具 / 路径）
4. 按「步骤」执行
5. 遇到问题查「卡点 & 兜底」
6. 完成后过「验证清单」

## 设计哲学

- **如何做（How-to）优先于为什么（Why）**：执行手册不是原理书
- **凭证先查记忆**：永远不要让用户重新登录已经登过的服务
- **历史教训固定段**：每次翻车都沉淀回 Playbook，不让团队重复踩
- **双轨审查**：代码视角（包拯）+ 用户视角（商鞅），单轨审查不可替代

## 贡献

欢迎 Issue / PR。新增 Playbook 请套用 `playbook-template.md` 元结构。

## License

MIT

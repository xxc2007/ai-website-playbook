# ai-website-playbook

**A reusable playbook (skill) for building, deploying, operating, and safely maintaining a self-hosted, AI-powered content website.**

一套可复用的 skill：把一个真实生产站点（单 VPS、TypeScript monorepo + PostgreSQL + systemd + nginx、AI 驱动的内容聚合站）从零到上线、再到长期运维的完整经验，提炼成「任何人 + AI Agent」都能照做成功的操作手册。

## What it covers / 覆盖什么

| 模块 | 内容 |
|---|---|
| 技术选型与代码结构 | monorepo 布局、编号迁移纪律、CI 定义双份同步、测试库命名硬规则 |
| 本地构建与校验 | 起站序列 + 四级验证梯子（typecheck → `_test` 库测试 → web 构建 → 只读冒烟） |
| 变更与发布 | 快进推送、分支保护 strict 语义、CI 抖动处置（确定性锁等待谓词）、同轮 changelog/tag |
| 部署三端同步 | DRY-RUN → `--apply --rev=<merge SHA>`；四端身份核对（本地 / GitHub / 每台 `.release` 40 位 / `health`）；逐台验收与回滚 |
| 后台运维 | systemd 单元拓扑、配置覆盖链（systemd drop-in > `.env`，验收读 `/proc/<pid>/environ`）、维护锁协议、journald 控量、PSI 资源观测 |
| 性能治理（真实案例） | 连接池被 systemd 覆盖压成 2；10 秒级全表聚合 + 57014 超时 + 静默失败；派生表反规范化的写路径收口与外键索引 |
| 安全清理纪律 | 证据化白名单 → 逐条确认 → archive 留底 → 红线清单；可再生/不可再生判别法 |
| 真实浏览器验收 | 桌面/移动/功能/后台四象限清单、后台门禁 curl 实测、**严禁误触生产按钮**纪律 |

## Structure / 结构

```
ai-website-playbook/
├── SKILL.md                              # 主 skill（frontmatter + 正文九大板块）
└── references/
    ├── deploy-runbook.md                 # 部署全流程 runbook（发布→上线→验收→回滚）
    ├── incident-cases.md                 # 故障案例库（18 条：现象/根因/定位/修复/防回归）
    └── security-review-checklist.md      # 安全审查清单（SSRF/SQL绑定/凭据/清理/报告诚信）
```

## Install / 安装使用

把本目录复制（或 clone）到你的 Agent 技能目录，例如：

```bash
git clone https://github.com/xxc2007/ai-website-playbook.git
cp -r ai-website-playbook ~/.zcode/skills/          # 或 ~/.claude/skills/ 等等价位置
```

Agent 加载后按 `SKILL.md` 的章节顺序执行；人也可以直接把 `SKILL.md` 与 `references/` 当作 runbook 照做。所有命令里的 `<占位符>`（`<app>`、`<user>@<host>`、`/opt/<app>` 等）替换为你自己的值。

## Design principles / 设计原则

1. **分级动作协议**：只读探查 → 提案清单 → 逐条确认 → 执行；删除/移动/凭据轮换永远先出清单。
2. **四端身份核对**：发布成败以逐字节核对 + 每台 `.release` 标记 + 线上 health 为准，不凭退出码。
3. **如实报告**：没跑的检查写「未验证」，失败如实报，不假绿、不假推。
4. **读者优先**：读者页面可用性永远优先于回填、工具升级、诊断与清理。

## License

MIT — see [LICENSE](LICENSE).

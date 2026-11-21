---
name: ai-website-playbook
description: >-
  End-to-end playbook for building, configuring, deploying, operating, and safely
  maintaining a self-hosted AI-powered content website (TypeScript monorepo +
  PostgreSQL + systemd + nginx on a single VPS). Use whenever a human or an AI
  agent needs to: scaffold or restructure such a site; run local builds and the
  staged verification ladder; publish via fast-forward pushes and protected-branch
  PRs; deploy with byte-verified three-way sync (local repo = GitHub = per-server
  .release marker = live /api/health); operate systemd units, connection pools,
  timers, and logs; diagnose and fix performance issues (systemd-overridden pool
  sizes, slow full-table aggregates, missing indexes, denormalized derived-table
  write paths); run evidence-based cleanup (whitelist, review-before-delete,
  rollback plans); or do real-browser acceptance (desktop / mobile / admin)
  without ever clicking production mutating buttons. 构建/配置/维护自托管 AI
  内容网站的全流程 playbook：技术选型与代码结构、本地构建与校验、部署三端同步、
  systemd/连接池/定时器/日志运维、用量统计与性能治理（真实根因案例）、安全清理
  纪律、真实浏览器验收。
---

# AI Website Playbook

一套让「人 + AI Agent」都能照做成功的自托管网站全流程操作手册。它把一个真实生产站点（2026 年上线、单 VPS、TypeScript monorepo + PostgreSQL + systemd + nginx、指标表数十万行、多角色多节点）的构建、发布、运维、性能治理与清理纪律，提炼成可复做的通用步骤。

**怎么用本 skill**：

- 正文是主干流程，按章节顺序执行即可；每一步给出可直接复制的命令（`<>` 为占位符，替换成你自己的值）。
- 细化文档在 `references/`：
  - `references/deploy-runbook.md` —— 部署全流程 runbook（前置门 → 发布 → PR → 上线 → 验收 → 回滚）。
  - `references/incident-cases.md` —— 故障案例库（每条：现象 / 根因 / 定位 / 修复 / 防回归）。
  - `references/security-review-checklist.md` —— 安全审查清单（凭据、SQL、SSRF、清理审查、报告诚信）。

**参考架构**（后文所有章节以此为准，替换成你的等价物）：

```
本地源码仓库 (git)  ──快进推送──▶  GitHub (main, 分支保护 + required checks)
        │                                    │ merge SHA
        └── deploy 脚本 (git archive 包) ──▶ 服务器 systemd 单元（nginx 反代）
                                                 │
                                        线上站点 https://example.com/<app>/
```

---

## 0. 安全门与行动纪律（任何动作之前）

这一节是纪律层，优先级高于后面所有技术步骤。

### 0.1 分级动作协议

| 级别 | 动作类型 | 要求 |
|---|---|---|
| L1 只读 | `git status/log/diff`、`systemctl status/list-*`、读配置、`curl` 健康检查、EXPLAIN、日志查询 | 随时可做，产出要带证据 |
| L2 提案 | 任何删除/移动/轮换/清理：**只出清单不执行** | 每条列 target / why / risk / rollback，逐条确认后才动 |
| L3 执行 | 部署、重启服务、迁移、改配置、删文件 | 前置门全过 + 逐条授权 + 当场验收 |
| L4 禁区 | force push 主干、跳过必需检查、`docker compose down -v`、未经 dump 删库、清审计数据制造吞吐 | 不做，无例外 |

- 清理类输出永远先出白名单（见 §7），**站长/用户逐条确认后才执行**；未确认的一律不动。
- 历史文档（审计、事故记录、changelog）**只增不改**；没实际跑过的检查，报告里必须如实写「未验证」，不许写成已通过。
- 维护窗口安排在**低流量且资源充足**时段；读者可用性永远优先于历史回填、工具升级、诊断和清理。不得通过清除未知状态回执、绕过保护机制或盲目提高并发来「制造吞吐」。

### 0.2 密钥纪律

- 密钥只存在于服务器 `.env`（或等价密钥服务）；**绝不进仓库、文档、日志、提交信息、聊天输出**。
- `.env`、`.data/`（运行数据）、密钥文件一律不提交；发布包与备份按 §4.3 排除清单排除。
- 公开仓库 / 公开文档里一律使用占位符（`<user>@<host>`、`https://example.com/<app>`），不出现真实主机名、IP、密钥路径。

### 0.3 如实报告

- 发布成功与否以**逐字节核对 + 每台机器的 `.release` 标记 + 线上健康端点**为准，不凭退出码、不凭「脚本没报错」。
- 推送/发布失败就报失败，写清卡在哪一步、错误原文是什么；**不假推、不假绿**。
- SSH 无法连接、脚本不可达、某节点未盘点——照实说「未能验证」，宁可漏报不猜。

### 0.4 AI Agent 附加纪律

- 只发只读命令，除非对当次会话已有明确书面授权；所有命令静默执行，不弹独立终端窗口。
- 改动前先完成安全扫描（本项目要求扫描完整通过；扫描器报资源不足等「未得出完整结论」时，**不得宣称项目级安全结论**，也不触发真实推送）。
- 真实浏览器验收时严禁误触生产按钮（见 §8.3）。

---

## 1. 技术选型与代码结构

### 1.1 推荐栈

| 层 | 选型 | 理由 |
|---|---|---|
| 语言/运行时 | TypeScript 全栈 + Node.js | 前后端同语言，类型检查可跨端复用 |
| 仓库形态 | npm/pnpm workspaces monorepo | `apps/api`（后端）、`apps/web`（前端）、`packages/backend`（共享领域逻辑）各自可独立构建与测试 |
| 数据库 | PostgreSQL | `GROUPING SETS`/部分索引/触发器/行级锁等治理手段齐全；迁移用编号递增的 SQL 文件 |
| 前端 | React Router（Vite 构建） | 支持 `BASE_PATH` 子路径部署（如 `https://example.com/<app>/`），**BASE_PATH 是构建期变量**，改了不重建不会生效 |
| 部署 | 单 VPS + systemd + nginx | nginx 静态反代 + `nginx -t` 门禁；systemd 管常驻单元与定时器 |
| CI | GitHub Actions，必需检查 strict | `check`（typecheck+tests）与 `docker` 两类，**对管理员同样生效** |

### 1.2 目录结构模板

```
repo/
├── apps/api/                 # HTTP 服务（ Fastify/Hono 均可）
│   └── src/routes/admin.ts   # 后台路由（注意缓存与截止时间）
├── apps/web/                 # 前端（构建产物带 BASE_PATH）
│   └── tests/                # 前端测试（node --test 可跑）
├── packages/backend/         # 共享领域逻辑（db、admin 读侧、快照）
├── database/migrations/      # NNNN_描述.sql，编号递增追加
├── scripts/
│   ├── smoke.ts              # 只读冒烟脚本（--base 可指本地或线上）
│   ├── migrate.ts            # 迁移执行器：每个迁移独立事务
│   └── backfill-*.ts         # 大回填：分批/游标/可续跑（不进迁移）
├── deploy/<app>/
│   ├── deploy.sh             # 默认 DRY-RUN；--apply --rev=<merge SHA>
│   ├── verify-deploy.sh      # 逐项验收清单脚本
│   ├── verify-github-sync.sh # 本地↔GitHub 逐字节核对
│   └── rollback.sh           # 默认 DRY-RUN 回滚计划
├── content/                  # 站点内容数据（changelog.json 等，与代码同轮更新）
├── docs/history/             # 审计/事故/清理文档，只增不改
└── .github/workflows/check.yml
```

### 1.3 结构纪律

- **迁移按编号追加**（`0127_xxx.sql` 接在最大编号后），每个迁移独立事务（`sql.begin`）；迁移失败即中止、不重启服务。注意：迁移单事务里 `CREATE INDEX CONCURRENTLY` 不可用，大索引构建要锁写表，必须选低流量窗口并如实声明代价。
- **CI 定义双份逐字节同步**：仓库内 `.github/workflows/check.yml` 与 `tooling/ci-check.yml`（或等价镜像）内容必须完全一致，用 `diff` 当场核对，防两处漂移。
- **测试库命名硬规则**：测试套件启动时校验 `DATABASE_URL` 库名必须以 `_test` 或 `_ci` 结尾，否则直接拒绝运行——防误连开发/生产库。
- 后台统计类读接口统一走「**快照 + 缓存 + 截止时间**」三件套（见 §6.2）。

---

## 2. 本地构建与校验

### 2.1 本地起站序列

```bash
npm run env:init                          # 生成 .env（密钥只落本地）
npm run db:up -- --daemon                 # 起本地 PostgreSQL（Docker 或本机）
DATABASE_URL=postgres://…/<app>_dev npm run db:migrate
npm run build -w @<scope>/web             # BASE_PATH=/app 是构建期变量
BASE_PATH=/app npm run dev                # 本地预览
```

判据：`schema_migrations` 表行数 = `database/migrations/` 文件数（**编号跳号不算漏跑**，只看已应用行数对上即可）；`/api/health` 返回 `ok:true, db:ok`。

### 2.2 验证梯子（逐级失败即停，禁止跳级）

```bash
# 第 1 级：类型检查（全部 tsconfig 工程 + web）
npm run typecheck

# 第 2 级：数据库测试（先建专用库；库名必须 _test/_ci 结尾）
createdb <app>_test
DATABASE_URL=postgres://…/<app>_test npm run db:migrate && npm test

# 第 3 级：前端构建 + 前端测试
npm run build -w @<scope>/web
node --test apps/web/tests/*.test.ts

# 第 4 级：只读冒烟（对本地或线上都安全）
node scripts/smoke.ts --base https://example.com/<app>
```

- 冒烟脚本必须是**只读**的（GET 校验、内容断言），对生产零副作用，因此可在部署后直接对线上跑。
- 改了 prompt/锚点类内容文件后，补跑对应的 lint 与 anchors 校验命令。
- 每级失败先修本级，不许带着红基线往下走，也不许把红基线带进后续 PR 的必需检查。

---

## 3. 变更与发布流程（GitHub 侧）

### 3.1 分支与 PR 规则

- 一切变更走分支 + PR；**推送一律快进（fast-forward），禁止 force push，禁止改写已上线历史**。
- `main` 分支保护：PR 必须先与 `origin/main` 同步，并通过 required checks（`check`、`docker`，strict 语义）+ CodeQL；**管理员同样受规则约束**，禁止 `--admin`、跳检查、force push main。
- 合并后的 **merge SHA 就是发布对象**：部署、tag、changelog 都锚定它。

### 3.2 CI 抖动处置路径

时序敏感的并发观察测试在 CI 负载下会抖动（真实案例：5 秒观察窗的锁等待测试，同一树哈希一次绿一次红）。处置顺序：

1. 先 `gh run rerun <run-id> --failed` 重跑失败 job（只重跑同一提交，不动任何代码）；转绿即恢复基线，到此为止。
2. 仍红则**修测试稳定性**而非跳过检查：加长观察窗 deadline；把依赖查询文本匹配的谓词（如 `query LIKE '%FOR KEY SHARE%'`）换成确定性阻塞判定：

   ```sql
   -- 确定性判"这个连接正被阻塞"：不依赖等待事件文本
   datname = current_database()
     AND pid <> pg_backend_pid()
     AND cardinality(pg_blocking_pids(pid)) > 0
   ```
3. 超时抛错时附上当时 `pg_stat_activity` 快照（pid / wait_event_type / query 前 80 字符），便于下次定位。
4. 按正常 PR 流程过 check + docker 后合并；**任何情况下不跳过必需检查**。

### 3.3 同轮 changelog 与版本 tag

- 每个可感知变更**同轮**更新 `content/changelog.json`（从用户/管理员视角描述结果，不写组件名与环境变量）。
- 发版时打附注 tag（`v<语义版本>`）锚定 merge SHA；紧急修复当轮也要回收为源码 + PR，不留线上手改。

---

## 4. 部署三端同步

> 细节全流程见 `references/deploy-runbook.md`，此处是主干。

### 4.1 四端身份核对（发布前后都要做）

发布是否成功以四端一致为准，**不能用本地 Git HEAD 代替服务器侧证据**：

| 端 | 核对方式 |
|---|---|
| 本地源码 | `git rev-parse HEAD` |
| GitHub | `git ls-remote origin main` |
| 服务器部署标记 | 逐台 `sudo cat /opt/<app>/.release` —— 必须读**完整 40 位 SHA** |
| 线上行为 | `curl https://example.com/<app>/api/health` 的 `ok/db/release` |

### 4.2 部署流程

```bash
# 1) 演练（默认 DRY-RUN，核对 rev、包成员数、排除路径，不碰线上）
bash deploy/<app>/deploy.sh --web-prebuilt

# 2) 正式上线（rev = 该轮 merge SHA；SSH 目标经环境变量注入，不写死在脚本里）
DEPLOY_SSH_TARGET=<user>@<host> DEPLOY_SSH_KEY=<key路径> \
  bash deploy/<app>/deploy.sh --apply --web-prebuilt --rev=<merge SHA>

# 3) 验收（不凭退出码）
bash deploy/<app>/verify-deploy.sh
bash deploy/<app>/verify-github-sync.sh          # 本地↔GitHub 逐字节
node scripts/smoke.ts --base https://example.com/<app>
curl -s https://example.com/<app>/api/health     # ok:true / db:ok / release=merge SHA
```

### 4.3 部署脚本必备护栏（自己写部署脚本时逐条对照）

- **默认 DRY-RUN**，`--apply` 才动手；`--rev` 显式指定发布对象。
- 传输用 `git archive` 打包上传解包，排除 `node_modules/ .data/ .env .git`。
- 大内存机器跳过远端构建：`--web-prebuilt` 让 web 在本地构建后上传（BASE_PATH 构建期注入）。
- 开工先**暂停归档类定时器 + 原子取得维护锁**（`O_CREAT|O_EXCL`，见 §5.3）。
- 代码快照回滚点：改动前整树快照，构建/验收失败自动还原。
- **迁移失败即中止、不重启服务**（迁移门在重启门之前）。
- `nginx` 配置变更先 `nginx -t`，通过后才 `systemctl reload nginx`；不过即还原不 reload。
- **角色化节点跳过无关单元**：如 readers-only 节点自动跳过 worker 重启；常驻单元清单集中在一处定义。
- 重启后对读者面做预热（打一轮首页/聚合接口）再收尾。

### 4.4 回滚预案

- 脚本内置：构建失败自动整树还原快照；验收不过自动还原 `.env` 备份并打印回滚命令。
- 独立回滚脚本默认 DRY-RUN 打印计划，确认后 `--apply`。
- 数据库默认保留；**真要删库回滚前强制 `pg_dump`**；禁止 `docker compose down -v`（卷里是数据库、上传件与证书）。
- 多 PR 迭代时**每轮独立部署窗口**，不把两轮改动混进同一次上线。

---

## 5. 后台运维（systemd / 连接池 / 定时器 / 日志）

### 5.1 常驻单元拓扑

- 单元清单集中在安装脚本一处定义；不同角色节点（全量 worker 节点 / readers-only 节点）装不同子集。
- **disabled 单元 ≠ 无效资产**：按角色休眠的单元（如备用网关、分时段 timer）是设计的一部分，勿清理。判断是否可清理必须查它在角色定义里的位置，而不是只看 `enabled` 状态。

### 5.2 连接池与配置覆盖链

配置生效链（高 → 低）：**systemd drop-in 的 `EnvironmentFile` > 单元 `Environment=` > 应用 `.env`**。改了 `.env` 没生效的最常见根因就是被 systemd 覆盖（真实案例见 §6.1）。

验收必须读**进程实际环境**，不能只看文件：

```bash
# 找到主进程 PID
systemctl show <app>-api --property=MainPID
# 读进程环境（只读）
sudo tr '\0' '\n' < /proc/<MainPID>/environ | grep -E 'DATABASE_POOL_MAX|PGBOSS_POOL_MAX'
```

### 5.3 定时器与维护锁

```bash
systemctl list-timers --all          # 巡检所有 timer
systemctl cat <app>-archive.timer    # 看定义
```

- 部署窗口先 `systemctl stop` 归档类 timer，收尾恢复；恢复后确认 `active + enabled`。
- **维护锁协议**：以 `O_CREAT | O_EXCL` 原子创建锁文件（如 `/var/lib/<app>-maintenance/owner.json`，内容含 owner 与时间）。拿到锁才可维护；发现锁属于别人、或无法确认排空，**立即停止维护**。
- **禁止**按时间推算、SSH 断连、PID 消失、数据库无连接等间接信号推断「对方已结束」并删锁。

### 5.4 日志治理

```bash
journalctl --disk-usage                                   # 现状
sudo journalctl --vacuum-size=500M                        # 控量（不手工删日志文件）
sudo journalctl -u <app>-api --since today | grep -c 57014 # 超时计数
```

- 日志**不可再生**，只控量不手删。
- **静默失败是诊断缺口**：审查每个快照/后台刷新的失败分支——失败必须落一条结构化 warn 日志（含错误码），否则故障期间服务端零痕迹（真实案例见 §6.2）。

### 5.5 nginx 纪律

```bash
sudo nginx -t && sudo systemctl reload nginx   # 先测后重载，失败即还原不 reload
```

### 5.6 资源压力观测（PSI）

```bash
cat /proc/pressure/io      # full avg60 < 40 为健康线
cat /proc/pressure/memory
df -h /
```

- 后台重聚合/快照刷新前可设**资源门槛**（IO/内存/磁盘余量），不满足即跳过本轮（自愈，只是延迟）；门槛值与 `.env`/文档口径保持一致。
- 磁盘上可再生缓存（图片代理、OG 卡、Playwright 浏览器、npm/node-gyp 缓存）与不可再生数据（日志、归档、备份）分开台账，见 §7.5。

---

## 6. 用量统计与性能治理（真实案例）

方法论先行：**EXPLAIN 复现要用应用同款会话设置**（`statement_timeout`、`work_mem`、`jit off`、`enable_sort` 等），否则耗时与计划都不可信。跑法：

```sql
SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY;
SET statement_timeout = '35s';
SET work_mem = '16MB';
SET jit = off;
SET enable_sort = off;
EXPLAIN ANALYZE <生产同款SQL>;
```

### 6.1 案例 A：连接池被 systemd EnvironmentFile 覆盖压成 2（根因已修复）

- **现象**：读者页聚合总览刷新不出来、点任意日期卡很久；`/api/health` 连续 `query_timeout`；后台某页 40 秒才出。
- **根因**：应用读侧按 `DATABASE_POOL_MAX` 建池（`packages/backend/src/db.ts` 的 `poolSize(...)`），但 systemd drop-in 的 `EnvironmentFile /etc/<app>/pools.env` 里 `DATABASE_POOL_MAX=2` **覆盖了 `.env` 的新值**——读者面 4 个聚合槽抢 2 条连接，全在排队。
- **定位**：改配置文件 → 确认文件 mtime 早于服务重启时间 → 读 `/proc/<api-pid>/environ` 证实进程内 `DATABASE_POOL_MAX=8` 已生效 → `journalctl` 里 4 次重启退出记录对上时间线。
- **修复**：改 `/etc/<app>/pools.env`（数据库池 ≥8、队列池 ≥3）→ 重启 api 单元 → **再次逐台读回 `/proc/<pid>/environ` 验收**。
- **教训**：「改了配置为什么没生效」九成出在覆盖链（§5.2）；验收口径永远是进程环境，不是文件内容。

### 6.2 案例 B：无时间下界的全表聚合，10 秒级超时 57014 + 静默失败

- **现象**：后台用量总览永远停在「正在生成统计」；`journalctl` 里 usage 相关日志恒为 0，但同日 `57014`（statement timeout）出现 19 次。
- **根因**：总览快照是一条**无时间下界**的 `GROUPING SETS` 全表聚合（Seq Scan 48.8 万行 metrics + 38.9 万行标签表，空载 10.1 秒）；IO 受压时超过 30 秒 `statement_timeout` → 快照建不出来；且**刷新失败分支只设 30 秒冷却、不打任何日志**——排查时服务端零痕迹。逐日点击是另一个坑：快照键含日期、盘上缓存 24 小时过期，每个日期每天首开都要现场跑 SQL，而 `LEFT JOIN` 标签表仍整表 Seq Scan（1.9 秒）。
- **止血（不改口径）**：
  1. 快照 `freshMs` 30 秒 → 300 秒（对齐 runs 口径），HTTP 层已有 60 秒 MicroCache 兜底展示，把刷新风暴降 10 倍；
  2. 失败分支补结构化 warn 日志（含 code），对齐有日志的同类 reader；
  3. 读接口补 stale fallback：`{staleIfErrorMs: 600_000, canUseStale: e => ['57014','55P03'].includes(e?.code ?? ''), markStale: v => v}`——重聚合窗口内超时变成「先给上次值、后台重算」，而不是 500。
- **根治（改口径需发版）**：给聚合加时间下界走部分索引；或仿照既有 trigger 维护模式建**按天×模型×服务×计费类型**的预聚合派生表，让五档读变成小表聚合。
- **教训**：`statement_timeout` 超时码（57014/55P03）要纳入 stale 可用集；「引用既有契约前必须 grep 证实实现存在」——注释说前端会显示提示，不代表实现了（本项目实测该 if 块为空、字段已被剥除）。

### 6.3 案例 C：派生表反规范化的写路径收口

同一事实表存在两条写路径（触发器维护 + 回填脚本直写）时：

- **传播触发器必须建在派生表本体上**（`AFTER INSERT OR UPDATE ON 派生表`），单触发器收口全部写路径——只给上游表加触发器覆盖不了直写回填路径，会出现「回填补的数不传播」的静默口径错误。
- **反向 UPDATE（按外键回填派生行）必须有外键索引**：否则每次上游写入都对几十万行做同步 Seq Scan，等于每条回执写入被拖慢秒级并持行锁，管道吞吐崩塌。注意迁移单事务内 `CREATE INDEX CONCURRENTLY` 不可用，普通构建锁写表分钟级，必须低流量窗口执行并如实声明。
- **大回填不进迁移**，走独立脚本：按主键游标分批（批上限 1000）、每批独立事务 `SET LOCAL statement_timeout='5s'; SET LOCAL lock_timeout='250ms'`、批间 pause、进度文件可续跑（中断后 `--after=<游标>` 继续）、结束全台账 gap-check + 就绪开关写入。
- `ON CONFLICT DO UPDATE` 值未变也计为行更新、会触发传播触发器；带外键索引后是幂等空转，无害。

### 6.4 慢查询可观测性分级

| 手段 | 风险 | 步骤 |
|---|---|---|
| 开慢查询日志 | 低（只 reload） | `ALTER SYSTEM SET log_min_duration_statement='2s'; SELECT pg_reload_conf();`（注意 journal 增量） |
| 装 pg_stat_statements | 高（要重启 PostgreSQL，读者面中断一次） | `shared_preload_libraries` + `CREATE EXTENSION`；必须走维护锁协议 + 低流量窗口 + 逐条确认 |

装上之后「哪条聚合最慢」可直接从库里排名，不必再靠 EXPLAIN 复现。

---

## 7. 安全清理纪律

### 7.1 只出清单，逐条确认

清理产出是**候选清单**，每条带四元组；删除、移动、凭据轮换都要站长/用户逐条确认后才执行。风险评级与既有治理文档对齐，**不私自降级**（治理文档判 medium 的不许在自己报告里写成 safe）。

```markdown
- target: repo: scripts/xxx.ts        # 或 server: /opt/<app>/xxx
  why: 零功能引用（git grep 仅 3 个自命中 + 7 个 docs/history 审计文档命中，后者是历史文档非引用）
  risk: safe | risky
  rollback: git 历史可 revert / 先 tar 到 /opt/<app>/archive/ 再删
```

### 7.2 候选必须证据化

- 零引用要用 `git grep -ln "<名字>"` 证明，并**区分历史文档命中与功能引用**（`docs/history/` 命中不算引用）。
- 服务器侧要实测：`sudo ss -ltnp` 看监听、`/etc/systemd/system` 与 cron 目录按名字 grep、目录 mtime 与事件日期对账。
- 与仓库跟踪面 diff 得出的未跟踪散落件，先确认不在任何运行链路再列候选。
- 只有主人知道的「是否仍有人工调用」必须标 `requiresManual`，只提案不判定。

### 7.3 archive 语义

「归档」= **移出活动树 + 外部留一份 + git 历史留一份**；不是 `git mv` 到仓库内的归档目录（那只是换个地方堆着）。

### 7.4 红线清单

- 禁止 `docker compose down -v`（卷里是数据库、上传截图与证书）。
- 回滚删库前强制 `pg_dump`。
- 不把旧共享工作树整体覆盖远端或生产；不夹带无关未提交改动。
- **凭据未轮换不删 vault/备份目录**（里面可能还有可用形态的密钥；轮换是删除的前置条件）。
- env 备份不放在代码目录里（放 `$APP_HOME` 上层或专用配置目录）。

### 7.5 可再生 / 不可再生判别法

| 可再生（可清） | 不可再生（只控量/归档） |
|---|---|
| 图片代理缓存、OG 卡缓存（服务自动重建） | journald 日志（只 `--vacuum-size` 控量） |
| Playwright 浏览器缓存、npm/node-gyp 缓存 | 一次性修复的原始备份落盘（先移档归档） |
| 钩子基线快照（会话结束后按策略清） | 退役目录里的旧凭据（先核对轮换） |

缓存清刷会打慢盘与读者面，按 mtime 分龄、低流量窗口执行。

### 7.6 宁漏不猜

没有可用入口的节点（如备用协调节点）如实写「未盘点」；涉及密钥面的文件没读就不列为候选——**宁可漏报，不误删**。

---

## 8. 真实浏览器验收

### 8.1 四象限清单（每次发布后跑一轮，记录耗时基线）

| 象限 | 检查项 |
|---|---|
| 桌面读者面 | 首页 200 且热缓存首屏 <3s；主要栏目页逐个 200；卡片→详情页→原文链接可达；正文无后台样式污染 |
| 移动端（视口 ≤430px） | 导航不溢出、卡片单列可读、图表可横向浏览、无横向滚动 |
| 读者功能回归 | 搜索、筛选、RSS、changelog 页可用；最新条目与内容台账一致 |
| 后台 | 登录后各页正常出数；角标有值；重聚合窗口内 journal 无 500；快照状态徽章是 ready/asOf 而非永久 generating |

### 8.2 后台安全门（curl 实测，不靠感觉）

```bash
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' https://example.com/<app>/admin
# 期望：302 → /admin/login
curl -s -o /dev/null -w '%{http_code}\n' https://example.com/<app>/api/admin/nav-counts
# 期望：401
```

### 8.3 严禁误触生产按钮

- 浏览器验收**一律只读导航**：看页面、点链接、跑查询；**不点任何「重建 / 清理 / 删除 / 发布 / 回填 / 重置」类按钮**。
- 后台数据的变更动作改为服务端只读观测（读 `.release`、读进程环境、读 journal），需要验证写路径时在 `_test` 环境做。
- 用 Playwright MCP 或无头浏览器时，先列快照再交互；对含按钮的后台页，交互前逐个核对按钮语义，拿不准就不点。
- 截图留档到工作区（不进仓库），作为验收证据。

### 8.4 性能基线记录法

每轮验收记录：首屏耗时、点击到出数耗时（首开含生成态轮询 ≤3s 为线）、连续点 5 个不同日期无一次卡 >5s、开着页面 5 分钟 journal 无大聚合连发。基线写进验收记录，供下轮对比。

---

## 9. 如实报告与升级路径

- **needsUser 信号**：需要人逐条确认的服务器动作、清理项、凭据轮换——列成清单交给 owner，不代签。
- 维护锁冲突即停；资源不满足即延期；扫描未完整通过不触发真实推送。
- 报告里「未验证」三字是合格报告的一部分：没跑的检查、连不上的节点、读不到的密钥面，都如实标注并写明原因。
- 部署/发布类结果按 §4.1 四端身份核对口径给结论，不凭脚本退出码。

---

## 参考文档

| 文档 | 内容 |
|---|---|
| `references/deploy-runbook.md` | 部署全流程：前置门、四端核对、验证梯子、发布、PR、上线、逐台验收、迁移专项、24h 观测、回滚、SSH 通道注意事项 |
| `references/incident-cases.md` | 故障案例库：整站 404（BASE_PATH）、跑旧码、闸门误伤、NAT 死链、IO 假红、连接池覆盖、慢聚合超时、静默失败、写路径不传播、CI 抖动、部署脚本自身 bug、SSH 停滞等 |
| `references/security-review-checklist.md` | 安全审查清单：生成前扫描、SSRF、SQL 绑定、凭据、仓库卫生、后台门禁、维护锁、清理审查、审计文档纪律 |

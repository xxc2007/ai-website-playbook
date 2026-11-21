# Deploy Runbook（部署全流程）

> 本 runbook 把「本地源码 → GitHub → 服务器部署 → 线上行为」三端/四端同步的完整流程写成可照做的步骤。占位符：`<app>`（应用名）、`<scope>`（npm workspace 前缀）、`<user>@<host>`（SSH 目标）、`/opt/<app>`（服务器应用目录）、`https://example.com/<app>`（站点地址）。所有命令静默执行，不弹独立终端窗口。

## 0. 前置门（全部满足才动手）

- [ ] 本机安全扫描完整通过（扫描器报「资源不足/未得出完整结论」时不得宣称安全、不得触发真实推送）。
- [ ] 站长/用户已逐条确认本轮涉及的服务器动作与清理项。
- [ ] 低流量且资源充足窗口（读者可用性优先于一切维护；`/proc/pressure/io` 的 `full avg60 < 40` 为健康线）。
- [ ] 以 `O_CREAT | O_EXCL` 原子取得维护锁；发现非自身 owner 或无法确认排空即停止，**不得**按时间、SSH 断连、PID、数据库无连接推断排空并删锁。
- [ ] 本轮 merge SHA 已确定（required checks 全绿的合并提交）。
- [ ] SSH 通道就绪：目标、密钥、已知主机文件可读（Windows 中文用户名的坑见 §11）。

## 1. 四端身份核对（部署前摸底 + 部署后终检，同一组命令）

```bash
# 本地
git rev-parse HEAD                      # 本地 HEAD
git status -sb                          # 必须是 clean、与 origin/main 零偏离

# GitHub
git ls-remote origin main               # 远端 main SHA

# 服务器（逐台；readers-only 节点也要）
ssh <user>@<host> 'sudo cat /opt/<app>/.release'
# 必须读完整 40 位 SHA，不能用本地 Git HEAD 代替

# 线上行为
curl -s https://example.com/<app>/api/health
# 期望 {"ok":true,"db":"ok","release":"<40位SHA>"}
```

结论口径：本地 HEAD = GitHub main = 每台 `.release` = `health.release`，四者一致才算同步。

## 2. 本地验证梯子（全套，逐级失败即停）

```bash
npm run typecheck                                     # 全部 tsc 工程 + web

createdb <app>_test                                   # 测试库名必须 _test/_ci 结尾
DATABASE_URL=postgres://…/<app>_test npm run db:migrate && npm test

npm run build -w @<scope>/web                         # BASE_PATH=/app 构建期变量
node --test apps/web/tests/*.test.ts

node scripts/smoke.ts --base https://example.com/<app>  # 只读冒烟，可对线上跑
```

改了 prompt/锚点类内容文件，补跑 `npm run brain -- --lint` 与 `--anchors`（或等价校验）。

## 3. 发布到 GitHub

```bash
# 演练：只打印将发布的差异，不推送
bash deploy/<app>/publish-to-github.sh --dry-run -m "<变更说明>"

# 核对差异后正式执行（快进推送 + 当场验收）
bash deploy/<app>/publish-to-github.sh -m "<变更说明>"

# 独立逐字节核对（本地 ↔ GitHub）
bash deploy/<app>/verify-github-sync.sh
```

护栏：发布是**快进推送**，不 force、不改写线上历史；`main` 受分支保护。若发布脚本带内网字面量护栏（拒绝源码中出现私有 IP），测试里要用 RFC 5737 文档段（192.0.2.0/24、198.51.100.0/24）替代 `10.0.0.0/8` 等——护栏拒绝既有提交时，先出 PR 改测试字面量或加排除，再走发布。

## 4. PR 与必需检查

- 分支命名带日期与主题（如 `fix/usage-snapshot-20261121`）。
- PR 必须先与 `origin/main` 同步，通过 required checks（`check`、`docker`，strict，**对管理员同样生效**）+ CodeQL。
- 禁止 `--admin`、跳过检查、force push main。
- CI 抖动：`gh run rerun <run-id> --failed`；仍红则修测试稳定性（放宽时序窗、确定性谓词），按 PR 流程合并（详见 SKILL.md §3.2）。
- 合并后的 **merge SHA 是发布对象**。

## 5. 同轮台账

- `content/changelog.json` 新条目（管理员可感知口径：说结果，不写组件名、环境变量、file:line）。
- 附注 tag `v<版本号>` 锚定 merge SHA，GitHub Release 同步建。
- 紧急修复当轮回收为源码 + PR；提交台账（docs 里的执行记录）同轮补。

## 6. 服务器部署

```bash
# 演练（默认 DRY-RUN）：核对 rev、tree、包成员数、排除路径
bash deploy/<app>/deploy.sh --web-prebuilt

# 正式上线（SSH 目标经环境变量注入，不写死在脚本里）
DEPLOY_SSH_TARGET=<user>@<host> DEPLOY_SSH_KEY=<key路径> \
  bash deploy/<app>/deploy.sh --apply --web-prebuilt --rev=<merge SHA>
```

脚本内建语义（自写部署脚本时逐条对照）：

1. 停归档类 timer + 拿维护锁（见 §0）。
2. 代码快照回滚点（改动前整树快照）。
3. `git archive` 包上传解包，排除 `node_modules/ .data/ .env .git`。
4. 迁移门在重启门之前：**迁移失败即中止、不重启服务**。
5. `wait_for_quiet` 门槛：等读者面安静再动（重迁移选此窗口）。
6. 重启常驻单元（清单集中定义）；readers-only 节点自动跳过 worker 重启。
7. 读者面预热（打一轮首页/聚合接口）。
8. `nginx -t` 不过即还原配置、不 reload；过了才 `systemctl reload nginx`。
9. 验收不过自动还原 `.env` 备份并打印回滚命令。

`--web-prebuilt` 说明：web 在本地构建（`BASE_PATH=/app` 注入）后上传，避免大内存机器跑 vite build。

## 7. 迁移专项

- 新迁移按编号追加到 `database/migrations/`；每个迁移独立事务。
- `CREATE INDEX CONCURRENTLY` 在单事务内不可用：大索引构建（数十万行）锁写表分钟级——放在 `wait_for_quiet` 后的低流量窗口，并在 PR 描述里如实声明代价；迁移失败部署脚本会安全中止不重启。
- 迁移不做大回填：几十万行单 UPDATE 持锁数分钟 IO。回填走独立脚本（分批/游标/每批 statement_timeout，见 SKILL.md §6.3）。
- 迁移落盘验收：`node scripts/migrate.ts` 或部署日志显示 `database is up to date`；`schema_migrations` 行数含新迁移。

## 8. 逐台验收清单（不凭退出码）

- [ ] `bash deploy/<app>/verify-deploy.sh` 全绿（判红时逐条核对是否既有漂移：diff 两棵树的相关配置，零变化即为既有项，如实记录另行处置）。
- [ ] `bash deploy/<app>/verify-github-sync.sh` 逐字节一致。
- [ ] `node scripts/smoke.ts --base https://example.com/<app>` 全绿。
- [ ] `curl -s https://example.com/<app>/api/health` → `ok:true, db:ok, release=<merge SHA>`。
- [ ] 逐台读 `.release` 完整 40 位 SHA = merge SHA。
- [ ] 迁移 applied 确认。
- [ ] 常驻单元 active；角色化跳过单元按预期 skipped；归档 timer `active+enabled` 还原。
- [ ] 未登录 `/admin` → 302 `/admin/login`；`/api/admin/*` → 401（curl 实测）。
- [ ] 真实浏览器四象限清单（桌面/移动/功能回归/后台），记录耗时基线（见 SKILL.md §8）。

## 9. 部署后 24 小时观测

```bash
# 关键失败是否落盘（应只有预期内计数）
sudo journalctl -u <app>-api --since "24 hours ago" | grep -c "admin snapshot refresh failed"
sudo journalctl -u <app>-api --since "24 hours ago" | grep -c 57014

# 资源压力
cat /proc/pressure/io        # full avg60 < 40
cat /proc/pressure/memory

# 读缓存头是否在工作（如实现了 x-admin-cache）
curl -sI https://example.com/<app>/api/admin/usage | grep -i x-admin-cache
```

只有新变更或实际失败才增加复测。

## 10. 回滚预案

- 脚本内置：构建失败自动整树还原快照；验收不过自动还原 `.env` 备份并打印回滚命令。
- 独立回滚：`bash deploy/<app>/rollback.sh`（默认 DRY-RUN 打印计划）→ 确认后 `--apply`。
- 数据库默认保留；真删库回滚前强制 `pg_dump`；**禁止 `docker compose down -v`**。
- 代码回滚是「整树还原快照」，不是 `git revert` 线上历史；历史只进不改。

## 11. SSH 通道注意事项

- **Windows 中文用户名的坑**：默认 ssh 读不到 `~/.ssh/config`（路径编码报错如 `/c/Users/\300\355...`）。必须显式指定 ASCII 路径：
  ```bash
  ssh -F <ASCII路径>/ssh-config -o UserKnownHostsFile=<ASCII路径>/known_hosts \
      -i <ASCII路径>/<key> <user>@<host> '<只读命令>'
  ```
  密钥、config、known_hosts 建议统一放 ASCII 路径（如 `D:\ssh\`）。
- **连发停滞**：服务器 `MaxStartups` + fail2ban 在持续扫描噪声下会令连接 banner 超时（实测可达 ~120s），直连与堡垒链路都可能触发。对策：部署脚本内置护栏重试循环（指数退避 + 次数上限），或改走低频直连路线；这是服务器侧整改项（调 `MaxStartups`、清 fail2ban 白名单），部署前与 owner 确认。
- 部署通道环境变量与脚本解耦（`DEPLOY_SSH_TARGET` / `DEPLOY_SSH_KEY`），远端一律以最小必要权限跑；脚本里不出现主机名/IP 字面量。

## 12. 多轮部署纪律

- 多 PR 迭代时**每轮独立部署窗口**：分支独立、验证梯子全套、PR 独立、部署独立、验收独立。
- 先让主干基线转绿，再开下一个 PR，避免把红基线带进后续必需检查判定。
- 服务器侧动作（慢查询日志、扩展安装、缓存清刷、日志控量、回填脚本）一律单独立项：先提案 → 逐条确认 → 部署验收之后执行，避免与上线互相干扰。

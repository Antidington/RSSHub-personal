# 本 fork 的自动同步与 Vercel 部署

配置核对日期：2026-11-03。工作流位于 `.github/workflows/personal-sync-deploy.yml`。

## 工作方式

- 每 6 小时将 `DIYgod/RSSHub` 的 `master` 合并至 `Antidington/RSSHub-personal` 的 `master`，保留上游 commit 历史和本 fork 的独立修改。跟踪开发分支最新提交，不以 release/tag 为更新条件，也不复制上游 tags 或 GitHub Releases。
- 有新提交且推送成功后，调用 Vercel Deploy Hook。没有新提交时不部署；手动运行时可以勾选 `redeploy` 重新部署。
- 合并冲突或推送失败时停止，不强制覆盖远端，不请求部署。需要先处理冲突或重新运行同步。
- 每次运行都核对目标 fork commit 的 Vercel 状态，再检查生产首页、`/healthz` 和 `/rsshub/routes?limit=3`。RSS 验收要求能解析出 3 条内容。没有新提交时也核对部署 SHA，避免旧站点可用掩盖目标版本部署失败。
- 手动勾选 `redeploy` 且 commit 未变时，工作流要求 Vercel 状态在本次 Hook 请求后更新，避免把旧的成功记录当作本次部署完成。等待超时或检查失败会使本次运行失败。
- 本仓库不单独创建常规 Dependabot 版本升级 PR；依赖升级随上游同步进入。三个生态的 `open-pull-requests-limit` 均为 `0`，不关闭 Dependabot 安全更新。
- `vercel.json` 排除 `dependabot/**` 分支的自动部署；`master` 部署和 Deploy Hook 保留。Vercel 项目的 Pull Request Comments 已关闭，上游 `github.silent: true` 配置也保留。

## 一次性配置

工作流绑定 GitHub 的 `Production` 环境。两个 Secret 当前配置在 Settings → Environments → Production → Environment secrets；也支持同名仓库 Actions secrets，环境内同名值优先。

本项目的 Vercel **Production** 环境已设置 `NODE_OPTIONS=--experimental-require-module`。新版依赖 `sanitize-html` 需要通过 CommonJS 加载 ESM 依赖；Vercel 默认禁用该能力时，构建虽可达到 Ready，函数仍会以 `ERR_REQUIRE_ESM` 启动失败并返回 HTTP 500。迁移项目时需一并配置此参数，修改后重新部署；若已有 `NODE_OPTIONS`，保留其他所需参数。参见 [Vercel Node.js 运行时说明](https://vercel.com/docs/functions/runtimes/node-js/advanced-node-configuration)。

1. 将此工作流和本文提交、推送到本 fork 的 `master`，确认 GitHub 默认分支也是 `master`，并在 Actions 页面启用此 fork 的工作流。
2. 创建仅授权 `Antidington/RSSHub-personal` 的 fine-grained GitHub PAT：仓库权限 **Contents: Read and write**、**Workflows: Read and write**。上游会修改 `.github/workflows/`，因此仅使用普通 `GITHUB_TOKEN` 的 contents 写权限不足以处理这类更新。将 PAT 保存为仓库 Actions secret `UPSTREAM_SYNC_TOKEN`，不要写入文件或日志。令牌用户需要有仓库写权限，令牌需要在有效期内；分支保护规则也必须允许该用户推送，否则同步会安全失败。
3. 在 Vercel 导入或连接 **`Antidington/RSSHub-personal`**，生产分支设为 **`master`**，根目录设为仓库根目录。在 Project Settings → Git → Deploy Hooks 中创建绑定 `master` 的 Hook，将完整 URL 保存为 GitHub Actions secret **`VERCEL_DEPLOY_HOOK`**。此 URL 本身就是凭据。
4. 在 Vercel 中配置需要的 RSSHub 环境变量。同步最新源码后，沿用上游 `vercel.json`、`package.json` 和锁文件的构建设置，清除旧项目中不适用的 Install/Build/Output 覆盖值；Node.js 版本需满足同步后 `package.json` 的 `engines`。不要把旧版 `api/vercel.js` 或旧构建命令手工固定到新版上。
5. 检查是否安装过 Pull GitHub App。本仓库遗留 `.github/pull.yml` 是该 App 的配置；如果它仍在管理本 fork，请在 App 设置中停用此仓库的同步，避免两个同步器竞争。
6. 在 Actions → **Sync upstream and deploy to Vercel** → Run workflow 中选择 `master`，首次运行勾选 `redeploy`。确认同步步骤完成后，再到 Vercel 检查目标 commit 的部署状态。

Vercel 的 Git 集成也可能在 PAT 推送后自动触发部署，因此可能同时出现 Git 部署和 Hook 部署。Hook 用于显式请求部署以及无提交时重新部署。不要设置 `github.enabled: false` 来消除重复部署：它也会阻止 Deploy Hook。部署受账户配额和计费规则约束；本流程没有开通付费服务或变更套餐。

## 验收与日常操作

1. 在 Actions summary 确认 upstream SHA 和 fork SHA；合并可能生成额外 merge commit，因此两个 SHA 不必相同。
2. 在 Vercel Deployments 中确认本次目标提交达到 **Ready**，并确认为生产部署。
3. 检查 Actions 的 **Verify production deployment and RSS** 步骤。确认 Vercel 部署状态、健康响应 `ok` 和 3 条 RSS 内容均通过。你实际使用的其他订阅路由仍需分别验收。

Vercel 按请求运行函数，不需要另行执行 `npm start` 或设置常驻保活任务。重新部署会构建并发布新版本，不代表所有 RSSHub 路由都能在 Serverless 环境运行。

首次同步会跨越此 fork 自 2023 年以来的上游变化，包含运行时、依赖、配置和许可证变化。自动合并成功不等于运行兼容；应核对同步后的上游说明、LICENSE、构建日志及实际订阅路由。此配置不执行 RSSHub 全量测试，也不保证未来任一上游提交都可部署。

## 故障处理

### 保留 fork 策略，避免反复合并冲突

2026-10-09 至 2026-11-03，个人修改过的 `pr-review.yml` 与上游冲突，导致连续 86 次同步失败。恢复时将共享的 `pr-review.yml`、`dependabot-fork.yml` 和 `semgrep.yml` 还原为上游内容。个人安全扫描改用独立的 `.github/workflows/personal-security.yml`。

`.github/dependabot.yml` 是本部署仓库拥有的策略文件。`.gitattributes` 仅为这个路径指定 `personal-dependabot` merge driver；同步命令通过 `-c merge.personal-dependabot.driver=true` 保留本地策略。这样，上游改动不会重新打开常规依赖升级 PR。此规则不应用于业务代码、Vercel 配置或其他工作流；其他路径发生冲突时仍中止同步。

手动同步时使用相同的 driver 配置：

```bash
git fetch --no-tags https://github.com/DIYgod/RSSHub.git master
git -c merge.personal-dependabot.driver=true merge --no-edit FETCH_HEAD
```

若上游新增包管理器或目录，先核查本地 Dependabot 策略是否也需添加对应的安全更新配置。

### 辅助工作流与通知

- 在本仓库的 Actions 设置中停用 `pr-review.yml`、`dependabot-fork.yml`、`ghcr-retention.yml` 和 `stale.yml`。它们分别依赖未配置的 AI 服务、发送不需要的忽略评论、清理本仓库不存在的镜像包或更新不需要的 PR 线程。共享文件保留上游原样，停用状态通过 GitHub 管理。
- 停用上游新增的 `sync-fork.yml`，本仓库仅保留一个同步器。这个仓库在 GitHub 中不是平台 Fork，不能直接使用 `gh repo sync`，也不能让它覆盖本地部署策略。
- 上游 `semgrep.yml` 也在本仓库停用。独立的 **Personal security scan** 保留扫描和 SARIF 上传；未配置 `SEMGREP_APP_TOKEN` 时使用 `p/default` 并关闭遥测。扫描成功不等于零告警，结果见 GitHub Code scanning。
- CodeQL、Test、Linter 和 Format 保留。个人工作流遵循仓库 `oxfmt` 格式，不向格式化任务提供同步 PAT。
- 本仓库不再订阅全部活动；这不会更改账号的全局邮件设置。生产工作流的失败通知继续由 GitHub Actions 通知设置控制，不用全局静音掩盖生产故障。
- 常规依赖 PR 的旧失败记录保留。当前上游已经移除 `teleproto@1.229.0` 旧补丁和不稳定的 Ubuntu Chromium Snap 测试矩阵，后续以同步后的 Test 结果验收。

在新部署仓库上复用此配置时，应核对上述工作流的停用状态；这些 GitHub 设置不会随 `git clone` 复制。

- **缺少 secret**：同步前即失败，补齐两个 secret 后重新运行。
- **401/403、workflow 权限或分支保护拒绝**：检查 PAT 的有效期、仓库范围、Contents/Workflows 权限和分支规则。不要用 force push 绕过。
- **合并冲突**：在本地拉取 fork，再按上文命令获取并合并上游、验证、推送。冲突解决之前工作流不会请求部署。共享辅助工作流优先保持上游原样，个人逻辑放在独立文件中。
- **Hook 超时、网络错误、响应缺少 job ID**：可能已创建部署，先检查 Vercel，确认状态后再决定是否勾选 `redeploy` 重跑。流程不会自动重试 POST。
- **同步成功但构建失败**：查看 Vercel 构建日志，修复配置或代码后手动勾选 `redeploy`。没有新上游提交时，定时任务不会重试部署。
- **定时任务未运行**：检查工作流是否启用、是否位于默认分支，以及 Actions 使用额度。GitHub 的计划任务可能延迟；公共仓库连续 60 天无活动时计划任务可能自动停用，需手动重新启用。

## 官方参考

- [RSSHub 上游 Vercel 配置](https://github.com/DIYgod/RSSHub/blob/master/vercel.json)
- [RSSHub 上游构建和运行时要求](https://github.com/DIYgod/RSSHub/blob/master/package.json)
- [Vercel Deploy Hooks](https://vercel.com/docs/deploy-hooks)
- [GitHub Actions 定时触发限制](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)

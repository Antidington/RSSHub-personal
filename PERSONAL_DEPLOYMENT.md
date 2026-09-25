# 本 fork 的自动同步与 Vercel 部署

配置核对日期：2026-09-26。工作流位于 `.github/workflows/personal-sync-deploy.yml`。

## 工作方式

- 每 6 小时将 `DIYgod/RSSHub` 的 `master` 合并至 `Antidington/RSSHub-personal` 的 `master`，保留上游 commit 历史和本 fork 的独立修改。跟踪开发分支最新提交，不以 release/tag 为更新条件，也不复制上游 tags 或 GitHub Releases。
- 有新提交且推送成功后，调用 Vercel Deploy Hook。没有新提交时不部署；手动运行时可以勾选 `redeploy` 重新部署。
- 合并冲突或推送失败时停止，不强制覆盖远端，不请求部署。需要先处理冲突或重新运行同步。
- Actions 成功仅代表同步成功及部署请求被接受；构建成功和实际路由可用性需要在 Vercel 和站点上验证。

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
3. 访问生产域名及你实际使用的一条 RSS 路由，确认返回预期内容。需要浏览器、登录凭据或外部依赖的路由应分别验证。

Vercel 按请求运行函数，不需要另行执行 `npm start` 或设置常驻保活任务。重新部署会构建并发布新版本，不代表所有 RSSHub 路由都能在 Serverless 环境运行。

首次同步会跨越此 fork 自 2023 年以来的上游变化，包含运行时、依赖、配置和许可证变化。自动合并成功不等于运行兼容；应核对同步后的上游说明、LICENSE、构建日志及实际订阅路由。此配置不执行 RSSHub 全量测试，也不保证未来任一上游提交都可部署。

## 故障处理

- **缺少 secret**：同步前即失败，补齐两个 secret 后重新运行。
- **401/403、workflow 权限或分支保护拒绝**：检查 PAT 的有效期、仓库范围、Contents/Workflows 权限和分支规则。不要用 force push 绕过。
- **合并冲突**：在本地拉取 fork，再获取上游并手动合并、验证、推送。冲突解决之前工作流不会请求部署。
- **Hook 超时、网络错误、响应缺少 job ID**：可能已创建部署，先检查 Vercel，确认状态后再决定是否勾选 `redeploy` 重跑。流程不会自动重试 POST。
- **同步成功但构建失败**：查看 Vercel 构建日志，修复配置或代码后手动勾选 `redeploy`。没有新上游提交时，定时任务不会重试部署。
- **定时任务未运行**：检查工作流是否启用、是否位于默认分支，以及 Actions 使用额度。GitHub 的计划任务可能延迟；公共仓库连续 60 天无活动时计划任务可能自动停用，需手动重新启用。

## 官方参考

- [RSSHub 上游 Vercel 配置](https://github.com/DIYgod/RSSHub/blob/master/vercel.json)
- [RSSHub 上游构建和运行时要求](https://github.com/DIYgod/RSSHub/blob/master/package.json)
- [Vercel Deploy Hooks](https://vercel.com/docs/deploy-hooks)
- [GitHub Actions 定时触发限制](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)

# anerp-platform/ci

anerp 客户项目的 PR 检查、部署和平台升级（GitHub Actions 可复用 workflow）。客户项目由 `anerp init` 生成 `.github/workflows/ci.yml`，只有引用这里某个版本的几行：

```yaml
jobs:
  check:
    uses: anerp-platform/ci/.github/workflows/check.yml@v<平台版本>
    secrets:
      ANERP_READ_TOKEN: ${{ secrets.ANERP_READ_TOKEN }}
```

步骤依次执行，任一步失败即阻断：

1. 安装依赖；
2. lint、Prettier、构建、类型检查；
3. `anerp check`（含接口契约）；
4. 单元测试和覆盖率，`domain/` 的变异测试（Stryker）；
5. 迁移检查；
6. 集成测试；
7. 端到端冒烟；
8. 依赖审计和密钥扫描。

PR 和推到 main 时，单元测试只跑改动的包和经 package.json 依赖它们的包（`anerp test --affected`），变异测试只跑 `src/domain/` 或单元测试有改动的模块（`anerp mutate --base`）；集成测试和端到端每次全跑。客户项目的 `ci.yml` 还在每晚、推送 `v*` 标签和手动运行时调用这个 workflow，这时跑全部。

输入：

- `coverage-threshold`：覆盖率门槛，默认 80；
- `mutation-threshold`：变异测试门槛，模块 `src/domain/` 的最低变异得分（%），默认 50；
- `node-version`：默认 22；
- `e2e`：是否运行端到端冒烟，默认运行。

## infra.yml

客户项目的 `.github/workflows/infra.yml`（`anerp init` 生成）调用它：改 `infra/` 的 PR 上 `terraform plan` 并把结果评论到 PR（一条评论，每次推送原地更新），合并到 main 后重新 plan 再 apply 到 dev。经 GitHub OIDC 扮演账号初始化建的角色，仓库里不存 AWS 密钥；仓库还没有角色的 secret 时跳过并在日志里说明。

输入：`environment`（默认 `dev`，对应 `infra/envs/<环境名>/`）、`aws-region`（默认 `eu-west-2`）、`terraform-version`。secret：`ANERP_READ_TOKEN`（从平台仓库拉模块）、`AWS_PLAN_ROLE_ARN`、`AWS_APPLY_ROLE_ARN`、`TF_STATE_BUCKET`。

## deploy.yml

客户项目的 `ci.yml` 在 main 上的检查通过后调用它，部署到 dev：构建 api、web 镜像（标签是 commit SHA，默认在 `ubuntu-24.04-arm` 上构建）→ 镜像大小上限 → Trivy 扫描（高危及以上、有修复版本的阻断）→ 推到 ECR → `anerp deploy release`（迁移 → 冒烟账号 → 滚动更新 → 冒烟测试，失败回滚）。经 GitHub OIDC 扮演部署角色；仓库还没有 `AWS_DEPLOY_ROLE_ARN` 时跳过并在日志里说明。

输入：`name`（环境前缀，如 `acme-dev`）、`aws-region`（默认 `eu-west-2`）、`runner`（默认 `ubuntu-24.04-arm`，CPU 架构要和环境一致）、`api-max-mb`（默认 400）、`web-max-mb`（默认 250）、`node-version`。secret：`ANERP_READ_TOKEN`、`AWS_DEPLOY_ROLE_ARN`。

## upgrade.yml

客户项目的 `.github/workflows/upgrade.yml`（`anerp init` 生成，老项目由 0.40.0 的 codemod 补上）每天和手动运行时调用它，平台升级不用人在本地操作：

1. 取目标版本：手动运行时填的版本，否则 `@anerp-platform/cli` 的最新版本。已经是这个版本，或者分支 `upgrade-platform-<版本>` 已存在（升级 PR 还开着，或关掉时没删分支），就什么都不做；
2. 在 GitHub 的运行器（`ubuntu-latest`）上运行目标版本的 `anerp upgrade`（codemod、改版本号、安装、构建、`anerp check`、单元测试）；
3. 提交到 `upgrade-platform-<版本>`，开 PR，PR 描述是升级摘要；
4. 升级顺利（各步骤通过、没有要手工处理的事）时，等 PR 触发的 workflow 全部通过后（每个 workflow 看最新一次运行）合并（merge commit）并删掉分支。之后在本地 `git pull` 即可；
5. 否则不合并：PR 加上 `upgrade-needs-attention` 标签，并评论要做什么（在这个分支上修好、推送，检查通过后合并）。

不用 GitHub 的自动合并：它要分支保护和必需的检查（私有仓库要付费计划），没有必需的检查时会立即合并。所以这个任务会一直等到 PR 的 workflow 结束（最长 2 小时）。

输入：`version`（默认空，即最新版本）、`node-version`。secret：

- `ANERP_READ_TOKEN`：同上，安装 `@anerp-platform/*`；
- `ANERP_UPGRADE_TOKEN`：只对本仓库的细粒度令牌（或 GitHub App 的令牌），权限 Contents、Pull requests、Issues、Workflows 读写（升级会改 `.github/workflows/` 里的版本号，所以要 Workflows），Actions 只读（等 PR 触发的 workflow 跑完；细粒度令牌没有 Checks 这一项，所以经 Actions 接口读，只看 GitHub Actions 的运行，缺了会直接报错，不合并）。用它推分支、开 PR、合并：`GITHUB_TOKEN` 开的 PR 不会触发检查。它只交给推送和 `gh` 的几步，不进 `.git/config`，安装和测试（运行项目的依赖）拿不到。没有设置时跳过并在日志里说明。

main 要求 PR 审批时，合并会失败，PR 加上标签等人合并。

这个仓库只有 workflow 定义。检查规则在 `@anerp-platform/cli` 里，随平台包发布。源码在平台仓库的 `ci/` 目录，平台发版时同步到这里，并打相同的版本标签，不要直接在这里修改。

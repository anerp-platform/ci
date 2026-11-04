# anerp-platform/ci

anerp 客户项目的 PR 检查（GitHub Actions 可复用 workflow）。客户项目由 `anerp init` 生成 `.github/workflows/ci.yml`，只有引用这里某个版本的几行：

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

这个仓库只有 workflow 定义。检查规则在 `@anerp-platform/cli` 里，随平台包发布。源码在平台仓库的 `ci/` 目录，平台发版时同步到这里，并打相同的版本标签，不要直接在这里修改。

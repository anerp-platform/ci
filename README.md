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

这个仓库只有 workflow 定义。检查规则在 `@anerp-platform/cli` 里，随平台包发布。源码在平台仓库的 `ci/` 目录，平台发版时同步到这里，并打相同的版本标签，不要直接在这里修改。

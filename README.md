下面是一套完整的 GitHub 网站操作步骤。配置完成后，每个 Fork 项目都会自动识别原始仓库和默认分支，不需要修改项目名称、仓库地址或分支名称。

## 1. 确认仓库是 Fork 项目

打开你自己的 GitHub 仓库，在仓库名称下方应该能看到：

```text
forked from 原作者/原仓库名
```

只有真正通过 **Fork** 创建的仓库，才能自动识别原始仓库。

## 2. 启用 GitHub Actions

1. 点击仓库顶部的 **Actions**。
2. 如果出现安全提示，点击：

```text
I understand my workflows, go ahead and enable them
```

3. 如果没有提示，说明 Actions 已经启用。

## 3. 创建自动同步工作流

在 **Actions** 页面：

1. 点击右上角绿色的 **New workflow**。
2. 点击 **Set up a workflow yourself**。
3. 将文件名修改为：

```text
.github/workflows/sync-fork.yml
```

不要选择左侧已有的 `deploy`、`test`、`docs-update` 等工作流，它们不是 Fork 同步任务。

## 4. 粘贴同步配置

删除编辑框中的原有内容，粘贴以下完整配置：

```yaml
# 工作流名称
name: Sync Fork

# 执行条件
on:
  # 每天 UTC 02:17 自动执行
  schedule:
    - cron: "17 2 * * *"

  # 允许在 Actions 页面手动执行
  workflow_dispatch:

# 允许工作流更新当前仓库
permissions:
  contents: write

# 防止多个同步任务同时运行
concurrency:
  group: sync-fork
  cancel-in-progress: false

jobs:
  sync:
    name: Sync upstream repository
    runs-on: ubuntu-latest

    steps:
      # 下载当前 Fork 仓库的完整代码和提交记录
      - name: Checkout fork repository
        uses: actions/checkout@v6
        with:
          fetch-depth: 0

      # 自动识别当前 Fork、原始仓库及双方默认分支
      - name: Detect fork and upstream repository
        id: repository
        env:
          GH_TOKEN: ${{ github.token }}
        shell: bash
        run: |
          set -euo pipefail

          IS_FORK="$(gh api "repos/${GITHUB_REPOSITORY}" --jq '.fork')"
          FORK_BRANCH="$(gh api "repos/${GITHUB_REPOSITORY}" --jq '.default_branch')"
          UPSTREAM_REPO="$(gh api "repos/${GITHUB_REPOSITORY}" --jq '.parent.full_name // empty')"
          UPSTREAM_BRANCH="$(gh api "repos/${GITHUB_REPOSITORY}" --jq '.parent.default_branch // empty')"

          if [ "${IS_FORK}" != "true" ]; then
            echo "::error::当前仓库不是 GitHub Fork，无法定位原始仓库。"
            exit 1
          fi

          if [ -z "${UPSTREAM_REPO}" ] || [ -z "${UPSTREAM_BRANCH}" ]; then
            echo "::error::未能识别原始仓库或默认分支。"
            exit 1
          fi

          echo "fork_branch=${FORK_BRANCH}" >> "${GITHUB_OUTPUT}"
          echo "upstream_repo=${UPSTREAM_REPO}" >> "${GITHUB_OUTPUT}"
          echo "upstream_branch=${UPSTREAM_BRANCH}" >> "${GITHUB_OUTPUT}"

          echo "当前 Fork：${GITHUB_REPOSITORY}"
          echo "Fork 默认分支：${FORK_BRANCH}"
          echo "原始仓库：${UPSTREAM_REPO}"
          echo "上游默认分支：${UPSTREAM_BRANCH}"

      # 将上游最新代码合并到当前 Fork
      - name: Merge upstream changes
        env:
          FORK_BRANCH: ${{ steps.repository.outputs.fork_branch }}
          UPSTREAM_REPO: ${{ steps.repository.outputs.upstream_repo }}
          UPSTREAM_BRANCH: ${{ steps.repository.outputs.upstream_branch }}
        shell: bash
        run: |
          set -euo pipefail

          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

          git remote remove upstream 2>/dev/null || true
          git remote add upstream "https://github.com/${UPSTREAM_REPO}.git"

          git fetch origin "${FORK_BRANCH}"
          git fetch upstream --prune
          git checkout "${FORK_BRANCH}"

          git merge --no-edit "upstream/${UPSTREAM_BRANCH}"

          if git diff --quiet "origin/${FORK_BRANCH}..${FORK_BRANCH}"; then
            echo "当前 Fork 已经是最新版本，无需推送。"
          else
            git push origin "${FORK_BRANCH}"
            echo "Fork 仓库同步完成。"
          fi
```

这段配置不需要针对不同项目进行修改。

## 5. 保存工作流

1. 点击右上角 **Commit changes...**。
2. 提交说明填写：

```text
添加 Fork 自动同步工作流
```

3. 选择：

```text
Commit directly to the main branch
```

4. 点击 **Commit changes**。

如果仓库默认分支是 `master`，GitHub 页面会自动显示提交到 `master`，不需要修改脚本。

## 6. 检查写入权限

进入：

```text
Settings
→ Actions
→ General
→ Workflow permissions
```

建议选择：

```text
Read and write permissions
```

然后点击 **Save**。

虽然脚本中已经声明了 `contents: write`，但如果仓库或组织设置限制写权限，仍需要在这里允许写入。

## 7. 第一次手动运行测试

1. 点击仓库顶部 **Actions**。
2. 左侧找到 **Sync Fork**。
3. 点击 **Run workflow**。
4. 选择默认分支。
5. 点击绿色 **Run workflow**。
6. 等待运行完成。

绿色对勾表示运行成功。

## 8. 检查识别结果

点击本次运行记录，打开：

```text
Detect fork and upstream repository
```

日志中应该显示：

```text
当前 Fork：你的用户名/仓库名
Fork 默认分支：main
原始仓库：原作者/原仓库名
上游默认分支：main
```

然后检查：

```text
Merge upstream changes
```

可能出现以下结果之一：

```text
Fork 仓库同步完成。
```

或者：

```text
当前 Fork 已经是最新版本，无需推送。
```

## 9. 后续自动运行

配置完成后，GitHub 每天 UTC 02:17 自动检查并同步一次，不需要再手动操作。

完整流程是：

```text
定时启动
→ 自动识别当前 Fork
→ 自动查询原始仓库
→ 自动识别默认分支
→ 拉取上游更新
→ 合并到 Fork
→ 推送更新
```

如果你在其他 Fork 项目中也需要自动同步，只需要重复创建同一个文件：

```text
.github/workflows/sync-fork.yml
```

并原样粘贴相同配置即可。存在代码冲突时任务会失败，但不会强制覆盖你自己的修改。

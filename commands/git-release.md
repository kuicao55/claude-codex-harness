# 版本发布：更新 README、版本号、GitHub Release、Marketplace

这个 command 会执行完整的版本发布流程。

## 前置条件

- 在 feature branch 上完成开发
- 测试通过

## 步骤

### 0. 前置检查

- 检查当前分支：必须在 feature branch，不能在 main
  - 如果在 main：提示用户先切换到 feature branch
- 检查 uncommitted 改动：**自动 commit**（见下方）
- 检查 worktree：提示确认无残留 worktree

```bash
git branch --show-current
git status --short
git worktree list
```

#### 自动 commit 未提交的改动

如果 `git status --short` 显示有未提交的文件（排除 `.DS_Store`、`.claude/.DS_Store`、`doc/.DS_Store`），自动执行：

```bash
# Stage 所有已 track 的改动和新的已 stage 文件（排除 .DS_Store 等）
git add .claude-plugin/marketplace.json .claude-plugin/plugin.json hooks/session-start
git add skills/ commands/ scripts/ agents/

# 确认有东西要提交
if git diff --cached --quiet; then
  echo "No significant changes to commit."
else
  git commit -m "chore: checkpoint before release
Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
fi
```

如果 commit 失败，提示用户检查。

#### 检查并推送 branch 到 remote

如果 branch 未 push 到 remote（`git log origin/<branch>..<branch>` 有输出），自动 push：

```bash
git push origin $(git branch --show-current)
```

如果 push 失败，提示用户检查。

### 1. 分析变更内容 + 合并到 main

#### 1.1 分析分支 diff

```bash
git diff main..HEAD --stat
git log main..HEAD --oneline
```

#### 1.2 创建 PR 并合并

```bash
# 创建 PR 并获取 PR number
PR_URL=$(gh pr create --title "Release vX.Y.Z" --body "Release notes" --base main)
gh pr merge --squash --delete-branch
```

或者手动合并后删除分支：
```bash
gh pr create --title "Release vX.Y.Z" --body "Release notes" --base main
# 在 GitHub 网页上手动 merge
# 然后继续执行后续步骤
```

#### 1.3 切换到 main 并分析真正要 release 的变更

```bash
git checkout main && git pull
git diff HEAD~1 --stat
git diff HEAD~1
```

### 2. 分析 README 是否需要更新

根据 git diff HEAD~1，检查 README.md 是否有变更：
- 如果有目录结构变更：更新 README 中的目录结构图
- 如果有新功能：确认 README 中的功能列表已更新
- 如果无变更：跳过此步骤

### 3. 确定新版本号

- 读取当前版本号（从 `.claude-plugin/plugin.json` 中获取）
- 自动去掉 `-devNNN` 后缀得到正式版本（如 `3.5.0-dev002` → `3.5.0`）
- 根据变更类型自动判断版本号递增：
  - **Patch** (x.x.N+1): bug 修复、小改动、文档更新
  - **Minor** (x.N+1.0): 新功能、新脚本、流程变更
  - **Major** (N+1.0.0): 架构变更、不兼容改动
- 告知用户新版本号，等待确认后再继续

### 4. 更新所有文件中的版本号

- 在整个项目中搜索当前的 dev 版本号（如 `3.5.0-dev001`）
- 将所有出现的地方替换为新正式版本号（如 `3.5.1`）
- 使用 python3 进行全局替换：
  ```bash
  python3 -c "
  import json, os
  for f in ['.claude-plugin/plugin.json', '.claude-plugin/marketplace.json']:
      with open(f, 'r') as fh:
          d = json.load(fh)
      d['version'] = 'X.Y.Z'
      if 'plugins' in d:
          d['plugins'][0]['version'] = 'X.Y.Z'
      with open(f, 'w') as fh:
          json.dump(d, fh, indent=2)
  "
  ```
- 更新 `hooks/session-start`：`sed -i '' 's/<old-version>/<new-version>/g' hooks/session-start`
- **必须**更新 `.claude-plugin/marketplace.json` 中的 version 字段

### 5. 提交并推送

- `git add` 所有修改的文件（**包括** `.claude-plugin/marketplace.json`，不要 add .DS_Store 等无关文件）
- 确保 `.claude-plugin/marketplace.json` 不是 untracked 状态
- 生成 commit message，格式：`chore: bump version to vX.Y.Z — <简要描述主要变更>`
- `git push origin main`

### 6. 创建 GitHub Release

- 使用 `gh release create` 创建新 release
- Title: `vX.Y.Z`
- Body: 列出本次所有变更，分为 New / Changed / Fixed 三类
- Tag: `vX.Y.Z`

### 7. 验证 tag 和版本一致性

- 运行 `git fetch origin tag vX.Y.Z` 获取 tag
- 运行 `git rev-parse vX.Y.Z` 确认 tag 指向的 commit 与版本 bump commit 相同
- 运行 `git show vX.Y.Z:.claude-plugin/plugin.json | grep version` 确认版本号正确
- 如果不一致：提示用户可能需要 `git push origin vX.Y.Z --force` 修正

### 8. 更新 claude-plugins marketplace

```bash
cd /Users/kuicao/Applications/claude-plugins
# 更新 marketplace.json
python3 -c "
import json
with open('.claude-plugin/marketplace.json', 'r') as f:
    d = json.load(f)
for plugin in d['plugins']:
    if plugin['name'] == 'super-harness':
        plugin['version'] = 'X.Y.Z'
with open('.claude-plugin/marketplace.json', 'w') as f:
    json.dump(d, f, indent=2)
"
# 更新 README.md 版本号
sed -i '' 's/super-harness (vX.Y-old)/super-harness (vX.Y-new)/g' README.md
# Commit and push
git add .claude-plugin/marketplace.json README.md
git commit -m "chore: bump super-harness to vX.Y.Z"
git push origin main
cd -
```

### 8.5 最终清理检查

```bash
git status          # 确认工作目录干净
git worktree list   # 确认无残留 worktree
git fetch + git status  # 确认本地远端一致
```

### 9. 输出结果

- 显示 commit 内容
- 显示 GitHub Release URL
- 确认 marketplace 已更新
- 显示 `plugin.json` 中的版本号和 tag 指向的 commit SHA，确认两者一致

## 注意

- **自动 commit**：Step 0 会自动 commit 所有已 stage 的改动，无需用户手动 commit
- **自动 push**：未 push 的 branch 会在 Step 0 自动 push 到 remote
- 版本号更新必须是全局的，不能遗漏任何文件
- Release notes 要详细，涵盖所有本次变更
- 如果 push 或 release 创建失败，提示用户检查 GitHub 状态
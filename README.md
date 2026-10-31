# mirror-worker

每 6 小时把 `wanghongyu666qiang` 账号下的**公开**仓库镜像到 Gitee（`gitee.com/starrylovetbao`）。
与 AskBridge 等业务仓库解耦：同步逻辑只存在于本仓库，业务仓库内不放任何同步工作流。

## 工作方式

`.github/workflows/mirror.yml`，单个 job，顺序如下：

1. 枚举仓库：`gh api /user/repos`，排除 fork、archived、**private**。
2. 对每个仓库两端 `git ls-remote`，过滤后比较 refs 集合；**完全一致就跳过**（这是耗时的关键，见下）。
3. Gitee 侧没有对应仓库时：调 Gitee API **自动创建**（建成私有，见已知边界），建完继续推。
4. 不一致才 `git clone --bare` → `git push --mirror`。

用 bare clone 而不是 `actions/checkout`：checkout 是 detached HEAD，只有当前分支是本地 ref，`--mirror` 会把 Gitee 上其他分支全部删除。

触发方式：cron `0 */6 * * *`（每 6 小时，北京时间 08/14/20/02 点），或手动：

```bash
# 全量
gh workflow run mirror.yml -R wanghongyu666qiang/mirror-worker
# 只跑一个仓
gh workflow run mirror.yml -R wanghongyu666qiang/mirror-worker -f repo=wanghongyu666qiang/AskBridge
```

## 配置

Secrets（都在仓库 Settings → Secrets and variables → Actions）：

| 名称 | 用途 |
| --- | --- |
| `GH_PAT` | 枚举仓库列表。classic PAT，只勾 `repo` |
| `GITEE_TOKEN` | Gitee 私人令牌，git push over HTTPS 当密码用；自动建库需要 `projects` 权限 |

Variable：`GITEE_NAMESPACE` = Gitee 空间名（当前 `starrylovetbao`）。

灌入命令（令牌不要经过聊天窗口）：

```bash
gh secret set GH_PAT        --repo wanghongyu666qiang/mirror-worker
gh secret set GITEE_TOKEN   --repo wanghongyu666qiang/mirror-worker
gh variable set GITEE_NAMESPACE --repo wanghongyu666qiang/mirror-worker --body "starrylovetbao"
```

两个令牌都写进 `~/.git-credentials`（`credential.helper store`），URL 里不出现令牌，git 报错也就无从回显；脚本另外对两者做了 `::add-mask::`。

## 控制文件

- `gitee-map.txt` —— `GitHub名=Gitee路径`。Gitee 的仓库路径**大小写敏感**，且"仓库名称"（显示名）与"仓库路径"是两个字段，git 只认路径。例如 `AskBridge=ask-bridge`。
- `gitee-skip.txt` —— 整行一个仓库名，完全不镜像。当前：`mirror-worker`、`wanghongyu666qiang`（profile 仓）。

## 加一个仓库到镜像

1. 什么都不用做：GitHub 上是公开仓库，下一轮就会自动建库 + 推送；不想等就手动触发一次。
2. Gitee 路径想与 GitHub 名不同，先在 `gitee-map.txt` 加一行再触发。
3. 自动建的仓库是**私有**的，去 Gitee 网页端 管理→基本信息 转一次公开即可（一次性，Gitee API 做不到）。

## 已知边界（都是实测结论，别再重复踩）

- **Gitee 接受来自 GitHub Actions runner 境外 IP 的 HTTPS 推送**，实测通过。
- **Gitee 建库 API 只能建成私有仓库**（2026-10-31 再次实测确认）：`POST /api/v5/user/repos` 传 `private=false` 被静默忽略，返回 201 但 `private: true`。转公开没有 API：`PUT /api/v5/repos/{o}/{r}` 回 405、`PATCH` 回 404，只能网页端手动转。推送到私有仓不受影响，所以工作流是自动建库 + 提醒手动转公开。
- **私有仓不镜像**（已定案）。另外当前 `GH_PAT` 能列出私有仓但读不到其内容（`git ls-remote` 回 `Repository not found`，像是 fine-grained 令牌缺 `Contents: Read-only`），要镜像私有仓得先修权限。
- **`--mirror` 是强制覆盖**：Gitee 上任何 GitHub 没有的分支会被删除，Gitee 侧不要有独立提交。
- **Release 附件（exe/zip）不同步**，Gitee 上只有代码和 tag。
- **耗时与配额**：真推一次 AskBridge 约 7 分 19 秒（47MB，跨境）；全量空转约 34 秒。本仓库 2026-10-31 起改为**公开仓**，Actions 分钟数免费不限量，不再有配额问题（顺带绕开了账户账单问题对私有仓 Actions 的封锁）。
- **Gitee 空仓库的默认分支记的是 `master`**，而镜像只有 `main`。因无更新仓库的 API（见上），只能网页端 管理→分支设置 改成 `main`。

## 排查

```bash
gh run list -R wanghongyu666qiang/mirror-worker --workflow mirror.yml --limit 5
gh run view <RUN_ID> -R wanghongyu666qiang/mirror-worker --log
```

日志最后一行是结论计数：

```
pushed=N created=N private_created=N skipped=N excluded=N missing=N failed=N
```

`failed>0` 才会让 job 变红；`created` 是本轮自动建的 Gitee 仓库数；`private_created` 是建完仍是私有、需要网页端手动转公开的数量；`missing` 是自动建库也失败（令牌缺 `projects` 权限或仓库名 Gitee 不接受），带 `::error` 注解，不算 job 失败。

比对 refs 时**必须**滤掉 `refs/pull/*`（GitHub 会附带）和 annotated tag 的 `^{}` 剥离线，否则两端永远不相等、每次都触发全量重推。

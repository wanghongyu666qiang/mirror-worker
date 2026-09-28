# mirror-worker

每小时把 `wanghongyu666qiang` 账号下的**公开**仓库镜像到 Gitee（`gitee.com/starrylovetbao`）。
与 AskBridge 等业务仓库解耦：同步逻辑只存在于本仓库，业务仓库内不放任何同步工作流。

## 工作方式

`.github/workflows/mirror.yml`，单个 job，顺序如下：

1. 枚举仓库：`gh api /user/repos`，排除 fork、archived、**private**。
2. 对每个仓库两端 `git ls-remote`，过滤后比较 refs 集合；**完全一致就跳过**（这是配额的关键，见下）。
3. 不一致才 `git clone --bare` → `git push --mirror`。

用 bare clone 而不是 `actions/checkout`：checkout 是 detached HEAD，只有当前分支是本地 ref，`--mirror` 会把 Gitee 上其他分支全部删除。

触发方式：cron `0 * * * *`（每小时），或手动：

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
| `GITEE_TOKEN` | Gitee 私人令牌，git push over HTTPS 当密码用 |

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

1. 在 Gitee 建**空**仓库：不要初始化 README/LICENSE/.gitignore，不要开分支保护。
2. 名字与 GitHub 一致就完事；不一致就在 `gitee-map.txt` 加一行。
3. 下一轮自动推；不想等就手动触发一次。

worker **不会**替你建仓库，只会在日志里报 `Gitee repo missing` 的 notice。

## 已知边界（都是实测结论，别再重复踩）

- **Gitee 接受来自 GitHub Actions runner 境外 IP 的 HTTPS 推送**，实测通过。
- **Gitee 建库接口无视可见性参数**：`POST /api/v5/user/repos` 传 `public=true` 直接回 `public is invalid`；传 `private=false` 或 `private=0` 被静默忽略，**一律建成私有**；返回的 `html_url` 带 `.git` 后缀。所以自动建库功能已删除，不要重试。
- **私有仓不镜像**（已定案）。另外当前 `GH_PAT` 能列出私有仓但读不到其内容（`git ls-remote` 回 `Repository not found`，像是 fine-grained 令牌缺 `Contents: Read-only`），要镜像私有仓得先修权限。
- **`--mirror` 是强制覆盖**：Gitee 上任何 GitHub 没有的分支会被删除，Gitee 侧不要有独立提交。
- **Release 附件（exe/zip）不同步**，Gitee 上只有代码和 tag。
- **耗时与配额**：真推一次 AskBridge 约 7 分 19 秒（47MB，跨境）；全量空转约 34 秒。本仓库是私有仓，Actions 分钟数计入免费 2000 分/月，按每小时 + 每天一次真实推送估算约 650 分/月。
- **Gitee 空仓库的默认分支记的是 `master`**，而镜像只有 `main`。若页面显示找不到分支，去 管理 → 分支设置 改成 `main`。

## 排查

```bash
gh run list -R wanghongyu666qiang/mirror-worker --workflow mirror.yml --limit 5
gh run view <RUN_ID> -R wanghongyu666qiang/mirror-worker --log
```

日志最后一行是结论计数：

```
pushed=N skipped=N excluded=N missing=N failed=N
```

`failed>0` 才会让 job 变红；`missing` 是 notice（Gitee 上还没建对应仓库），不算失败。

比对 refs 时**必须**滤掉 `refs/pull/*`（GitHub 会附带）和 annotated tag 的 `^{}` 剥离线，否则两端永远不相等、每次都触发全量重推。

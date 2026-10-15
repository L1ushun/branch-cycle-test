# branch-cycle-test

GitHub Actions 分支循环自动化：**从主分支切开发分支 → 开发 → CI 全绿自动 ready → 任意时间点合并 → 自动切下一轮分支**，循环往复。

合并时间点不固定、不需要排期——想合并就触发一次 `cycle`，其余全部自动。

## 一图看懂

```
            ┌──────────────────────────────────────────────────┐
            │                                                  │
            ▼                                                  │
┌─── 触发 cycle ───┐                                           │
│ ① 合并所有 ready PR │ → master 前进一步                        │
│ ② 切新 dev 分支    │ → 从最新 master 切出                      │
│ ③ 建 wip PR       │ → 红色 wip 标签 = 开发中                  │
└──────────────────┘                                           │
            │                                                  │
            ▼                                                  │
    开发者 push 代码 ──→ CI 检查 ──→ 全绿？──是──→ auto-ready 摘掉 wip ──→ 等待下次触发 cycle
                              │
                              └─否→ 保持 wip，不会被合并，修好再 push
```

## 三个 workflow

| 文件 | 职责 | 触发方式 |
|---|---|---|
| `ci.yml` | 跑检查（当前是占位 echo，**换成你的 lint/test/build**） | push 到 `dev-*` 分支 / PR 更新 |
| `auto-ready.yml` | CI 全绿 → 自动摘除 wip 标签（等价"转 ready"） | `ci` 运行完成 |
| `cycle.yml` | 合并所有 ready PR + 从最新 master 切新分支 + 建新 wip PR | 手动 / 定时 / push `TRIGGER_CYCLE` / API |

## 状态语义：wip 标签

| 状态 | 含义 | 会被合并吗 |
|---|---|---|
| 打着 `wip` 标签 | 开发中（CI 未过或未跑完） | ❌ 永远不会 |
| `wip` 已被摘掉 | 开发完成、CI 全绿、待合并 | ✅ 下次触发 cycle 即合并 |

> 为什么不用 draft：`GITHUB_TOKEN` 无权调用 `markPullRequestReadyForReview`（转 ready），但有权操作标签，所以用 `wip` 标签等价替代，这是社区标准做法。

## 日常怎么用

### 开发者：只管 push

```bash
git push origin dev-20261015
```

push 后自动走：CI 检查 → 全绿 → 摘 wip → 进入待合并状态。CI 失败则保持 wip，修好再 push，不用担心半成品被合并。

### 合并负责人：想合并时触发一次 cycle

```bash
gh workflow run cycle.yml --repo L1ushun/branch-cycle-test
```

可选触发方式：

| 方式 | 操作 |
|---|---|
| 网页 | Actions → cycle → Run workflow |
| 命令行 | `gh workflow run cycle.yml` |
| API | `POST /repos/{owner}/{repo}/actions/workflows/cycle.yml/dispatches` |
| 文件触发 | 修改 `TRIGGER_CYCLE` 文件内容推到 master |
| 定时兜底 | 取消 cycle.yml 中 `schedule` 段的注释，配 cron 表达式 |

触发即一轮：合并所有 ready PR → 切新分支 → 新 wip PR 就位，等下一轮。

## 部署到你的仓库（checklist）

1. **复制** `.github/workflows/` 下三个文件到你的仓库，把 `master` 全局替换为你的主分支名
2. **替换 ci.yml** 为真实的检查命令（lint / test / build）
3. **开启仓库设置**（默认关闭，不开则建 PR 报错 `not permitted to create pull requests`）：
   - 网页：Settings → Actions → General → Workflow permissions → 勾选 **Allow GitHub Actions to create and approve pull requests**
   - 或 API（注意字段名，传错会被静默忽略）：
     ```bash
     gh api -X PUT repos/{owner}/{repo}/actions/permissions/workflow \
       -f default_workflow_permissions=write -F can_approve_pull_request_reviews=true
     ```
4. **触发一次 cycle**，循环即开始

## 踩坑记录（FAQ）

- **Q：cycle 建的 PR，CI 卡在 `action_required`？**
  A：GitHub 的防递归安全机制——GITHUB_TOKEN 创建的 PR 首次触发 CI 需人工批准一次。日常以"人 push 代码"为链条源头时不会遇到（真实协作天然如此）。

- **Q：为什么不用 `check_suite` 触发器？**
  A：它有大量静默不触发的已知问题，`workflow_run` 是官方推荐的稳定替代。

- **Q：auto-ready 为什么有时跑两次？**
  A：同一次 push 产生 push + pull_request 双事件，各发一次 `workflow_run`。逻辑幂等，无害。

- **Q：同一天多次触发，分支会重名吗？**
  A：不会，检测到重名自动追加 `-HHMMSS` 时间戳；且合并时 `--delete-branch` 会先清掉旧分支，多数情况原名可复用。

- **Q：`gh pr checks` 报 `Resource not accessible by integration`？**
  A：workflow 的 `permissions` 需要 `actions: read` + `checks: read`（读 checks 汇总）。

- **Q：无 checkout 步骤的 job 里 gh 报 `not a git repository`？**
  A：加环境变量 `GH_REPO: ${{ github.repository }}`。

- **Q：runner 里 `gh label create --if-not-exists` 报 unknown flag？**
  A：runner 内置 gh 版本较旧，用 `|| echo "已存在"` 兜底实现幂等。

## 验证记录

本仓库已跑通完整闭环（2026-10-15）：

- 触发 cycle → 切分支 + 建 wip PR ✓
- push 代码 → CI 绿 → auto-ready 自动摘 wip ✓
- 再触发 cycle → PR 自动合并 + 下一轮分支和 wip PR 就位 ✓
- v1.17.0 开发提交

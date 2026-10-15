# branch-cycle-test

测试 GitHub Actions 实现的分支循环：

```
触发 cycle → 合并所有 ready PR 到主分支
           → 从最新主分支切新 dev 分支
           → 建 draft PR（不合并）
           → 开发/推送代码
           → 再次触发 → 下一轮循环
```

## 触发方式

1. **网页手动**：Actions → cycle → Run workflow（时间不固定，随时触发）
2. **Push 触发**：修改根目录 `TRIGGER_CYCLE` 文件内容并推送到主分支
3. **命令行**：`gh workflow run cycle.yml`（需 gh auth login）
4. **API**：`POST /repos/{owner}/{repo}/actions/workflows/cycle.yml/dispatches`

## 验证要点

- 第一次触发：无 PR 可合并 → 切出 `dev-YYYYMMDD` 分支 → 出现 draft PR
- 将 draft PR mark ready 后再触发：旧 PR 被合并 → 自动切出新分支 → 出现新 draft PR
- draft 状态的 PR 永远不会被合并（合并步骤只挑非 draft 的）
- 开发提交：验证 auto-ready 全链路

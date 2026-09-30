# hello-pipeline

最小测试：往 `inbox/` 上传一个 txt、md、pdf 或 docx，Claude Code 云端在内容末尾加一段 hello，生成 Word，自动放回 `outbox/`。用来确认“GitHub 上传 → Claude Code 云端执行 → 结果回到 GitHub”这条链路是通的。

```
上传 inbox/xxx.txt
  │ fire.yml（GitHub Actions）：找出新上传的文件，调用 Routine 的 API
  ▼
Claude Code 云端（Routine）：按 routine-prompt.md 运行 make_hello.py，推送 claude/hello-<时间> 分支
  │ merge.yml（GitHub Actions）：确认只改了 outbox/，合并到 main，删除分支
  ▼
main 分支出现 outbox/xxx.docx
```

## 一次性设置

**GitHub**

1. 新建一个私有仓库（例如 `hello-pipeline`），把本文件夹的内容推上去。
2. 给 Claude GitHub App 授权这个仓库：github.com/apps/claude/installations/new → 选你的账号 → Repository access 勾上这个仓库 → Save。
3. 仓库 Settings → Actions → General → Workflow permissions 选 **Read and write permissions** → Save。

**Claude Code 云端**

4. 打开 claude.ai/code/routines → New routine：
   - Name：hello 测试
   - Instructions：粘贴 `routine-prompt.md` 的全部内容
   - Repository：选第1步的仓库
   - Environment：Default 即可（默认的 Trusted 网络允许 pip 安装）
   - Trigger：选 API
   - Connectors：全部移除
   - 点 Create
5. 进入这个 Routine → 菜单 → Edit → 在 Select a trigger 下找到 API → 复制 URL → 点 Generate token，复制令牌（只显示一次）。

**连起来**

6. 仓库 Settings → Secrets and variables → Actions → New repository secret，添加：
   - `ROUTINE_FIRE_URL`：第5步的 URL
   - `ROUTINE_FIRE_TOKEN`：第5步的令牌

## 测试

1. GitHub 网页进入 `inbox/` → Add file → Upload files → 上传一个空的 txt（或 pdf、docx）→ Commit changes。
2. Actions 页面会出现“上传后触发 Claude 云端”。点进去，运行摘要里有云端会话链接，可以实时看 Claude 在做什么。
3. 云端推送分支后，Actions 页面出现“云端结果合并回 main”。
4. 两个都变绿后，回到仓库首页，`outbox/` 下有同名的 .docx，下载打开，最后一段是 hello。

## 哪一步出问题，说明什么

| 现象 | 原因 |
|---|---|
| 上传后 Actions 没有任何运行 | 文件没放在 `inbox/` 下，或者没传到 main 分支 |
| “触发云端”报“缺少 ROUTINE_FIRE_URL” | 第6步的 Secrets 没设置，或名称拼错 |
| “触发云端”报 HTTP 401 | 令牌错误或已被重新生成，回第5步重新生成并更新 Secret |
| “触发云端”报 HTTP 400 且提到 paused | Routine 被暂停了，在 claude.ai/code/routines 打开开关 |
| “触发云端”报 HTTP 429 | 触发太频繁（每个 Routine 每小时 30 次），稍后再试 |
| 云端会话里 git clone 或 push 失败 | 第2步 Claude GitHub App 没有授权这个仓库 |
| 云端会话里 pip install 失败 | 环境的网络访问被改成了 None，改回 Trusted |
| “合并回 main”报改动了 outbox 以外的文件 | 云端没按提示词只提交 outbox/，打开分支看看改了什么 |
| “合并回 main”在 push 时报 403 | 第3步没开写权限，或 main 有分支保护 |

出问题时，把 Actions 运行记录的报错和云端会话链接发给我。

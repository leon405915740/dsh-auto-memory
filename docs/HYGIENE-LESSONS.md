# 环境操作踩坑清单（HYGIENE LESSONS）

> 2026-08-31 建立。开发/部署 dsh-auto-memory 时遇到的环境坑与标准动作，跨机器可参考（尤其家里电脑）。

## 1. 版本号陷阱：commit message ≠ package.json version
- 坑：改代码后 commit 写 `v0.1.32`，但 `package.json` 的 `version` 仍 `0.1.30`——"看版本号判断是否最新"会误判。
- 标准动作：发版时**同步升 package.json version**（与提交信息一致）；判断是否最新看**提交 hash + 代码特征**，不能只看版本号或 commit 字样。
- 参考：commit `5a5326f` 修复了此问题。

## 2. git push 前必须探活代理（Windows 本机）
- 坑：v2rayN 进程可能没在运行；git 直连 github.com 会被重置/连接失败——commit 成功但 push 失败，需二次补救。
- 标准动作（push 前）：
  ```powershell
  if (-not (Test-NetConnection -ComputerName 127.0.0.1 -Port 10808 -InformationLevel Quiet -WarningAction SilentlyContinue)) {
    Start-Process "D:\v2rayN-windows-64\v2rayN.exe" -WindowStyle Hidden
    Start-Sleep -Seconds 8
  }
  git push
  ```
- git 全局代理已配 `http://127.0.0.1:10808`，但**代理进程不活时该配置无效**。

## 3. PowerShell 读/写 UTF-8 中文文件（GBK 坑）
- 坑：`Get-Content -Raw`（默认编码）或 `ConvertFrom-Json` 读中文 JSON 会出现乱码或 "传入的对象无效" 报错——本机 PowerShell 默认按 GBK 读，文件是 UTF-8。
- 标准动作：
  - 读：`[System.IO.File]::ReadAllText($path, [System.Text.Encoding]::UTF8)`
  - 写（无 BOM，避免 pnpm/JSON.parse 失败）：`$utf8 = New-Object System.Text.UTF8Encoding($false); [System.IO.File]::WriteAllText($path, $txt, $utf8)`
- 注意：Windows PowerShell 5 的 `Set-Content -Encoding UTF8` 是**带 BOM** 的，会破坏 JSON 解析（曾导致 pnpm git 安装失败）。

## 4. pnpm git 依赖更新不重取 tip
- 坑：git 安装的依赖（`github:user/repo`）用 `pnpm update` 可能不重取分支 tip（仍用旧 commit）。
- 标准动作：强制重解析用 `pnpm add github:user/repo#branch`（或指定 commit SHA）；安装后**核对 node_modules 实际文件**（如 lib/index.js 是否含新特征），不能只看 lockfile 或版本号。

## 5. gh api --jq 在 PowerShell 的引号坑
- 坑：`gh api ... --jq ".field + ' @ ' + .field2"` 报 "failed to parse jq expression"（单引号在 PowerShell 中转义问题）。
- 标准动作：避免在 jq 表达式里拼字符串；用单字段 `--jq ".sha"`，或 `--jq "[.sha,.message] | .[]"`，或用双引号包裹。

---

## 配套纪律（已写入插件注入，家里电脑装插件自动生效）
- **踩坑留痕铁律**：当轮发生失败-重试/被纠正/二次动作，必须当轮写入 Lessons（坑+标准动作），禁止只修不记。
- **动手前先查教训**：关键操作（push/发版/改代码/网络/曾失败任务）前先 memory_recall 检索教训。
- **留痕核验铁律**：声称"已记录"必须读回核验文件实际内容，不以声称/转述为准。

# Windows 无管理员权限：把 Codex CLI 编译成 `coder.exe`（GitHub Actions SOP）

这份 SOP 记录了我这次在 **本机无法安装/无法编译** 的前提下，仍然在 Windows 上拿到可用 `coder.exe` 的完整流程。后续你改代码 → 重新编译，只要重复 “循环编译” 部分即可。

> 核心思路：本机不装 VS/WSL，不在本机编译；把编译搬到 **GitHub Actions 的 `windows-latest`**（自带 MSVC/Windows SDK）上跑，产物下载回来用。

---

## 0. 之前为啥一直失败？

Rust 在 Windows 默认用 `x86_64-pc-windows-msvc` 目标，需要 MSVC 链接器（`link.exe`）和 Windows SDK。你本机没有这些（且没管理员权限装 VS Build Tools / WSL），所以 `cargo build` 在本机必然失败。

---

## 1. 前置条件（一次性）

本机需要：

- Git for Windows（能跑 `git`）
- GitHub CLI（能跑 `gh`，并已登录）
- 你能访问 GitHub（HTTPS）

> PowerShell 提醒：你这个环境不支持 `&&`，下面命令都用 **一行一个命令**（或用 `;`）。

---

## 2. 一次性初始化：让 `gh` 有 workflow 权限

如果你要 **push/更新** `.github/workflows/*.yml`，GitHub 要求 token 有 `workflow` scope，否则会报：
`refusing to allow an OAuth App to create or update workflow ... without workflow scope`

在仓库目录里跑：

```powershell
cd D:\Users\johntitor.wu\github\codex
gh auth refresh --scopes workflow
gh auth setup-git
```

---

## 3. 一次性初始化：准备 fork + remote

推荐用 fork，别直接动 `openai/codex`。

如果你已经在 `D:\Users\...\github\codex` 里有 upstream 的 clone（我们就是这种情况），确保有 `fork` remote 指向你自己的仓库：

```powershell
cd D:\Users\johntitor.wu\github\codex

# 看当前 remotes
git remote -v

# 如果没有 fork remote，就加一个（把 demon2036 换成你的 GitHub 用户名）
git remote add fork https://github.com/<YOUR_GH_USERNAME>/codex.git

# （可选）确保 upstream 还是 origin
git remote -v
```

---

## 4. 一次性初始化：添加 Windows 编译 workflow（关键）

文件路径（必须在默认分支可见）：`.github/workflows/windows-dev-build.yml`

这就是我们这次用的版本（会把 `codex.exe` 改名成 `coder.exe` 并上传 artifact）：

```yaml
name: windows-dev-build

on:
  workflow_dispatch:

concurrency:
  group: windows-dev-build-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    name: Build (windows-latest)
    runs-on: windows-latest
    defaults:
      run:
        working-directory: codex-rs

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@1.92.0

      # 缓存：第一次跑会慢（20~30min 很常见），但后续改代码重复编译会快很多
      - uses: Swatinem/rust-cache@v2
        with:
          workspaces: |
            codex-rs -> target

      - name: Cargo build (release)
        shell: bash
        run: |
          set -euo pipefail
          cargo build --locked --release --bin codex --bin codex-windows-sandbox-setup --bin codex-command-runner

      - name: Collect artifacts
        shell: bash
        run: |
          set -euo pipefail
          mkdir -p ../dist
          cp target/release/codex.exe ../dist/coder.exe
          cp target/release/codex-windows-sandbox-setup.exe ../dist/
          cp target/release/codex-command-runner.exe ../dist/

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: codex-windows-x86_64-pc-windows-msvc
          path: |
            dist/coder.exe
            dist/codex-windows-sandbox-setup.exe
            dist/codex-command-runner.exe
```

把它提交并推到你 fork 的 **默认分支（通常是 main）**：

```powershell
cd D:\Users\johntitor.wu\github\codex

git checkout -B fork-main fork/main
git add .github/workflows/windows-dev-build.yml
git commit -m "ci: build windows coder artifact"
git push fork HEAD:main
```

### 常见坑：`workflow ... not found on the default branch`（你之前遇到的 404）

你之前 `gh workflow run windows-dev-build.yml ...` 报 404，就是因为：

- `gh` 会到 **默认分支** 去找 workflow；
- workflow 只在 `coder` 分支时，默认分支找不到 → 直接 404。

解决就是：**把 workflow 文件放进默认分支**（上面这一步）。

---

## 5. 循环编译（你后面大量改代码就重复这个）

### 5.1 改代码并推送到一个分支（例如 `coder`）

```powershell
cd D:\Users\johntitor.wu\github\codex

git checkout -B coder
# ...改代码...
git add <你改的文件>
git commit -m "your change"
git push -u fork coder
```

### 5.2 触发 GitHub Actions 编译（在 Windows runner 上）

```powershell
cd D:\Users\johntitor.wu\github\codex

$me = gh api user -q .login
gh workflow run windows-dev-build -R "$me/codex" --ref coder
```

### 5.3 找到 run id 并等待跑完

```powershell
$me = gh api user -q .login
gh run list -R "$me/codex" --workflow windows-dev-build --limit 5

# 复制最新那条的 run id，然后：
gh run watch <RUN_ID> -R "$me/codex" --exit-status
```

### 5.4 下载 artifact（里面就是 `coder.exe`）

```powershell
cd D:\Users\johntitor.wu\github\codex

$me = gh api user -q .login
if (Test-Path .\_coder_artifact) { Remove-Item -Recurse -Force .\_coder_artifact }
New-Item -ItemType Directory -Force .\_coder_artifact | Out-Null

gh run download <RUN_ID> -R "$me/codex" -n codex-windows-x86_64-pc-windows-msvc -D .\_coder_artifact
```

下载后你会看到（示例）：

- `.\_coder_artifact\coder.exe`
- `.\_coder_artifact\codex-command-runner.exe`
- `.\_coder_artifact\codex-windows-sandbox-setup.exe`

---

## 6. 安装到本机（不需要管理员）

我这次的安装路径是：`%USERPROFILE%\.coder\bin`

```powershell
$bin = Join-Path $env:USERPROFILE ".coder\\bin"
New-Item -ItemType Directory -Force $bin | Out-Null

# 先更新配套 exe（通常不会被占用）
Copy-Item -Force .\_coder_artifact\codex-command-runner.exe $bin
Copy-Item -Force .\_coder_artifact\codex-windows-sandbox-setup.exe $bin

# 再更新主程序（如果正在运行 coder.exe，这一步会报 “being used by another process”）
Copy-Item -Force .\_coder_artifact\coder.exe (Join-Path $bin "coder.exe")
```

如果你看到 “because it is being used by another process”，说明你有 `coder.exe` 还在运行。两种做法：

1) 退出所有 `coder.exe` 后重跑上面那行覆盖
2) 先并排放一个新版本（不用停旧进程也能立即用），等方便的时候再切换：

```powershell
$runId = <RUN_ID>
Copy-Item -Force .\_coder_artifact\coder.exe (Join-Path $bin "coder-$runId.exe")
& (Join-Path $bin "coder-$runId.exe") --help

# 退出所有 coder 后切换成正式名字
Move-Item -Force (Join-Path $bin "coder-$runId.exe") (Join-Path $bin "coder.exe")
```

把这个目录加入 **用户 PATH**（写 User 环境变量，不需要管理员；新开一个终端才生效）：

```powershell
$bin = (Join-Path $env:USERPROFILE ".coder\\bin")
$userPath = [Environment]::GetEnvironmentVariable("Path","User")
if ([string]::IsNullOrEmpty($userPath)) {
  $newPath = $bin
} elseif ($userPath.Split(";") -contains $bin) {
  $newPath = $userPath
} else {
  $newPath = "$userPath;$bin"
}
[Environment]::SetEnvironmentVariable("Path",$newPath,"User")
```

验证：

```powershell
& "$env:USERPROFILE\.coder\bin\coder.exe" --help
```

> 注意：我们只是把文件名改成 `coder.exe`，但 CLI help 里仍显示 `Usage: codex ...`，这是代码里硬写了 `bin_name = "codex"`，属于“只改 exe 名字”的预期行为。

---

## 7. 常见问题排障

### 7.1 push workflow 报没有 `workflow` scope

重新跑：

```powershell
gh auth refresh --scopes workflow
gh auth setup-git
```

### 7.2 `gh workflow run ...` 报 404

确保 workflow 在默认分支可见：

```powershell
$me = gh api user -q .login
gh workflow list -R "$me/codex"
```

能看到 `windows-dev-build` 才算 OK。

### 7.3 安全软件提示风险

- 这套 SOP 没有在你机器上安装编译器/驱动/内核组件；只是下载你自己 fork 在 GitHub Actions 上编出来的 exe。
- 但产物 **没有代码签名**，很多企业安全策略会拦未知签名的 exe。
- 你可以先做 hash 备案：

```powershell
Get-FileHash -Algorithm SHA256 "$env:USERPROFILE\.coder\bin\coder.exe"
```

---

## 8. （可选）真正把显示名/帮助也改成 `coder`

现在只是 `codex.exe -> coder.exe` 的重命名；如果你想 help/usage/completion 也都叫 `coder`，需要改 Rust 代码（例如 `codex-rs/cli/src/main.rs` 里 `bin_name = "codex"`、`override_usage = "codex ..."`、以及 `print_completion()` 里 hardcode 的 `"codex"`）。

做这件事会牵连一些测试/文案（比如内部用 `try_parse_from(["codex", ...])`），建议你确认需求后再动；你确定要我把这部分也“魔改”掉的话，我可以给你一套最小改动方案 + 重新跑 Actions 出新 `coder.exe`。

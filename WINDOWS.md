# 在 Windows 上跑 dsh-superagent —— 本地适配记录

> 2026-11-01 实测。上游 `deepseek-ai/dsh-superagent-prototype`（单提交 `6337fc8`，v0.1.0）**未针对 Windows 做适配**。
> 本文记三处不兼容、装法与验证结果。**改上游代码前先读这里，别重复排查。**

---

## 一、四处不兼容

### ① `stateDir` 只认 POSIX 绝对路径（**必须打补丁，否则插件不加载**）

**症状**：面板上该插件显示 `异常`，配置校验失败：

```
$.stateDir expect string to match regexp /^\// (at stateDir)
```

**位置**：`src/deployment.ts:13`

```ts
// 上游
const absolute = (description: string) => s.string().pattern(/^\//).description(`${description} Absolute path.`)
```

**原因**：描述写的是 `Absolute path`，正则写的是 `POSIX 绝对路径`。`C:/Users/...` 首字符是 `C`，直接不匹配。
而 `stateDir` 后续只喂给 `mkdirSync` / `join` / `lockSync`——**都是 Node 标准 fs，Windows 路径本来完全可用**，卡住的只有这道正则。

**本地改法**：

```ts
const absolute = (description: string) => s.string().pattern(/^(?:\/|[A-Za-z]:[\\/])/).description(`${description} Absolute path.`)
```

**影响面**：任何 Windows 用户装它都会撞上；错误信息不会告诉你该怎么办。
仓库 `has_issues: false`，**报不进上游**，所以只能靠这份文件。

### ② `pnpm test` 有 38 个假失败（**不影响 build，别被吓到**）

```
Test Files  9 failed | 50 passed (59)
     Tests  38 failed | 1159 passed (1197)
```

**原因**：测试断言 POSIX 权限位。

```js
expect([dir.mode & 0o777, file.mode & 0o777]).toEqual([448, 384])   // 0700 / 0600
// 实际收到 [438, 438] = 0666
```

Windows 的 `chmod` 只切只读位，`0o700` / `0o600` 设不上，所有文件都回 `0666`。

**处置**：不认识，但**不要因此以为代码坏了**。`pnpm typecheck` 通过、`pnpm build` 通过。
`pnpm check` 是 `typecheck && test && build`——**测试挂了 build 就不跑**，所以单独跑：

```bash
pnpm build
```

### ③ 缺 `tmux` ⇒ 服务管理脚本不可用（**可绕过**）

**位置**：`scripts/dsh-superagent`（Python 脚本）

```python
if not dsh or not shutil.which('tmux'):
    raise RuntimeError('Both dsh and tmux must be on PATH.')
```

它用 `tmux new-session -d` 把 dsh 托管到后台 session 里。

**处置**：**插件本身不依赖 tmux**，只有这个便利脚本依赖。
手动在终端里跑 `dsh <profile>`，终端别关即可。

**附带影响**：`006-deployment.md` 那个"每小时检查"也依赖这个脚本——**在 Windows 上它会一直报错**。
所以**不要**把它加进你日常用的 profile。

---

### ④ 找不到 `gh`（以及 `git` / `dsh` / `lark-cli`）—— 只按裸名找，不补 `.exe`

**症状**：面板部署检查报

```
找不到智能体的 GitHub CLI。
gh is not an executable file on PATH
```

**位置**：`src/deployment.ts:62` 的 `executable()`

```ts
// 上游
.map(dir => join(dir, name))     // 找 "C:\Program Files\GitHub CLI\gh"
                                 // 真文件是 "gh.exe" —— 永远找不到
```

**原因**：Windows 上 PATH 里存的是**不带扩展名的命令名**，磁盘上是 `gh.exe`。Node 的 `statSync('<dir>/gh')` 直接 ENOENT。

**影响面**：`git`、`dsh`、`lark-cli` 走的是**同一个函数**——**在 Windows 上，凡是用它定位的命令全部失败**，不止 gh。

**本地改法**：按 `PATHEXT` 展开候选

```ts
const PATHEXT = process.platform === 'win32' ? (process.env.PATHEXT ?? '.COM;.EXE;.BAT;.CMD').split(';').filter(Boolean) : []

export function executable(name: string, path: string, exclude?: string): string {
  if (name.includes('/') && !isAbsolute(name)) throw new Error(`${name} must be a command name or an absolute path`)
  const bare = isAbsolute(name) ? [name] : path.split(delimiter).filter(dir => isAbsolute(dir) && !(exclude && sameDirectory(dir, exclude))).map(dir => join(dir, name))
  // 只对"没有扩展名"的候选做补全，避免把 gh.exe 变成 gh.exe.exe
  const candidates = bare.flatMap(p => PATHEXT.length && !/\.[A-Za-z0-9]+$/.test(p) ? ['', ...PATHEXT].map(ext => p + ext) : [p])
  for (const candidate of candidates) {
    try { if (statSync(candidate).isFile()) { accessSync(candidate, constants.X_OK); return candidate } }
    catch { /* Not an executable file here; try the next PATH directory. */ }
  }
  throw new Error(`${name} is not an executable file${isAbsolute(name) ? '' : ' on PATH'}`)
}
```

**注意**：正则用 `$` 锚定在**末尾**判断有无扩展名——所以目录名里带点（`C:\Users\lmq\.local\bin\gh`）不会误判。

### ⑤ 打完上面两个补丁之后，还会撞到的下一层（**不是 bug，是设计**）

`gh` 找到之后，部署检查还会要求：**这个 gh 登录的是「智能体账号」，不是你自己的**。

> SETUP.md 原话：**A dedicated account, never the operator's own**: everything the agent writes appears under this name.

**处置**（三选一）：

| | 做法 | 代价 |
|---|---|---|
| A | 建个小号专门给它 | 要时间；建号本身不复杂，但要配 app / 权限 |
| B | 用你自己的账号 | 能用，但**它写的每条评论、每个 PR 都挂你名字**，且违反项目设计 |
| C | **不配 GitHub 源** | 面板常驻两条"系统问题"，**但不影响浏览与理解** |

**只想看看长什么样 ⇒ 选 C。** 那两条红字不挡任何东西。

> 2026-11-01 实测：`gh auth status` 已登录 `liumiaoqi`，scopes 含 `repo` / `workflow` —— **技术上够用**，只是不符合"独立账号"的设计要求。



## 二、装法（Windows 实测）

前置：Node ≥24（实测 v26.12.1 可用）、pnpm、git、Python、gh。**不需要 tmux**。

```bash
# 1. 克隆（fork 或上游都行；fork 记得加 upstream）
git clone https://github.com/<you>/dsh-superagent-prototype.git dsh-superagent
cd dsh-superagent
git remote add upstream https://github.com/deepseek-ai/dsh-superagent-prototype.git

# 2. 装依赖 + 构建
pnpm install
pnpm build          # 不要用 pnpm check，见 ① ②

# 3. 建 profile（--from-default-profile 是"替换"语义，重跑会覆盖配置）
dsh superagent --from-default-profile web --dump-config > /dev/null

# 4. 写配置（只需填 stateDir）
#    ~/.dsh/profiles/superagent/cordis.patch.yml
```

配置内容：

```yaml
- id: webserver
  config:
    host: 127.0.0.1
    port: 3082

- id: dsh-superagent
  config:
    stateDir: C:/Users/<you>/.dsh/state/superagent   # 打完补丁后可用盘符形式
```

```bash
# 5. 装 bundle 进 profile
dsh plugin --profile superagent add "link:/绝对路径/dsh-superagent"

# 6. 启动
dsh superagent
```

### ⚠️ 两个坑

**不要 `| tail` 或 `> file.log` 重定向启动输出。**
token URL 只打 stdout，而重定向到管道/文件时 Node 是**块缓冲**——进程活着的时候那行不 flush，你永远看不到。
**要拿 URL，必须在真终端里跑。**

**`link:` 协议不解析 peer deps**（pnpm 会 WARN）。实测 23 个 peer 依赖都能从 profile 继承的 bundle 解析到，不影响运行；若哪天出问题，改 `file:` 协议。

---

## 三、验证结果（2026-11-01）

| 项 | 结果 |
|---|---|
| `pnpm install` | ✅ 1 分 43 秒 |
| `pnpm typecheck` | ✅ |
| `pnpm build` | ✅ 5.5–7 秒，12 文件 693 KB |
| `import('./lib/index.js')` | ✅ 导出 `Config, Engine, apply, canRetry…` |
| `--dump-config` 合成 | ✅ 1408 行，bundle 在树里 |
| 面板启动 | ✅ `127.0.0.1:3082`（需带 token 的 URL，纯访问返回 401） |
| 插件加载 | ✅ 打补丁 ① 后 `异常` 消失，显示正常空状态 |
| 壁纸插件 bgwall | ✅ 复制 `plugins/bgwall.mjs` + `assets/wallpaper.jpg` 到本 profile，`--dump-config` 里能看到 `file:///…/bgwall.mjs` |

### 未端到端验证的一项

**④ 的 `PATHEXT` 补丁没跑过完整流程。** 只验到：

- `gh.exe` 确实在 PATH 上（`C:\Program Files\GitHub CLI\gh.exe`）
- 补全逻辑单独模拟能命中
- 编译产物里 `PATHEXT` 常量已就位

**没有**实际看到面板上那句 `gh is not an executable file on PATH` 消失——那需要重启服务，而重启在本机由操作者做。

**重启后如果还报同一句**，说明还有第三层（`workerProbe` 用 `runAsWorker(['env', 'PATH=…', …])`，Windows 上 `env.exe` 来自 Git 的 `usr\bin`，要确认它在 PATH 里）。


---

## 四、纪律

**每次改上游代码，都往这里加一条。** 否则三个月后你不知道 fork 和上游差在哪。

维护方式：

```bash
git diff upstream/main --stat        # 看差了多少
git log --oneline upstream/main -3   # 上游有没有新提交
```

上游若发了 Windows 适配，回来把 ① 撤掉。

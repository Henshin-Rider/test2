# Debug 记录：附件下载在 server 进程里偶发失败（已解决）

> 留档供以后排查类似 lark-cli 偶发问题时参考。
> 解决时间：2026-07-28

---

## 问题

飞书 Base 附件下载，`downloadLarkAttachment` 在 server 进程里（`npm run dev` 启动）**偶发失败**，导致带附件的 task 提交时附件上传这步报错。

### 现象
- 提交带附件的 task → 创建 TFS task 成功 → 下载附件这步偶发失败
- server 日志（失败时）：
  ```
  [downloadLarkAttachment] exited 1 outPath exists: false
  lark-cli exited 1: {
    "ok": false,
    "error": {
      "type": "api_error",
      "message": "API returned an empty JSON response body",
      "hint": "The endpoint may have returned an empty or non-standard JSON body. If it returns a file, rerun with --output."
    }
  }
  ```

### 核心矛盾
**同一个 token、同一份代码、同一个 server 进程**，下载结果偶发：
- 有时 `exited 0` + 文件落盘 → 完全成功
- 有时 `exited 1` + 文件**已**落盘 → lark-cli 误报（文件其实下好了）
- 有时 `exited 1` + 文件**没**落盘 → 真失败

而**命令行直接 `node` 测任意 token 几乎 100% 成功**，很难复现失败。

---

## 涉及的代码

### 下载函数：`server/services/larkService.js` 的 `downloadLarkAttachment`

调用方式（关键点）：
```js
const args = [
  LARK_RUN_JS, 'api', 'GET',
  `/open-apis/drive/v1/medias/${fileToken}/download`,
  '--params', params,        // {"extra":"{\"bitablePerm\":{\"tableId\":...,\"rev\":0}}"}
  '--as', 'user',
  '--output', fileName       // 相对路径，相对 cwd=outDir
]
const proc = spawn(process.execPath, args, {
  cwd: outDir,               // C:\...\Temp\lark_attachments
  env: { ...process.env, MSYS_NO_PATHCONV: '1' }
})
```

### 调用处：`server/routes/tfs.js` batch-create-tasks 路由
提交 TFS task 成功后，如果 `task.attachments` 非空，循环调 `downloadLarkAttachment` 下载，再调 `addAttachmentsToTask(taskId, localPaths)` 上传到 TFS。

### CLI：`C:/temp_tfs_build/TFSProxyCLI.exe` 的 `add-attachment` 命令
用 TFS 原生 `WorkItem.Attachments.Add()` 上传。单独验证可用，与本次问题无关。

### lark-cli 入口
`C:/Users/zihao.zhou01/AppData/Roaming/npm/node_modules/@larksuite/cli/scripts/run.js` 是 Node wrapper，内部 `execFileSync(bin, args, {stdio:'inherit'})` 调原生 `lark-cli.exe`。

---

## 飞书附件下载 API（已确认正确）

- **端点**：`GET /open-apis/drive/v1/medias/{file_token}/download`
- **高级权限 Base 必须带 extra 参数**：`{"bitablePerm":{"tableId":"xxx","rev":0}}`
- **scope**：`docs:document.media:download` 或 `bitable:app`
- **lark-cli 1.0.14 调用方式**：`lark-cli api GET /open-apis/drive/v1/medias/{token}/download --params '{"extra":"..."}' --as user --output <相对路径>`

---

## 排查过程（走过的弯路）

最初以为是「直接测成功 vs server 进程失败」的环境差异，逐一排除：

| 假说 | 验证方式 | 结论 |
|------|----------|------|
| server 用旧代码 | 对比进程启动时间 vs 文件 mtime | 早期日志确有旧进程，但重启后仍偶发失败，非主因 |
| `.env` 加载 / `LARK_RUN_JS` 值不同 | 加载 dotenv 后直接测 | 成功，无关 |
| concurrently / npm 子进程环境差异 | 用 `concurrently`、`npm run` 链路测 | 都成功，无关 |
| PATH 找不到 lark-cli | cmd.exe / where 验证 | 都能找到，无关 |
| stdio 配置（pipe/inherit/ignore） | 三种模式都测 | 都成功，无关 |
| TTY vs 非 TTY（stdout pipe） | `node ... \| cat` 模拟 | 都成功，无关 |
| HTTP 请求上下文本身 | 加 `/api/_diag/download` 端点 curl 触发 | 成功，无关 |
| `createTFSTask` 副作用 | 诊断端点先 spawn TFS CLI 再下载 / 在路由 createTask 后立即下载 | 成功，无关 |
| 成功/失败 context 字段差异 | 打印 pid/cwd/tmpdir/TMP/TEMP/USERPROFILE/execPath | **逐项完全一致** |

**所有环境/上下文假说全部排除。** 成功和失败时 server 进程的 context 字段完全相同。

### 决定性证据
在 `batch-create-tasks` 路由里 `createTFSTask` 之后**连续调两次同一个 token**，抓到：
```
[diag-after-createTask] 下载 6.jpg (ZjVsbpz0HoEuo8xpDF1ccGpAnqb)
exited 1 outPath exists: false   ← 第一次：失败
[diag-after-createTask] 失败

[tfs] 下载附件: 6.jpg (同一个 token)
exited 0 outPath exists: true    ← 第二次：成功
[tfs] Task: added 1 attachments
```

**同一个 token 第一次失败、第二次成功，间隔不到 1 秒。** 确认是 lark-cli 1.0.14 调飞书 download API 的偶发问题，不是环境问题。

---

## 根因

lark-cli 1.0.14 下载飞书 Base 附件有两个问题：

1. **偶发真失败**：返回 `empty JSON response body` 且 `--output` 文件不落盘。同一 token 重试通常即成功（飞书侧偶发或 lark-cli 内部 race）。
2. **误报失败**：文件已成功落盘，但 lark-cli 把二进制响应当 JSON 解析失败，`exited 1` 报同样的 empty JSON。

之前的代码 `if (code !== 0) reject(new Error(...))` 直接按退出码判定，两种情况都当失败丢弃，导致附件上传频频失败。

---

## 修复

`server/services/larkService.js` 的 `downloadLarkAttachment`：

1. **失败重试 4 次** —— 吸收偶发真失败（问题 1）
2. **以文件落盘且非空为准，忽略 lark-cli 退出码** —— 处理误报（问题 2）

```js
const MAX_ATTEMPTS = 4
for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
  try { unlinkSync(outPath) } catch {}
  // spawn lark-cli ...
  proc.on('close', code => {
    if (existsSync(outPath) && statSync(outPath).size > 0) {
      resolve({ ok: true })   // 文件落盘即成功，忽略 code
      return
    }
    resolve({ ok: false, code })
  })
  if (out.ok) return outPath
  // 否则记日志，进下一轮重试
}
throw new Error(`附件下载失败（重试 ${MAX_ATTEMPTS} 次）: ...`)
```

`execLarkCLI` 也加了失败诊断（打印 stderr/exit code），供 `fetchBitableFields` 等其他 lark 调用排查用。

---

## 验证结果

重启 server 后提交带附件的 task，日志显示重试机制工作正常：
- `2.jpg`：重试 4 次成功
- `201155.jpg`：重试 3 次成功
- `8.jpg`：重试 2 次成功
- 后续多个 task：第 1 次直接成功

提交 1/2/3 个带附件 task **全部成功**，附件都上传到 TFS（`addedCount: 1`）。

---

## 复现用的测试 token（都是 zihao.zhou01 的 Base 附件）

- `MGlrbcd4Domj9txRFLnc6Jatngd` → 201155.jpg
- `OkrubYmE2okKvixfGvZcxPzfn9f` → 7.jpg
- `T5w7b1k5Uoa8QhxpKaVcFHKFnDg` → 9.jpg
- `ZjVsbpz0HoEuo8xpDF1ccGpAnqb` → 6.jpg（抓到连续两次 fail/success 铁证的那个）

Base：appToken `QxFeb4XsoaJ21Qsks9ucBMEFnTh`，tableId `tblmcOYFLl0AulxA`

---

## 环境信息

- OS: Windows 10 Enterprise LTSC 2021
- Node: v24.11.0（`C:\Program Files\nodejs\node.exe`）
- lark-cli: 1.0.14（入口 `C:/Users/zihao.zhou01/AppData/Roaming/npm/node_modules/@larksuite/cli/scripts/run.js`，原生 exe 在 `.../bin/lark-cli.exe`）。最新版 1.0.78，升级可能修复偶发，但当前重试已兜底。
- 项目路径: `d:\task提交工具`
- server 启动: `npm run dev` → `concurrently "npm run server" "npm run client"` → `node server/index.js`（ESM）

---

## 经验教训

1. **lark-cli "命令行成功 / server 失败"类问题，先怀疑偶发重试即能解决**，别急着归因环境差异。本案例花了大量时间排除环境因素，最终发现是偶发。
2. **判定下载成功要用文件落盘，不能用 CLI 退出码** —— lark-cli 1.0.14 对二进制响应的退出码不可靠。
3. **抓偶发铁证的最有效手段**：在失败上下文里对同一输入连续调用两次，看是否一次 fail 一次 success。这比对比成功/失败的环境差异更直接。

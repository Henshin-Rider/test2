# Changelog

All notable changes to this project will be documented in this file.

## 2026-10-15

### Feat: Assign Owner 移到 task 级 + 可指派用户自动补全
- 飞书 Base 加「Assign Owner」人员列；分析后把文档里的飞书用户转成 LDAP 账号（`resolveAssignOwners` + `larkService.fetchLarkUserEmail`）
- CLI 加 `get-assignable-users` 命令；tfsService 新增 `searchAssignableUsers`（从 TFS 拉全量可指派用户 1.3 万+，内存+文件双层缓存，忽略 `_BU` 后缀模糊搜索）
- tfs 路由加 `GET /assignable-users`；TaskPreview 的 Assign Owner 改组合框：输入搜索 → 从清单命中选择，未匹配标红拦提交

### Feat: 只分析当前登录用户记录的 Task
- authService 登录时从 LDAP UPN 构造邮箱（`zihao.zhou01@united-imaging.com`），resolveLarkUser 改用邮箱搜飞书用户拿 open_id
- document.js 把 email/username/displayName 传进分析；移除 fetchDocumentData 里硬编码「Found By」过滤
- documentAnalyzer 过滤改多键匹配（邮箱 > 账号 > open_id > 中文名）；foundBy 关键词补「记录人/创建人/录入人」；列名匹配大小写不敏感

### Feat: reproducible 移到 task 级 + 从文档列提取
- 全局配置去掉 reproducible，改为 Task 预览里每个 task 单独选（Yes/No/Sporadic 下拉）
- 飞书 Base 加「Reproducible」单选列；分析时直接取该列枚举值（列值优先，AI 提取兜底），提交用 task 自身值

### Fix: 提交 TFS 字段校验失败
- Activity 字段通过 API 设置任意 allowed 值都被 TFS 报 InvalidNotEmpty，改为不再传 --activity（留空走 TFS 默认）
- 恢复 assignOwner 自动追加 BU 后缀（`yuanzhi.sun` → `yuanzhi.sun_CO`），解决 Assigned To InvalidListValue

### Change: 清理死代码 + lark-cli 输出解析增强
- 删除 TaskList.jsx / TaskSubmit.jsx（提交链路已合并进 TaskPreview）
- larkService 抽 `parseLarkOutput` 兼容 JSON/markdown 代码块等包装，export `execLarkCLI`；base record-list 加 --json、分页上限改 200

## 2026-10-10

### Feat: 飞书机器人模板化录入 Task
- 新增 `server/bot/botServer.js` 常驻进程，订阅 `im.message.receive_v1` 事件
- 用户在单聊发任意消息 → 回「当前 TFS 配置 + task 模板」；发「换项目」→ 编号点选 AreaPath/Found in Version/Product/Phase/Activity 写进 user-preferences；按模板填好发回 → `base +record-batch-create` 建记录（Found By=发送人、Found Date=今天、Status=待提交）
- 复用现有「开始分析 → 预览 → 提交」链路，回填等零改动
- `writePreferences`/`readPreferences` 从 routes/tfs.js 抽到 tfsService 层共享；larkService 导出 `execLarkCLI` + 新增 `resolveOpenIdToUsername`
- `npm run bot` 独立启动

## 2026-07-29

### Feat: 使用说明页
- header 加「使用说明」按钮，点击后右侧显示 `HelpPage` 组件
- 内容 copy 自飞书文档（`lark-cli docs +fetch` 拉取），含工具简介/使用流程/表格结构/常见问题
- 右侧渲染优先级：使用说明 > 配置 > 预览
- 使用说明第一步删掉「Found By 过滤」那条

### UI: 登录页文案 + 配色
- 卡片下方加 LDAP 说明文案
- 登录按钮改横向长方形，背景 `#04485F`，hover `#033645`
- 「开始分析」按钮同色 `#04485F`

### Change: 非默认文档分析下线
- `DocumentInput.jsx` 移除自定义 URL 输入和「分析文档」按钮（代码保留注释，后端能力仍在）
- 「分析默认文档」按钮文字改为「开始分析」

### Fix: 点 task 直接转预览
- `handleSelectTask` 清 `editingConfig`/`showHelp`，点左侧 task 右侧直接转预览
- TFS配置页「返回预览」→「返回」

### Feat: 分析开始显示 AI 模型名
- `analyzeDocument` 开头发 `step:'model'` 事件，显示「AI 模型：DeepSeek-V4-Flash」
- 前端 `STEP_LABELS`/`STEP_ORDER` 加 `model` 项，model step 保持 active 始终显示

## 2026-07-28

### Feat: 配置常驻主页右侧 + 配置未完成拦截
- `App.jsx` 右侧分两态：配置未完成（7 必填项任一空）→ 显示 ConfigManager + 黄色提示；配置完成 → 显示 Task 预览
- 新增 `isConfigComplete()` 判定 7 个必填项（areaPath/iterationPath/product/assignOwner/phase/activity/reproducible）
- 删掉整页 `showConfig` 切换；header 保留「配置」按钮，点一下临时切回配置（`editingConfig`），保存成功自动回预览
- 没填配置就点「分析默认文档」：后台照常分析，左侧顶部黄色警告条提示先填配置
- `DocumentInput.jsx` 新增 `onAnalyzeStart` 回调通知父组件

### Feat: 去掉「验证全部」环节，改为提交前空值检查
- 删掉 TaskPreview 的 AI 验证逻辑（`handleValidateAll`/`validationMap`/`allValidated`/AI 高亮）及 App.jsx 对应状态和 `validateTask` import
- 提交前检查 8 个必填字段（title + swVersion/hwEnvironment/historyAction/expectedResult/error/foundDate/foundBy），扫描所有要提交的 task，任一空就拦下
- 拦下时弹窗只报数量（N 个 Task 共 M 个字段空），不列具体字段
- 空字段实时红色高亮（`data-empty` 属性），填入即消失
- 左侧 task 列表空字段标识：橙色「AI」→ 红色「补全」+ 红色竖条
- 后端 `/api/task/validate` 路由保留（前端不再调）

### Feat: AI 生成 Task 改为只提取不补全
- `documentAnalyzer.js` Pass3 prompt 从「生成完整 Task」改为「只提取，不补全不推断」
- 原文没有的字段返回空字符串，禁止编造/填占位词
- `fallbackTaskGeneration` 的 `hwEnvironment: 'N.A.'` → `''`

### UI: 登录页美化
- `Login.jsx` 接上 CSS 里闲置的 `.login-page`/`.login-card`/`.login-field`/`.login-submit` 样式（之前 class 名对不上）
- 卡片加宽 420px，输入框/按钮放大，加副标题

## 2026-07-22

### Feat: 外层平台集成最终回到 LDAP 域账户登录
- 工具被集成到外层平台（飞书 OAuth 入口），最终方案：外层只做入口，工具内自己走 LDAP 域账户登录，让 TFS Created By 记录真实用户
- 方案演变：先试 verify-token（外层 token 换身份，email/user_id 为空）→ 想加 scope 拿 email（卡 open_id 跨 app 不可用）→ 最终回 LDAP
- CLI `create-task` 新增 `--tfs-user`/`--tfs-password`，`InitTFS` 有凭据时用 `NetworkCredential` 以用户身份连 TFS

## 2026-07-14

### Feat: 飞书 Base 结构调整 + 附件上传到 TFS
- Base 删 Description 列（和 Title 重复），新增 Additional Attachment 单选列（Yes/No，默认 No）+ Attachments 附件列
- CLI 新增 `add-attachment` 命令（TFS 原生 `WorkItem.Attachments.Add`），提交流程：创建 task → 下载飞书附件 → 上传 → 清理临时文件
- 下拉数据切真 TFS（`MOCK_TFS=false`），AreaPath 过滤只保留 CO 分支（12 条）避免 2000+ 条加载慢
- 删掉单 Task 解析入口 + IVR 文件版本提取；TaskPreview 切换 task 时重置 state

## 2026-07-13

### Feat: LDAP 真域控 + TFS ID 回填 + 性能优化
- 探测真域控 `cnshd004.united-imaging.com:389`，UPN `{user}@united-imaging.com`，关掉 dev bypass
- 提交成功后自动写 TFS ID + Status 改「已提交」回填飞书（`backfill-tfs-id`，record_id_list 挂 `_recordId`）
- AI 模型换 DeepSeek-V4-Flash（校验 61s→14s）；Pass1 列识别默认跳过 AI；校验 prompt 精简 + max_tokens
- AI 高亮修复：`parseAIResponse` 对比 `correctedData` vs `originalData` 标记，不依赖 AI 自报 fieldChanges

## 2026-06-23

### Fix: 去掉 description 中自动追加的日志路径
- **问题**: CLI `CreateTask` 第二次 Save 时把 `"日志路径：\\dataserver03\..."` 拼到 description 末尾
- **修复**: `myBug.Description = description + strBugLogInfo` → `myBug.Description = description`，只保留原始 description
- 日志目录创建和 robocopy 复制逻辑仍保留（只是不写入 description）
- **已重新编译**: `TFSProxyCLI.exe`

### Fix: CLI timeout 导致 Task 提交失败
- **问题**: 部分 Task 提交时 CLI 进程 30 秒内未完成，导致 `CLI timeout` 错误
- **根因**: 每个 CLI 进程需要重新 `InitTFS()` 连接 TFS 服务器，加上 create-task + 第二次 Save，网络慢时 30 秒不够
- **修复**: `execCLI` timeout 从 30s → 60s

### Fix: 提交后无反馈，直接回到主页
- **问题**: Task 全部提交成功后 `onSubmitComplete` 直接触发 `setView('input')`，用户看不到结果
- **修复**:
  - TaskSubmit: 去掉提交成功后的自动 `onSubmitComplete` 调用
  - TaskSubmit: 提交完成后显示结果面板（成功/失败详情 + Task ID），底部改为"返回主页"按钮
  - 未提交时仍显示"返回编辑" + "提交"按钮；提交完成后只显示"返回主页"

### Fix: UTF-8 description 乱码 (CLI 端 — 第二次修复)
- **问题**: Node.js 端 spawn 修复后仍然乱码（`鐢ㄦ埛鎵撳紑` 特征），说明问题在 C# CLI 端
- **根因**: `Console.In.ReadToEnd()` 和 `Console.WriteLine()` 在中文 Windows 上使用默认 GBK 编码读写 stdin/stdout，Node.js 写入 UTF-8 字节被 CLI 用 GBK 解码导致乱码
- **修复**: CLI `Main()` 入口处强制设置 `Console.InputEncoding = Encoding.UTF8` 和 `Console.OutputEncoding = Encoding.UTF8`
- **已重新编译**: `TFSProxyCLI.exe`

### Fix: UTF-8 description 乱码 (Node.js 端 — 第一次修复)
- **问题**: 批量提交 TFS Task 后，description 中的中文变成乱码（如 "杩涘叆璐ㄦ浮" ）
- **根因**: `execCLI` 使用 `spawn(..., { shell: true })` 通过 cmd.exe 执行，中文 Windows cmd.exe 默认 GBK (CP936) 编码，UTF-8 stdin 被转码损坏
- **修复**:
  - `tfsService.js`: 去掉 `shell: true`，改为 `spawn(CLI_PATH, argArray)` 直接 spawn
  - `execCLI` 参数从字符串改为数组，去掉手工 `escapeArg` 拼引号
  - `child.stdin.write(stdinInput, 'utf8')` 明确指定 UTF-8 编码
  - 清理无用代码: `exec/promisify/os` 导入、`tfsInitialized/mattInitialized` 变量

### Feat: 新增 Phase / Activity / Reproducible TFS 字段
- **CLI**: 新增 `get-phases`, `get-activities`, `get-reproducibles`, `get-task-fields` 命令
- **CLI**: `create-task` 新增 `--phase`, `--activity`, `--reproducible` flag 参数
- **CLI**: `CreateTask` 方法签名扩展，设置 `Fields["Phase"]`, `Fields["Activity"]`, `Fields["Reproducible"]`
- **CLI**: 已重新编译 `TFSProxyCLI.exe`（csc.exe → temp_build → 复制回项目目录）
- **Service**: 新增 `getPhases()`, `getActivities()`, `getReproducibles()` + mock 数据
- **Service**: `createTFSTask` 接收并传递 phase/activity/reproducible 到 CLI args
- **Routes**: 新增 `GET /phases`, `/activities`, `/reproducibles` 端点
- **Routes**: `create-task`, `batch-create-tasks`, `user-preferences` 支持新字段
- **Routes**: `user-preferences` GET/POST 默认值包含 phase/activity/reproducible
- **API**: 新增 `getPhases()`, `getActivities()`, `getReproducibles()`, `loadTFSConfig()`
- **前端**: ConfigManager + TaskSubmit 新增 Phase/Activity/Reproducible 下拉选择框
- **TFS 实际字段名验证** (通过 `get-task-fields` 命令):
  - Phase: referenceName `UI.Bug.Phase`, 可选值: [IB], [IB-After], [Integration], [Pilot], [Pre-SIT], [Pre-ST], [Pre-UT], [Production Evaluation], [SIT], [ST], [UT], Others
  - Activity: referenceName `Microsoft.VSTS.Common.Activity`, 可选值: Design, Improvement, Management, Other, TDOC, Testing, UDOC
  - Reproducible: referenceName `UI.Bug.Reproducible`, 可选值: Yes, Sporadic, No（注意有 Sporadic）

### Feat: IterationPath 标签重命名为 "Found in Version"
- ConfigManager / TaskSubmit 中 "IterationPath" 标签改为 "Found in Version"
- 功能不变，TreeSelect 组件仍正常工作

### Perf: 配置页/提交页加载速度优化
- **问题**: 每次进入配置页/提交页，7个并行 API 请求 + 每个请求 spawn CLI init-tfs + spawn CLI 查询 = 14次进程启动
- **修复**:
  - 服务端 `dropdownCache`: 5分钟 TTL 内存缓存，首次查询后命中缓存不再 spawn CLI
  - 新增 `GET /api/tfs/config` 批量端点，一次请求返回所有下拉数据 + preferences
  - 前端从 `Promise.all(7个请求)` 改为 `loadTFSConfig()` 单次请求
  - 各下拉端点去掉 `ensureTFSInitialized()`，由缓存层处理
  - 删除旧的重复函数声明（缓存版 `getCachedDropdown` 替代了旧的直接 CLI 调用版）

### UI: Phase/Activity/Reproducible 改为纯下拉框
- ConfigManager / TaskSubmit 中 Phase/Activity/Reproducible/AreaPath/Product 全改为 `<select>`，不再有 fallback `<input>`
- `renderField` 拆分为 `renderSelect` 和 `renderInput`，职责清晰

### UI: 去掉 "Task 默认值" section
- ConfigManager 中删除 "SW Version（默认）" 和 "HW Environment（默认）" 输入框及整个 section
- TaskSubmit 中同样删除 SW Version / HW Environment row

### UI: AssignOwner placeholder 简化
- ConfigManager: "如 zihao.zhou01" → "AssignOwner"
- TaskSubmit: "输入 AssignOwner" → "AssignOwner"

## 2026-06-22 (之前 session)

### 初始版本 v1.0.0
- React 18 + Vite 前端 SPA
- Express 后端服务
- 飞书文档解析 (docx/bitable/sheet/wiki via lark-cli)
- AI 三步分析引擎 (documentAnalyzer): Pass1 列识别 → Pass2 问题发现 → Pass3 Task 生成
- SSE 流式分析 (`analyze-stream` 端点)
- TaskList 编辑/选择 UI
- TaskSubmit 提交 UI
- ConfigManager 配置页
- TFS CLI wrapper (TFSProxyCLI.exe): init-tfs, create-task, get-area-paths, get-iteration-paths, get-products
- AI 校验、IVR 文件版本提取
- description 通过 stdin 传递（避免命令行换行/转义问题）
- `formatTaskDescription()` 自动格式化 object → Task Sample string
- AssignOwner 自动追加 BU suffix (如 "zihao.zhou01" → "zihao.zhou01_CO")

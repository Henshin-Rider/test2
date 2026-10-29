# task 提交工具 · 部署 runbook（给接收方 AI）

> 你拿到的是一个打包好的文件夹（里面是完整的 task 提交工具：前端 `src/`、后端 `server/`、TFS CLI `tfs_api/TFSProxy/CLI/`、以及 `.env`、`config/`、`config/user-preferences/` 等）。**读完目录后，按下面 step 一步步做就能跑起来。**
>
> 运行环境：**Windows**（TFS CLI 是 .NET Framework 的 Windows exe）。目标机器需要能访问：内网 TFS `http://tfs:8080/tfs`、LDAP 域控、公网飞书、AI 大模型 API。

## Step 0 — 解压 + 确认目录

解压后先确认这些关键文件都在（缺了说明打包不全）：

```
.env                        # 环境变量（含 AI key、LDAP、路径等）
package.json                # 依赖与脚本
config/projects.json        # AI 模型配置 + 项目配置
config/user-preferences/    # 各用户的 TFS 路径配置
server/index.js             # Express 后端入口
server/bot/botServer.js     # 飞书机器人（可选进程）
tfs_api/TFSProxy/CLI/TFSProxyCLI.exe   # TFS 命令行（提交/拉下拉用）
```

## Step 1 — 装 Node.js

- 装 Node.js LTS（>=18），确认 `node -v`、`npm -v` 正常。

## Step 2 — 装依赖

```bash
# 在项目根目录执行
npm install
```

> 如果打包时把 `node_modules/` 也一起打进来了，且解压后 `node_modules` 完整，这步可以跳过；但跨机器拷贝 node_modules 偶有二进制不兼容（vite 的 esbuild 等），**稳妥起见还是删掉重装一次**。

## Step 3 — 装 lark-cli（飞书操作全走它）

```bash
npm install -g @larksuite/cli
```

装完后找到 lark-cli 的入口 JS（下面改 .env 要用），一般是：

```
C:/Users/<你的用户名>/AppData/Roaming/npm/node_modules/@larksuite/cli/scripts/run.js
```

可以用 `where lark-cli` 定位，再顺着 `node_modules/@larksuite/cli/scripts/run.js` 找全路径。

## Step 4 — 改 .env 里两个机器相关的路径

打开根目录 `.env`，**只改这两个**（其余 COPY 过来别动）：

1. `TFS_CLI_PATH` —— 旧机器写的是 `C:/temp_tfs_build/TFSProxyCLI.exe`。二选一：
   - 直接删掉这一行（代码会回退到仓库里的 `tfs_api/TFSProxy/CLI/TFSProxyCLI.exe`）；或
   - 改成 `<项目目录绝对路径>/tfs_api/TFSProxy/CLI/TFSProxyCLI.exe`。
2. `LARK_RUN_JS` —— 旧机器写的是 `C:/Users/zihao.zhou01/AppData/.../run.js`，改成你 Step 3 找到的那条**你机器上的实际路径**。

其它项（`AI_API_KEY`/`AI_BASE_URL`/`AI_MODEL`/`LDAP_URL`/`LDAP_USER_DN_TEMPLATE`/`MOCK_TFS`/`OUTER_BASE`）保持原值。

## Step 5 — 装 TFS 客户端程序集（最易漏，漏了提交会报错）

`TFSProxyCLI.exe` 运行期依赖 `Microsoft.TeamFoundation.*.dll`，这组 DLL 由 **Visual Studio / Team Explorer** 装进 GAC，**不随 exe 走**。所以：

- 新机器**装一个带 Team Explorer 的 Visual Studio**（版本和旧机器一致最好，旧的是 VS2010 那个年代的 TFS 客户端）；或
- 把那几个 DLL（`Microsoft.TeamFoundation.Client.dll`、`Microsoft.TeamFoundation.WorkItemTracking.Client.dll` 等）从旧机器 GAC 拷到 `TFSProxyCLI.exe` 同目录。

**验证**：命令行跑一次
```bash
tfs_api/TFSProxy/CLI/TFSProxyCLI.exe get-products
```
能吐出 JSON 数组（不是报「Could not load file or assembly」）就说明这步 OK。

## Step 6 — 配置 lark-cli 的 app + 授权 user token

> 注意：app 的 App Secret 和 user token 是**机器级/设备级**的，打包的文件夹里没有，必须在这台机器重新配。

1. 配 app（需要 App Secret，从飞书开放平台后台拿，别贴进聊天/日志）：
   ```bash
   echo "<App Secret>" | lark-cli config init --app-id cli_a955f199bf7adbb3 --app-secret-stdin
   ```
2. 授权 user token（走设备码，浏览器点确认即可）：
   ```bash
   lark-cli auth login --scope "auth:user.id:read"
   ```
   （这个 scope 是「只续期、不加新敏感权限」；lark-cli 增量授权，不会删掉之前的 scope。）
3. 验证：`lark-cli auth status` 显示 `appId: cli_a955f199bf7adbb3`、user token `valid`。

## Step 7 — 网络连通性自检

- TFS：`curl http://tfs:8080/tfs`（或前面 Step 5 的 `get-products` 能出 JSON 即可）。
- LDAP：`.env` 里 `LDAP_URL` 指向的域控能连上。
- 飞书：`lark-cli auth status` 正常就说明飞书 API 通。

## Step 8 — 启动

开发模式（前后端一起）：
```bash
npm run dev        # 前端 vite(5173) + 后端(3102)
```

生产模式：
```bash
npm run build      # 产出 dist/
node server/index.js   # 只跑后端，serve dist
```

如果还要飞书机器人（可选项）：
```bash
npm run bot        # server/bot/botServer.js，订阅飞书消息
```

## Step 9 — 端到端验证

1. 浏览器打开前端，用**域账户登录**（LDAP）。
2. 进「TFS 配置」页，确认 AreaPath / Product / Phase / Activity 下拉能拉出来（证明 TFS 网络 + SDK 正常、lark-cli 正常）。
3. 「开始分析」默认文档 → 预览 → 提交。默认 `MOCK_TFS=true` 会返回假 ID；要连真 TFS 就临时把 `.env` 的 `MOCK_TFS` 改成 `false`。
4. 回飞书看提交结果、TFS ID 有没有回填。

## 易错清单（做完过一遍）

- [ ] `.env` 的 `TFS_CLI_PATH`、`LARK_RUN_JS` 改成新机器路径了吗？
- [ ] TFS SDK 装了吗？`get-products` 能出 JSON 吗？
- [ ] `lark-cli config init` + `auth login` 做了吗？`auth status` 是 valid 吗？
- [ ] `npm install` 成功了吗？
- [ ] TFS / LDAP 网络通了吗？
- [ ] 浏览器登录（LDAP）能通过吗？
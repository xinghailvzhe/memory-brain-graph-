# memory-brain-graph 工作区开关（workspace gate）补丁说明

> 基于 **v0.5.2** 源码。目标：让「脑图」按钮 / 「脑图」标签页只在**指定工作区**的对话窗口里出现，
> 而不是所有工作区、所有会话都注入；并在脑图页面右侧提供一个紧凑的勾选面板来管理这份名单。

---

## 1. 需求（用户原话）

> 「它不在每个工作区都启用，它需要选定了某个工作区，它需要把那个工作区选定，它才能这个工作区才能启用这个插件。还有就是它不在所有对话窗口都启用。它只有在选定工作区的对话窗口才能启用。」

即：

1. 插件按工作区启用，需要一个「选定工作区」的动作；
2. 只有被选定工作区的会话窗口才显示插件 UI；
3. 其他工作区不显示。

---

## 2. 设计

### 2.1 数据：`assets/workspace-gate.json`

```json
{ "workspaces": ["F:\\output\\newgame-管理区"], "updatedAt": 1793798747711 }
```

- 文件不存在或 `workspaces` 为空 = **默认只启用「记忆库所在工作区」**（由 `--db` 反推），不会出现「默认全开」。
- 放在 assets 下，跨重启持久。

### 2.2 HTTP 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/gate` | 返回 `{workspace, workspaces}`，`workspace` = 当前库所在工作区 |
| POST | `/api/gate` | body `{workspaces: [...]}`，逐项 `os.path.isdir` 校验、去重后写盘，返回 `{ok, workspaces}` |
| GET | `/api/workspaces` | 返回全部工作区 `[{path, enabled, current}]`；枚举方式复用已有的 `_recent_workspaces()`（读 `session_projcache` 里每个会话的 `identity.cwd`），`limit=500`，再确保当前库工作区在列 |

所有接口带 `Access-Control-Allow-Origin: *`、`Cache-Control: no-store`，方便 GUI 侧跨端口 fetch（8787 ← GUI 任意端口）。

### 2.3 客户端：在 `conversation.composer.dock` 挂隐形探针

**踩到的坑**：宿主渲染 `conversation.input.right` / `conversation.view` 时是 `renderSlot(name, {})`，
**props 是空对象**，拿不到会话/sp工作区信息；`conversation.composer.dock` 的组件能收到一批注入的 hooks
（`useSessions / useSession / useSessionStatus / useWorkspaces / useSessionAccess / sessionId / useChat …`）。

做法：

1. 在 `conversation.composer.dock` 注册一个渲染 `null` 的探针组件；
2. `sessionId` 取 `props.sessionId`，回退 `useSessions(s => s.current)`；
3. 工作区取 `useSessions(s => s.byId)[sessionId].cwd`（实测该字段就是工作区绝对路径）；
4. 结果写进模块级 store，`脑图` 按钮与标签页订阅它：
   - 名单为空 → 显示（等价于未启用 gate）；
   - 拿不到 cwd → 显示（避免误伤），按钮 title 里会带 `cwd=?` 方便排查；
   - 命中名单 → 显示，否则 `return null`。

> **给作者的建议**：`byId[id].cwd` 是内部状态字段，若能官方暴露一个 `useCurrentSession()`（或给会话级 slot 注入
> `sessionId`/`cwd`），客户端插件就不必依赖 `composer.dock` 做探针，实现会更干净。

### 2.4 页面面板（`index.html`）

在 `</body>` 前注入一段自包含的 HTML+JS：

- 固定右下（`right:16px; top:150px; width:232px`），深色半透明卡片，复用页面既有配色；
- 列出全部工作区（显示路径末两段，`title` 里是全路径），当前库所在工作区绿色高亮；
- 勾选 → 「保存」→ `POST /api/gate`；「刷新」重新拉 `/api/workspaces`；
- 顶部显示 `已启用/总数`；
- **可收起**：点标题行收起面板 → 右上角只剩一个小按钮「⚙ 工作区」，点它展开；收起状态记在 `localStorage`。

---

## 3. 改动清单

| 文件 | 改动 |
|---|---|
| `assets/server.py` | 新增 4 个 helper（`_mbg_ws_gate_file / _mbg_current_workspace / _mbg_gate_read / _mbg_gate_write / _mbg_all_workspaces`）+ `do_GET` 里 `/api/gate`、`/api/workspaces` + `do_POST` 里 `/api/gate`。见 `server-workspace-gate.diff` |
| `client.js` | 重写：gate 拉取、探针、按工作区显隐。见 `client-workspace-gate.diff`（整文件替换更简单，`patched/client.js` 可直接用） |
| `assets/index.html` | `</body>` 前注入右侧面板。见 `index-workspace-panel.diff`（约 3.5KB，单个 hunk） |
| `assets/workspace-gate.json` | 新增数据文件（可选，缺省时按默认规则工作） |

`base/` 是干净的 v0.5.2 源文件，`patched/` 是应用后的完整文件，便于直接 diff 对照。

---

## 4. 应用方式

任选其一：

1. **补丁**：`git apply server-workspace-gate.diff index-workspace-panel.diff`，再整体替换 `client.js`；
2. **整文件**：用 `patched/server.py`、`patched/client.js` 覆盖，`index.html` 按 diff 插入面板段；
3. 重装插件后，页面刷新一次即可看到面板；DSH 会话里的按钮要刷新页面才按新名单显示（客户端在加载时取一次 gate）。

---

## 5. 我们这边的额外加固（**不属于本特性，仅供参考**）

本机为了「绝不读其它工作区」还改了 `server.py` 的库发现逻辑，没有放进这个补丁：

- `DBS` 钉死：有 `--db` 就只用它，不做多库聚合；
- `_scan_disk_dbs()` 直接返回空；`_default_dbs()` 不再扫「最近会话工作区」；
- 启动时校验库路径必须在 `allowed-db-roots.txt` 白名单内，否则 `exit 2`。

如果作者愿意，这几项也可以作为上游选项（例如配置 `singleDb: true`、`allowedDbRoots: []`），
比用户各自打补丁更稳。

---

## 6. 已知边界

- 探针依赖 `conversation.composer.dock` 的注入 props；若宿主后续改动该 slot 的注入项，需要同步调整 `pickCwdFrom` 的候选字段。
- `index.html` 的面板是纯前端注入，未接入页面自身的组件系统；若官方提供设置页/配置 schema，建议改为正式设置项。
- 名单比较做了大小写与斜杠归一（Windows 友好），不做软链接/短路径（8.3）解析。

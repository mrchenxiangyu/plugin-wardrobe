# 🧥 plugin-wardrobe 兼容规范（SPEC）

> 供 DSH 用户插件作者对照编写插件，使插件能被「插件衣帽间」（plugin-wardrobe）统一管理：
> 上架（publish）/ 更新（update）/ 穿脱（toggle）/ 登记（registry）。
> 核心原则：**插件是衣服** —— 可穿可脱、不影响 DSH 主体、数据与代码分离。

## 1. 目录与文件布局

插件仓库根目录应包含：

```
<插件仓库>/
├── package.json          # 包元数据（版本号、入口、DSH 挂载配置）
├── lib/
│   ├── index.js          # Host 半边（Cordis 插件模块：apply/inject/name）
│   └── client.js         # Client 半边（预构建 web bundle，可被 dsh.client 扫描加载）
├── cordis.patch.yml      # 随包分发的挂载补丁（可选，供 dsh plugin add 使用）
├── install-<名>.ps1      # 幂等安装脚本（本机直装用）
├── CHANGELOG.md          # 更新日志（强制）
└── README.md             # 安装/使用说明
```

- Host 半边 = 标准 Cordis 插件：导出 `{ apply(ctx), inject: [...], name? }`；工具用 `ctx.tools.register(defineTool(...))`，Client→Host RPC 用 `TypertRemoteService` 或等效网关。
- Client 半边 = **预构建 bundle**（不依赖源码编译）：入口/按钮/面板直接可用，DSH 扫描包即可加载，无需重新构建 web 应用。

## 2. package.json 要求

```jsonc
{
  "name": "@<scope>/<plugin-name>",   // scoped 包名，全局唯一
  "version": "0.2.1",                 // 语义化版本（见 §5）
  "type": "module",
  "main": "lib/index.js",
  "exports": {
    ".": "./lib/index.js",
    "./client": "./lib/client.js",
    "./package.json": "./package.json"
  },
  "files": ["lib", "cordis.patch.yml", "package.json"],
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },   // 挂载补丁（供 dsh plugin add）
    "client": {
      "platform": "web",
      "inject": ["@deepseek-ai/dsh-client-runtime", /* 依赖的 client 服务包 */]
    }
  },
  "dependencies": {
    "@deepseek-ai/cordis": "^4.0.1",
    "@deepseek-ai/dsh-typert-protocol": "^0.1.0-rc.6",
    "@deepseek-ai/dsh-tools": "^0.1.0-rc.6"
  }
}
```

- 依赖只声明运行所需的最小集；DSH 自带包无需打包安装。
- `dsh.client.inject` 列出 Client 半边用到的服务包（如 sidebar/settings/workspace 等）。

## 3. 安装脚本规范（install-*.ps1）

目标：把插件装进 `$DSH_HOME\profiles\node_modules\<包名>` 并写挂载行；**幂等可重复运行**。

必做项：

1. **解析 profile**：`$ProfileRoot = "$env:USERPROFILE\.dsh\profiles\web"`（可用参数覆盖）；校验 `cordis.yml` 存在。
2. **定位 node_modules**：优先 `$DSH_HOME\profiles\node_modules`（兄弟共享），否则 `profiles\<profile>\node_modules`。
3. **清理旧包名遗留**：若曾用别的包名，删除旧目录（如 `@cindy\taskman`）。
4. **重装陷阱**：目标 `lib` 已存在时，`Copy-Item -Recurse` 会把新文件嵌成 `lib\lib\` 嵌套、顶层仍是旧文件。
   **必须先删除整个目标包目录再复制**（或先删目标 `lib`）。
5. **写挂载行**（`cordis.patch.yml`）：
   - 若文件恰好是 `[]`（纯空数组）：整体替换为注释 + insert 块；
   - 若已含 `name: '<包名>'`：跳过（幂等）；
   - 否则：**重写**为「注释 + insert 块」，**不要**把 insert 追加到 `[]` 之后。
6. 输出校验提示：`Restart DSH to apply` + 数据迁移说明。

挂载行格式（穿脱开关）：

```yaml
- insert:
    - id: <插件id>
      name: '<包名>'
```

## 4. 数据与主体分离（红线）

- 插件数据（`.taskman/`、配置记忆文件等）放**用户指定目录**，可读 JSON/Markdown、可 git 追踪、可整体迁移。
- 插件只写：`$DSH_HOME\profiles\node_modules\<包名>`、profile 的 `cordis.patch.yml`、`$DSH_HOME\plugins.json`、用户数据目录。
- **禁止**：修改 DSH 主体安装目录、apps/web、出厂预设。
- 卸载（脱）只删挂载行与包目录，**不删用户数据**。

## 5. 版本号（semver 映射）

| 变更类型 | 示例 |
| --- | --- |
| 修复 / 界面调整 | 0.2.0 → 0.2.1 |
| 新增功能（向后兼容） | 0.2.1 → 0.3.0 |
| 破坏性变更 / 数据格式不兼容 | 0.3.0 → 1.0.0 |

每次提升版本必须同步更新 CHANGELOG.md（见 §6）。

## 6. CHANGELOG 约定（强制）

仓库根 `CHANGELOG.md`，首行写明约定；每次发布追加：

```markdown
# 更新日志（Changelog）

> 约定：每次发布都更新本文件，说明「版本 → 版本」的变更内容与升级步骤。

## [0.2.1] - YYYY-MM-DD（当前）

### 变更（新增 / 修复 / UI 调整）
- ...

### 从上一版本（0.2.0）升级
1. git pull（或重新 clone）
2. 重新运行 install-<名>.ps1（本次改了 lib/，必须覆盖）
3. 重启 DSH
> 数据兼容性：完全兼容 / 需要迁移 / 需要重新选择根目录（写明确）
```

其他设备用户只需按「从上一版本升级」操作。

## 7. 登记（plugins.json）

安装时（或首次管理时）在 `$DSH_HOME\plugins.json` 登记：

```jsonc
{
  "plugins": [
    {
      "id": "<插件id>",                  // 与 patchId 一致
      "packageName": "@<scope>/<name>",
      "repo": "D:\\path\\to\\repo",
      "repoUrl": "https://github.com/<owner>/<name>.git",
      "installScript": "install-<名>.ps1",
      "patchId": "<插件id>",
      "sourceLib": "lib",
      "installedVersion": "0.2.1"
    }
  ]
}
```

## 8. 发布前自检清单（作者）

- [ ] `package.json` 版本已按 §5 提升
- [ ] `CHANGELOG.md` 已追加（含升级步骤与数据兼容性说明）
- [ ] `lib/index.js`、`lib/client.js` 为最终版（预构建）
- [ ] 安装脚本在**已装过旧版**的机器上重跑不会产生 `lib\lib` 嵌套
- [ ] 挂载行格式正确（`- insert:` 独立成行，name 用单引号包包名）
- [ ] 数据目录与代码分离；未触碰 DSH 主体
- [ ] 已提交并推送（`git push origin HEAD`）

## 9. 更新后四连校验（更新方执行）

1. 版本号：已装 `package.json` 与仓库一致
2. 哈希：`Get-FileHash` 对比已装 `lib\*` 与仓库 `lib\*` → MATCH
3. 挂载行：`Select-String -Path <patch> -Pattern "<包名>"` 命中
4. YAML：node 用 DSH 同款解析器 load patch 文件，结构为 `[{insert:[{id,name}]}]`

全部通过后重启 DSH 生效。

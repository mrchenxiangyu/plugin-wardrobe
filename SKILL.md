---
name: plugin-wardrobe
description: 插件衣帽间 —— 通用的 DSH 用户插件更新管理技能。管理按「源码仓库 + 本地安装」模式交付的 DSH 插件（如 cindy-taskman）：发布新版本到 git、从 git 更新本地插件、穿脱开关（增删挂载行）、维护 CHANGELOG 与版本号、登记插件清单。不绑定任何具体插件，适用于所有遵循同一规范的 DSH 用户插件。
---

# 插件衣帽间（plugin-wardrobe）

DSH 用户插件 =「衣服」：可以穿（挂载）、可以脱（卸载），不影响 DSH 主体。
本技能统一管理这类插件的完整生命周期：

1. **上架（publish）**：版本号提升 + CHANGELOG + 提交 + 推送 GitHub
2. **更新（update）**：拉取仓库 → 安装到本机 DSH profile → 校验 → 提示重启
3. **穿脱（toggle）**：增删 profile 补丁文件的挂载行（穿=加行，脱=删行）
4. **登记（registry）**：`$DSH_HOME/plugins.json` 记录每个已装插件的映射（仓库/包名/路径/安装脚本），多插件共存互不干扰

## 0. 已核实的环境事实（先读，勿臆测）

- DSH 用户目录 = `$DSH_HOME`：**以进程与用户级环境变量为准**（本机为 `D:\dsh`；曾有旧 home `C:\Users\19121\.dsh`，迁移脚本 `migrate-dsh.ps1` 会把 DSH_HOME 指到 D:）。操作前必须 `$env:DSH_HOME` 与 `[Environment]::GetEnvironmentVariable('DSH_HOME','User')` 双确认。
- profile 目录 = `$DSH_HOME\profiles\<profile名>`（默认 `web`）。
- 插件安装位置 = `$DSH_HOME\profiles\node_modules\<包名>`（优先）或 `profile\node_modules\<包名>`。
- 挂载开关 = `$DSH_HOME\profiles\<profile名>\cordis.patch.yml`：YAML 顶层数组；`- insert:\n    - id: <id>\n      name: '<包名>'` 即挂载（穿），删除该块即卸载（脱）。**文件本身是用户层补丁，允许编辑**。
- 插件数据目录（`.taskman/`、`cindy-root.json` 等）与穿脱无关：脱衣服不丢数据。
- **DSH 主体安装目录（`@deepseek-ai/dsh` 所在）、apps/web、出厂预设：禁止任何修改**。
- git 代理：本机 git 全局配置了 `http://127.0.0.1:7897`。代理未开时克隆会报 `Failed to connect to 127.0.0.1 port 7897`；推送直连可能 `Connection was reset`。策略：**默认用配置的代理，失败再试直连**（`git -c http.proxy= -c https.proxy= ...`）。
- 控制台中文乱码是显示问题：文件本身常是合法 UTF-8，用 `node` 读取确认内容（如 patch 文件逐行 JSON.stringify 打印），不要据此判断文件损坏。

## 1. 插件登记（registry）

管理任何插件前，先读 `$DSH_HOME\plugins.json`；不存在则创建。结构：

```json
{
  "plugins": [
    {
      "id": "cindy",
      "packageName": "@mrchenxiangyu/cindy-taskman",
      "repo": "D:\\deepseek_workspace\\cindy-taskman",
      "repoUrl": "https://github.com/mrchenxiangyu/cindy-taskman.git",
      "installScript": "install-cindy.ps1",
      "patchId": "cindy",
      "sourceLib": "lib",
      "installedVersion": "0.2.1"
    }
  ]
}
```

- 登记/操作新插件：先询问用户补齐关键字段（仓库路径或 URL、包名、安装脚本名、patchId）。
- 操作完成后更新 `installedVersion`。

## 2. 上架发布（publish）

1. `git -C <repo> status --short` 确认改动；`git -C <repo> log --oneline -3` 看上下文。
2. **提升版本号**（`package.json` 的 `"version"`，注意保留原有格式与其余字节）：
   - 新增功能 → 次版本（0.2.0 → 0.3.0）；修复/界面调整 → 补丁版本（0.2.0 → 0.2.1）。
   - 用 node 脚本做精确字符串替换（PowerShell 转义易坏，别在 `-e` 里嵌引号）。
3. **必须更新 CHANGELOG.md**（仓库根，无则创建，首行写明约定）：追加 `[新版本] - 日期` 段，含：
   - 变更内容（新增/修复分列）
   - 「从上一版本升级」步骤（其他设备照做：pull → 重新运行安装脚本 → 重启；数据是否兼容要写明）
4. 提交：`git -C <repo> add -A && git -C <repo> commit -m "feat/fix/docs: ..."`（提交信息遵循仓库现有风格）。
5. 推送：`git -C <repo> push origin HEAD`（默认代理；失败换直连）。
6. 若本机已装该插件：把 `package.json`（与改动的 `lib/`）同步到 `$DSH_HOME\profiles\node_modules\<包名>`，告知用户重启。

## 3. 从 git 更新本地（update）

1. 仓库不存在 → `git clone <repoUrl> <repo>`；已存在 → `git -C <repo> pull`（代理失败换直连）。
2. 按 `CHANGELOG.md` 的升级步骤执行；通用要点：
   - 优先运行插件自带安装脚本（`install-*.ps1`，幂等）。
   - 手工复制时**必须**先删除目标包整个目录再复制（`Copy-Item -Recurse` 在目标 `lib` 已存在时会把新文件嵌成 `lib\lib\` 嵌套，顶层仍是旧文件——这是最常见的"装完没生效"根因）。
3. 确保挂载行在 `cordis.patch.yml` 中：
   - patch 文件若内容为「注释 + `[]`」，要**重写**为「注释 + insert 块」，不要追加到 `[]` 后面（会产生无效 YAML）。
4. **校验四连**：
   - 版本号：已装 `package.json` 与仓库一致
   - 哈希：`Get-FileHash` 对比已装 `lib\*` 与仓库 `lib\*`（MATCH）
   - 挂载行：`Select-String -Path <patch> -Pattern "<包名>"` 命中
   - YAML 可解析：用 `node` 以 DSH 同款解析器（js-yaml / yaml）`load` 该文件，确认结构为 `[{insert:[{id,name}]}]`
5. 更新 registry 的 `installedVersion`；告知用户 `Ctrl+C` 重启 DSH。
6. 若没生效：**先确认 DSH_HOME 是哪个**（可能装到了非活跃 home），再查 patch 与哈希。

## 4. 穿脱（toggle）

- **穿**：patch 文件加入（若在则跳过）
  ```yaml
  - insert:
      - id: <patchId>
        name: '<packageName>'
  ```
- **脱**：删除该块（连同其上方注释行）。
- 重启生效；数据不受影响；主体目录绝不动。

## 5. CHANGELOG 约定（强制）

- 每次发布必须更新 CHANGELOG.md：版本号 → 变更 → 升级步骤。
- 让其他设备用户能回答：「从哪个版本跳到哪个版本，需要怎么操作」。
- 数据兼容性必须明确说明（完全兼容 / 需要迁移 / 需要重新选择根目录等）。

## 6. 故障排查

| 现象 | 原因与处理 |
| --- | --- |
| 克隆 `Failed to connect 127.0.0.1:7897` | 代理未开 → 直连重试 |
| 推送 `Connection was reset` | 直连被墙 → 用配置的代理推送 |
| 重启后插件没出现 | ① DSH_HOME 不是你以为的目录（双确认环境变量）② 装到了旧 home ③ patch 行缺失/被注释/追加到 `[]` 后 ④ `lib\lib` 嵌套 |
| 版本号已变但功能没变 | `lib` 未覆盖 → 删包目录重装；或旧 home 残留 |
| 控制台中文乱码 | 显示问题，用 node 读内容判断，别按乱码改文件 |
| `dsh plugin` 命令报 pnpm 找不到 | 该 CLI 依赖 pnpm，未装则直接编辑 patch 文件完成穿脱 |

## 7. 安全边界

- 只动：`$DSH_HOME\profiles\node_modules\<包名>`、profile 的 `cordis.patch.yml`、`$DSH_HOME\plugins.json`、插件数据目录（用户指定）。
- 禁止：修改 DSH 主体安装目录、apps/web、出厂预设；删除插件时不得删除用户数据目录。
- 所有删除/覆盖前先 `Test-Path` 确认目标，并在回复中列明将写入/删除的路径。

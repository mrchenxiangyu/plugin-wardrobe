# 🧥 plugin-wardrobe（插件衣帽间）

> 通用的 DSH 用户插件更新管理技能 —— 插件像衣服，可穿可脱，不影响 DSH 主体。

管理按「**源码仓库 + 本地安装**」模式交付的 DSH 用户插件（如 `cindy-taskman`）的完整生命周期：

- **上架**：提升版本号 + 更新 CHANGELOG + 提交 + 推送 GitHub
- **更新**：从 git 拉取 → 安装到本机 DSH profile → 校验（版本/哈希/挂载行/YAML）→ 重启
- **穿脱**：增删 `cordis.patch.yml` 挂载行（穿=加行，脱=删行），数据不丢、主体不动
- **登记**：`$DSH_HOME/plugins.json` 维护多插件清单

**不绑定任何具体插件** —— 所有遵循下述规范的 DSH 用户插件都适用。

## 安装到一台设备

1. 获取本仓库：
   ```powershell
   git clone https://github.com/mrchenxiangyu/plugin-wardrobe.git
   ```
2. 把 `plugin-wardrobe` 目录放进目标设备的 **agent preset 的 skills 目录**：
   - 部署预设：`<DSH安装目录>\config\agent-presets\<预设名>\skills\plugin-wardrobe\`（注意：出厂预设升级会被覆盖，建议放进你自己的预设副本）
   - 或自定义预设：`$DSH_HOME\.agent-presets\<你的预设>\skills\plugin-wardrobe\`
3. 新开一个使用该预设的会话，即可用该技能（例：「用 plugin-wardrobe 更新 cindy」）。

> 技能即本仓库根目录（`SKILL.md`），安装 = 整个目录拷入 skills 目录。

## 快速使用

| 你想做什么 | 对话示例 |
| --- | --- |
| 发布新版本 | 「用 plugin-wardrobe 把 cindy 的改动发布到 git（升版本+写 CHANGELOG）」 |
| 更新本机插件 | 「用 plugin-wardrobe 按最新 git 更新本机 cindy」 |
| 穿/脱 | 「把 cindy 脱掉（卸载）」/「把 cindy 穿回来」 |
| 登记新插件 | 「登记一个新插件：仓库 xxx，包名 yyy」 |

## 插件「衣帽间兼容」规范

详细规范见 **`SPEC.md`**（目录布局、package.json 要求、安装脚本规范、版本号 semver 映射、CHANGELOG 约定、发布自检清单、更新四连校验）。

要让任意插件被本技能统一管理，插件应满足（速览）：

1. `package.json` 有语义化 `version`；源码含 `lib/`（host + client bundle）
2. 提供幂等安装脚本 `install-*.ps1`（复制 package.json + lib 到 profile node_modules，并处理 `cordis.patch.yml` 挂载行；重装必须先删目标包目录避免 `lib\lib` 嵌套）
3. 仓库根有 `CHANGELOG.md`，每次发布更新「版本 → 变更 → 升级步骤」
4. 数据与主体分离：插件数据放用户指定目录，绝不修改 DSH 主体安装目录

## 已登记插件示例

| 插件 | 仓库 | 包名 | 当前版本 |
| --- | --- | --- | --- |
| Cindy 任务秘书 | https://github.com/mrchenxiangyu/cindy-taskman | `@mrchenxiangyu/cindy-taskman` | 0.2.1 |

## 安全边界

- 只写 `$DSH_HOME\profiles\node_modules\<包名>`、profile 的 `cordis.patch.yml`、`$DSH_HOME\plugins.json`、插件数据目录
- **禁止**修改 DSH 主体安装目录、apps/web、出厂预设
- 卸载插件不删数据；所有删除/覆盖前先确认路径

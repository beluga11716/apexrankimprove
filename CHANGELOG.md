# 更新日志 (CHANGELOG)

## [v2.5.0] - 2026-07-06 — Steam 支持

### ✨ 新功能

- **🔗 Steam 链接/ID 自动识别**：`/apexrank`、`/apex绑定`、`/持续视奸` 等所有玩家相关命令直接支持：
  - Steam 个人资料链接（`https://steamcommunity.com/profiles/7656119...`）
  - 原始 17 位 SteamID64（`76561199382991896`）
  - `steam:` 前缀格式（`steam:76561199382991896`）
  输入后自动提取数字作为 UID 查询，**所有已有命令自动受益**。

- **🖼️ Steam 头像自动获取**：通过 SteamID 查询时，自动从 Steam 公开 XML API 获取玩家头像（184×184），显示在段位卡片的八边形区域中，替代默认灰色头像。
  - 使用 `steamcommunity.com/profiles/{id}/?xml=1` 公开端点，**无需额外 API Key**
  - 头像缓存到本地 `{data_dir}/avatars/` 目录
  - 获取失败静默降级到默认头像，不阻塞查询

- **🔖 绑定后自动获取 Steam 昵称**：`/apex绑定` 检测到 SteamID 时，自动调用 Steam API 获取显示名（personaname）作为默认查询昵称。
  - 绑定成功后，可直接用昵称替代 uid 查询：`/apexrank Beluga`
  - 昵称匹配基于当前用户（per-user），不与全局别名冲突
  - 命名优先级：全局别名 > 个人绑定昵称 > 原始输入

- **🏷️ `/apex昵称` 命令（新增）**：
  - `/apex昵称 老王` — 为当前用户绑定设置自定义查询昵称
  - `/apex昵称` — 查看当前昵称
  - `/apex昵称 清除` — 清除昵称（回复到无昵称状态）

- **📦 内置中文字体**：`NotoSansCJKsc-Regular.otf` 打包进 `assets/fonts/`，开箱即用，无需 `/apex_download`。解决 Docker 容器无系统字体、GitHub 被墙无法下载的问题。

- **🖼️ `/apex头像` 命令（新增）**：用户可在聊天中直接发送图片设置自定义头像，适合无法访问 Steam 的服务器环境。
  - `/apex头像` + 附带图片 → 设置自定义头像
  - `/apex头像` → 查看当前头像
  - `/apex头像 清除` → 恢复默认头像
  - 头像优先级：自定义上传 > Steam 自动获取 > 默认头像

### 🐛 Bug 修复

- **Steam 头像下载 NameError**：`tmp_path` 变量在 try 块内定义导致 except 块引用时可能未定义，已移至 try 之前
- **错误日志静默吞没**：`_fetch_and_apply_steam_profile` 和 `fetch_steam_profile` 的错误从 `debug` / `pass` 提升为 `logger.warning`

### 🔧 技术变更

- `_parse_identifier`：扩展标识符解析链，新增 Steam URL/ID 检测和 `steam:` 前缀
- `_normalize_user_bindings`：绑定存储从纯字符串扩展为 dict 格式（`{"target": "...", "nickname": "...", "steam_id": "...", "avatar_path": "..."}`），读取时兼容旧字符串格式
- `ApexPlayerStats` 新增 `steam_name` 和 `avatar_path` 字段
- `ApexApiClient` 新增 `fetch_steam_profile()` 方法（Steam XML API 调用）
- `_resolve_player_alias_info` 新增 `event` 可选参数，支持昵称解析
- 新增 `_get_binding_avatar()`、`_apply_custom_binding_avatar()`、`_resolve_bundled_cjk_font_path()` 辅助方法
- 字体加载优先级：本地打包 > 系统字体 > 下载缓存 > DejaVu Sans > PIL default
- `metadata.yaml`：repo 改为 `https://github.com/beluga11716/apexrankimprove`，版本 `2.5.0`
- 帮助文本更新：新增 Steam URL/ID、`/apex昵称`、`/apex头像` 命令说明

---

## [v2.5.1] - 2026-07-07 — 头像 per-binding 修复

### 🐛 Bug 修复

- **自定义头像错误应用到所有查询**：v2.5.0 中 `/apex头像` 设置的自定义头像会被应用到该用户查询的**所有**玩家，而不是仅绑定的那个。例如用户绑定了账号 A 并设了头像，然后查询玩家 B，B 也会显示 A 的头像。

### ✨ 改进

- **`/apex头像` 支持昵称/UID 参数**：
  - `/apex头像 老王` + 图片 → 为绑定的「老王」设置头像
  - `/apex头像 uid:76561199382991896` + 图片 → 通过 UID 指定绑定设置头像
  - `/apex头像 老王 清除` → 清除头像
  - 参数与绑定不匹配时会提示错误
- **`_apply_custom_binding_avatar` 增加匹配检查**：仅当查询到的玩家 UID/名字 + 平台匹配绑定目标时，才应用自定义头像
- **头像缓存路径改为 per-binding**：从 `custom_{user_id}.png` 改为基于绑定目标 UID 的路径，让头像真正对应到具体账号

### 🔧 技术变更

- 新增 `_player_matches_binding_target()` 静态方法：按平台 + UID（优先）或名字验证玩家是否匹配绑定目标
- `_custom_avatar_cache_path()` 参数从 `user_id` 改为 `binding_identifier`
- `metadata.yaml` 帮助文本更新：头像命令参数格式

---

## [v2.4.1] - 原始上游版本

原始插件版本，支持：
- Apex 段位/分数/在线状态查询（按名字或 `uid:`/`uuid:` 前缀）
- 群聊持续监控（通知模式与仅记录模式）
- 分数变化高清长图
- 地图轮换、赛季信息、猎杀阈值查询
- 全局别名系统、个人绑定系统

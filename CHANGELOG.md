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

### 🔧 技术变更

- `_parse_identifier`：扩展标识符解析链，新增 Steam URL/ID 检测和 `steam:` 前缀
- `_normalize_user_bindings`：绑定存储从纯字符串扩展为 dict 格式（`{"target": "...", "nickname": "...", "steam_id": "..."}`），读取时兼容旧字符串格式
- `ApexPlayerStats` 新增 `steam_name` 和 `avatar_path` 字段
- `ApexApiClient` 新增 `fetch_steam_profile()` 方法（Steam XML API 调用）
- `_resolve_player_alias_info` 新增 `event` 可选参数，支持昵称解析
- 帮助文本更新：新增 Steam URL/ID 使用说明和 `/apex昵称` 命令

### 📝 涉及文件

| 文件 | 说明 |
|------|------|
| `main.py` | +`import re`、正则模式、Steam ID 提取、标识符解析扩展、头像缓存与渲染、绑定 dict 格式、昵称解析、`/apex昵称` 命令、帮助文本 |
| `apex_service.py` | +`import xml.etree.ElementTree`、`ApexPlayerStats` 新字段、`fetch_steam_profile` 方法 |

---

## [v2.4.1] - 原始上游版本

原始插件版本，支持：
- Apex 段位/分数/在线状态查询（按名字或 `uid:`/`uuid:` 前缀）
- 群聊持续监控（通知模式与仅记录模式）
- 分数变化高清长图
- 地图轮换、赛季信息、猎杀阈值查询
- 全局别名系统、个人绑定系统

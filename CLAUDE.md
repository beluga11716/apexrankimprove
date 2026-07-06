# CLAUDE.md — astrbot_plugin_apexrankwatch

## 项目概述

AstrBot 插件，基于 https://github.com/moeneri/astrbot_plugin_apexrankwatch ，查询 Apex Legends 玩家段位/分数/在线状态，支持持续监控、分数变化图表、地图轮换、赛季信息、猎杀阈值。

本项目在原始项目基础上增加了 Steam 支持：
- Steam 个人资料链接/SteamID64 自动识别为 UID
- Steam 头像自动获取（通过公开 XML API，无需额外 Key）
- 绑定后自动获取 Steam 显示名作为查询昵称
- `/apex昵称` 命令自定义昵称

## 目录结构

| 路径 | 说明 |
|------|------|
| `main.py` | 插件主逻辑（命令注册、标识符解析、图片渲染、监控轮询） |
| `apex_service.py` | API 服务层（Mozambiquehe.re API 客户端、Steam 资料获取） |
| `storage.py` | 数据持久化（GroupStore、PlayerRecord、ScoreChangeRecord） |
| `utils.py` | 工具函数（上海时区、类型转换） |
| `translations.json` | 英文→中文翻译映射 |
| `_conf_schema.json` | AstrBot 面板配置 Schema |
| `metadata.yaml` | 插件元数据 |
| `assets/` | 静态资源（段位图标、传奇图标、地图图片、默认头像） |
| `tests/` | 测试文件 |

## 核心架构

### 标识符解析链

```
用户输入 → _parse_player_platform → _resolve_player_alias_info → _parse_identifier → API 调用
```

`_parse_identifier` 支持的格式（按优先级）：
1. Steam 个人资料链接 → 提取 SteamID64 → `use_uid=True`
2. 原始 SteamID64（17 位，7656119 开头） → `use_uid=True`
3. `steam:` 前缀 → SteamID64 → `use_uid=True`
4. `uid:` / `uuid:` 前缀 → `use_uid=True`
5. 其他 → 视为玩家名 → `use_uid=False`

### API 数据流

```
Mozambiquehe.re API (api.mozambiquehe.re/bridge)
├── ?player=NAME&platform=PC  → 按名字查询
├── ?uid=STEAMID64&platform=PC → 按 UID 查询（SteamID64 天然被接受）
├── /maprotation → 地图轮换
├── /predator → 猎杀阈值

Steam Community XML (steamcommunity.com/profiles/{id}/?xml=1)
├── 公开端点，无需 API Key
├── 获取 personaname（显示名）、avatarFull（头像 URL）
└── Valve 已标记 deprecated，但目前仍可用
```

### 绑定存储格式

```python
# 旧格式（向后兼容）
_runtime_user_bindings = {"user_id": "uid:7656... PC"}

# 新格式（支持昵称和 Steam 元数据）
_runtime_user_bindings = {
    "user_id": {
        "target": "uid:7656... PC",
        "nickname": "Beluga",
        "steam_id": "76561199382991896",
        "steam_name": "Beluga"
    }
}
```

读取时使用 `_get_binding_target()` / `_get_binding_nickname()` 兼容两种格式。

### 昵称解析优先级

1. 全局别名（`/apexalias` 设置，管理员管理）
2. 个人绑定昵称（`/apex昵称` 或 Steam 自动获取）
3. 无匹配 → 按原始输入查询 API

## 关键约束

- 所有改动必须保证原项目全部功能可用
- 新增功能只是扩展，不破坏已有行为
- Steam 资料/头像获取失败时必须静默降级，不能阻塞命令
- 头像缓存到本地 `{data_dir}/avatars/` 目录
- 任何不确定的细节必须先与用户确认

---

## 本次修改日志（2026-07-06）

### 背景

用户使用插件时遇到三个问题：
1. 不知道自己的 UID — EA 不再暴露数字 UID，`beluga11716` 是 EA Account ID 而非游戏名
2. 返回的头像永远是默认灰色图
3. 每次查询都要输入长串 uid，没有便捷方式

经过 API 实测确认：Mozambiquehe.re 的 `uid` 参数接受 SteamID64（17 位，以 7656119 开头）。SteamID64 可从 Steam 客户端直接获取，无需任何第三方网站。

### 对话流程

1. **用户描述问题**：uid 查询功能不会用，找不到自己的 uid
2. **探索代码库**：分析 `_parse_identifier`、API 调用链、命令注册、渲染流程
3. **搜索确认**：验证 Mozambiquehe.re API 状态（确认存活）、Steam XML 端点可用性
4. **定位根因**：`beluga11716` 是 EA Account ID ≠ Origin Persona Name，API 查不到
5. **找到方案**：用户在 apexlegendsstatus.com 用 Steam 登录后得到 `/uid/PC/76561199382991896`，这串数字就是 SteamID64
6. **确认需求 → 进入计划模式 → 批准后实施**

### 设计决策

| 决策 | 结论 |
|------|------|
| Steam 资料获取方式 | Steam Community XML (`?xml=1`)，公开免费，无需额外 API Key |
| Steam API Key 需求 | 不强制，使用 XML 公开端点；未来可切换到 Web API |
| 头像下载方式 | 同步 `httpx.Client` 在 `asyncio.to_thread` 中运行，3 秒超时 |
| 头像缓存 | 本地 `{data_dir}/avatars/{steam_id}.png`，不清除 |
| 头像失败降级 | 静默回退到默认头像，不阻塞命令 |
| 绑定存储 | 从字符串扩展为 dict 格式，读取时兼容两种格式 |
| 昵称范围 | 个人（per-user），不是全局别名 |
| 昵称解析位置 | 在 `_resolve_player_alias_info` 中，全局别名优先，未命中时查昵称 |
| `_resolve_player_alias_info` 签名 | 新增可选 `event` 参数，缺省为 None（不影响已有调用） |
| Steam URL 格式 | 支持完整链接、短链接、原始 17 位数字、`steam:` 前缀 |

### 代码改动详情

#### main.py（约 250 行变更）

- **导入新增**：`import re`
- **正则模式**：`_STEAM_PROFILE_URL_RE` — 匹配 `steamcommunity.com/profiles/{17位数字}`；`_STEAM_RAW_ID_RE` — 匹配 `7656119` 开头的 17 位纯数字
- **`_extract_steam_id(text)`**：静态方法，从 URL 或原始 SteamID64 提取数字，返回 `str | None`
- **`_parse_identifier`**：在 `uid:`/`uuid:` 之前新增 Steam 检测；新增 `steam:` 前缀支持
- **`_avatars_dir()` / `_steam_avatar_cache_path()`**：头像缓存路径工具
- **`_fetch_and_apply_steam_profile(player_data)`**：异步获取 Steam 资料并下载头像，设置 `steam_name` / `avatar_path`
- **`_download_steam_avatar(steam_id, avatar_url)`**：同步下载头像 PNG 到本地缓存
- **`_draw_player_profile_panel`**：优先使用 `player_data.avatar_path` 渲染头像
- **`_get_binding_target/nickname/steam_id()`**：绑定数据读取辅助，兼容字符串和 dict 格式
- **`_normalize_user_bindings`**：保留 dict 格式（nickname、steam_id、steam_name 字段）
- **`_resolve_player_alias_info`**：新增 `event` 参数；全局别名未命中时调用 `_resolve_binding_nickname`
- **`_resolve_binding_nickname(event, alias_key)`**：检查当前用户的绑定昵称是否匹配
- **`apexbind` 命令**：检测 Steam URL → 自动获取 Steam 显示名作为昵称；存储 dict 格式
- **`/apex昵称` 命令（新增）**：设置/查看/清除自定义昵称
- **帮助文本更新**：Steam URL/ID 示例、`/apex昵称` 说明

#### apex_service.py（约 25 行变更）

- **导入新增**：`from xml.etree import ElementTree`
- **`ApexPlayerStats` 新增字段**：`steam_name: str = ""`、`avatar_path: str = ""`
- **`fetch_steam_profile(steam_id)`**：异步方法，请求 `steamcommunity.com/profiles/{id}/?xml=1`，解析 XML 获取 `steamID`（显示名）和 `avatarFull`（头像 URL），失败返回空 dict

#### tests/test_output_mode.py（2 行变更）

- `test_apexbind_stores_user_binding_with_uid_slash_prefix`：适配 dict 格式断言

### 涉及文件

| 文件 | 改动 |
|------|------|
| main.py | +`import re`、正则模式、Steam ID 提取、标识符解析扩展、头像缓存、绑定 dict 格式、昵称解析、`/apex昵称` 命令、帮助文本 |
| apex_service.py | +`import xml.etree.ElementTree`、`ApexPlayerStats` 新字段、`fetch_steam_profile` 方法 |
| tests/test_output_mode.py | 适配 dict 格式断言 |

### 保留的全部功能

- ✅ `uid:` / `uuid:` 前缀查询 — 不变
- ✅ 玩家名查询 — 不变
- ✅ 全局别名系统 (`/apexalias`) — 不变
- ✅ 用户绑定系统 (`/apex绑定`) — 扩展而非重写
- ✅ 监控系统 (`/持续视奸`) — 自动受益于 Steam ID 识别
- ✅ 地图/赛季/猎杀查询 — 零影响
- ✅ `steam:` 前缀 — 新增
- ✅ 帮助文本 — 更新

### 测试结果

```
87 passed, 5 failed
5 failed = 全部 Pillow 环境缺失（已有问题，与本次改动无关）
0 regressions
```

### 部署注意事项

- **Steam XML 端点** (`steamcommunity.com/profiles/{id}/?xml=1`) 是公开端点，但 Valve 已 deprecated。如果未来失效，可切换到 Steam Web API（需额外 Key）
- **头像下载** 使用 `asyncio.to_thread`（Python 3.9+），在 Docker 中应正常工作
- **网络要求**：插件需要访问 `api.mozambiquehe.re`（Apex API）和 `steamcommunity.com`（Steam 资料）。如果 Docker 容器需要代理访问外网，需在 httpx client 中配置
- **向后兼容**：旧版字符串格式的绑定数据自动兼容，无需迁移

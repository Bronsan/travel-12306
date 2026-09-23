# travel-12306

> 查询中国铁路 12306 余票，制定出行 / 订票计划的 **Agent 技能包（Skill）**。  
> **MIT 开源** · 仅查询与规划 · 不登录、不占座、不代购、不抢票。

本仓库提供可安装到 MiMo Desktop / MiMoCode（及兼容 SKILL.md 约定的 Agent）的技能指令与元数据，配合 [12306-mcp](https://github.com/Joooook/12306-mcp) 等 MCP 查询服务，让模型能规范地：

- 查城市 / 车站代码  
- 查直达余票、中转余票  
- 查车次经停与时刻  
- 按偏好整理 2–3 个可选方案并给出推荐  

**使用前请阅读 [DISCLAIMER.md](./DISCLAIMER.md) 免责声明与使用声明。**

---

## 目录结构

```text
12306-mit/
├── LICENSE                 # MIT 许可证
├── README.md               # 本文件：安装与使用
├── DISCLAIMER.md           # 免责声明、合规边界、第三方归属
├── skill/                  # 可安装的技能包
│   ├── SKILL.md            # 技能正文（触发条件、工具、流程）
│   └── locales/
│       ├── zh-CN.json      # 插件页显示名（中文）
│       └── en-US.json      # 插件页显示名（英文）
└── examples/
    └── mcp-config.example.jsonc
```

---

## 功能一览

| 能力 | 说明 | 状态 |
| --- | --- | --- |
| 城市 / 车站 → telecode | `get-station-code-of-citys` 等 | ✅ |
| 直达余票 | `get-tickets`，可过滤车次类型与时段 | ✅ |
| 中转余票 | `get-interline-tickets` | ✅ |
| 经停站 / 时刻 | `get-train-route-stations` | ✅ |
| 方案对比与推荐 | 技能流程约定（最快 / 有票 / 时段） | ✅ |
| 登录 / 下单 / 抢票 | **不做** | ❌ |

数据来自 12306 公开查询接口（经 MCP 服务封装）。余票实时变动，下单前以 12306 官方为准。

---

## 系统要求

| 依赖 | 版本建议 | 用途 |
| --- | --- | --- |
| Node.js | ≥ 18 | 运行 `npx 12306-mcp` |
| 网络 | 可访问 npm 与 12306 查询接口 | 安装与查询 |
| Agent 运行时 | MiMo Desktop / MiMoCode，或支持 `SKILL.md` 的同类环境 | 加载本技能 |
| MCP 宿主 | 支持 stdio MCP 的客户端 | 调用查票工具 |

---

## 安装

### 1. 安装技能包（Skill）

**MiMo Desktop / MiMoCode（推荐）**

把 `skill/` 目录复制到技能根目录，目录名保持 `travel-12306`：

```text
# 全局（所有项目可用）
~/.config/mimocode/skills/travel-12306/

# 或当前项目
<project>/.mimocode/skills/travel-12306/
```

目标目录结构应为：

```text
travel-12306/
├── SKILL.md
└── locales/
    ├── zh-CN.json
    └── en-US.json
```

新开对话后，引擎会扫描发现该技能；插件页会显示「12306 出行查票」。

**其他支持 SKILL.md 的 Agent**

将 `skill/` 下内容放到该 Agent 约定的技能路径（如部分工具的 `~/.agents/skills/travel-12306/`），保证 frontmatter 的 `name: travel-12306` 与目录名一致即可。

### 2. 配置 MCP 查询服务

本技能默认对接 npm 包 [`12306-mcp`](https://github.com/Joooook/12306-mcp)（MIT）。

**MiMo Desktop：** Settings → MCP，或编辑 `~/.config/mimocode/mimocode.jsonc` 的顶层 `mcp` 字段，参考 [examples/mcp-config.example.jsonc](./examples/mcp-config.example.jsonc)：

```jsonc
{
  "mcp": {
    "12306-mcp": {
      "type": "local",
      "command": ["npx", "-y", "12306-mcp"],
      "enabled": true
    }
  }
}
```

**Claude Desktop / 其他 mcpServers 风格：**

```json
{
  "mcpServers": {
    "12306-mcp": {
      "command": "npx",
      "args": ["-y", "12306-mcp"]
    }
  }
}
```

改完配置后**重启客户端或新开会话**。也可用 HTTP 模式：

```bash
npx -y 12306-mcp --port 8080
```

### 3. 验证

新开会话后试一句：

```text
明天上海去北京，帮我看看高铁票
```

若模型正确调用了车站查询与 `get-tickets` 并给出方案表，即接入成功。

---

## 使用方法

### 直接对 Agent 说人话

技能按自然语言触发，无需记住指令。常见说法：

| 你说 | Agent 会做 |
| --- | --- |
| 明天上海去北京，查一下余票 | 定日期 → 定站码 → 查直达 → 出方案 |
| 下周三杭州到南京，想坐高铁，上午出发 | 加车次/时段过滤后推荐 |
| 上海到兰州直达不合适，能不能中转？ | 对比直达后查 `get-interline-tickets` |
| G1033 经停哪些站？ | `get-train-route-stations` |
| 周末出去玩，从广州到厦门 | 补齐日期后查票并排行程 |

缺出发地 / 目的地 / 日期时，Agent 会先追问，而不是瞎猜。

### 标准工作流（技能内部约定）

1. 解析：出发地、目的地、日期、车次类型、时段、是否中转  
2. 相对日期 → `get-current-date` → `yyyy-MM-dd`（Asia/Shanghai）  
3. 城市/站名 → telecode  
4. `get-tickets` 直达  
5. 不理想则 `get-interline-tickets`  
6. 整理 2–3 个方案（最快 / 有票 / 时段）+ 一句推荐  
7. 引导到 **12306 官方渠道** 购票  

### 输出示例

```text
推荐（按你的偏好排序）

1. G8  上海虹桥 08:00 → 北京南 12:26（4h26）
   二等座 有票 661元 | 一等座 无票
   理由：上午出发里最快

2. G10  上海虹桥 10:00 → 北京南 14:28（4h28）
   二等座 有票 661元 | 一等座 有票 1060元
   理由：一等座也有票，时间更宽松

购票请到 12306 官方 App / 官网完成；余票实时变动，下单前以官方为准。
```

### MCP 工具参考

| 工具 | 参数要点 | 用途 |
| --- | --- | --- |
| `get-current-date` | — | 上海时区当前日期 |
| `get-station-code-of-citys` | `citys`，可用 `\|` 批量 | 城市代表站 code |
| `get-stations-code-in-city` | `city` | 城市内全部车站 |
| `get-station-code-by-names` | 站名，如 `北京南` | 站名 → code |
| `get-station-by-telecode` | telecode | 反查车站 |
| `get-tickets` | `date`, `from`, `to`, 过滤项 | 直达余票 |
| `get-interline-tickets` | 出发/到达/日期，可选 `middleStation` | 中转方案 |
| `get-train-route-stations` | 车次与日期 | 经停与时刻 |

`trainFilterFlags`：`G` 高铁/城际 · `D` 动车 · `Z` 直达 · `T` 特快 · `K` 快速 · `O` 其他 · `F` 复兴号 · `S` 智能动车组。

常用过滤：`earliestStartTime` / `latestStartTime`（小时）、`showWZ`（是否显示无座）。

### 边界（请遵守）

- ✅ 查余票、排行程、看经停、比中转  
- ❌ 登录账号、占座、下单、支付、改签退票  
- ❌ 抢票 / 刷票 / 倒卖车票  
- ❌ 高频批量请求、绕过限流与风控  

用户要求代购或抢票时，应拒绝并引导官方渠道。详见 [DISCLAIMER.md](./DISCLAIMER.md)。

---

## 故障排查

| 现象 | 处理 |
| --- | --- |
| 会话里看不到 12306 工具 | 确认 MCP 已配置且 `enabled: true`，然后**新开对话** |
| `npx 12306-mcp` 失败 | 检查 Node ≥ 18、网络与 npm 源；或 `npm i -g 12306-mcp` 后改用绝对路径 |
| 找不到城市/站名 | 去掉「站」字；或先查该城全部车站 |
| 余票为空 | 放宽时段、允许中转、换相邻日期 |
| 相对日期算错 | 必须先 `get-current-date`，按 Asia/Shanghai 计算 |
| 接口报错 / 限流 | 降低频率稍后重试；以官方 App 复核 |

---

## 声明（摘要）

完整条款见 **[DISCLAIMER.md](./DISCLAIMER.md)**，要点如下：

1. **非官方**：与中国铁路、12306 无隶属或授权关系。  
2. **仅查询规划**：不登录、不购票、不抢票。  
3. **数据仅供参考**：余票票价实时变动，以 12306 官方为准。  
4. **合规使用**：遵守法律与 12306 服务条款，禁止倒票与滥用接口。  
5. **现状提供**：在法律允许范围内不承担因使用产生的损失责任。

---

## 第三方与致谢

| 项目 | 用途 | 许可 |
| --- | --- | --- |
| [Joooook/12306-mcp](https://github.com/Joooook/12306-mcp) | 默认 MCP 查票服务（npm: `12306-mcp`） | MIT，Copyright (c) 2025 Jok |
| [modelcontextprotocol](https://github.com/modelcontextprotocol/modelcontextprotocol) | MCP 协议与生态 | 见上游仓库 |
| 12306 公开查询接口 | 车次 / 余票数据来源 | 权利归中国铁路 / 12306 相关权利人 |

本仓库**不包含**第三方源码副本，仅提供技能指令与接入文档。使用第三方组件时请遵守其许可证。

---

## 许可证

本项目自有内容采用 [MIT License](./LICENSE)。

```text
MIT License
Copyright (c) 2026 Florance / travel-12306 contributors
```

贡献即表示你同意以 MIT 许可你的改动，并已阅读 [DISCLAIMER.md](./DISCLAIMER.md)。

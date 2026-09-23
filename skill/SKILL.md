---
name: travel-12306
description: 查询中国铁路12306余票并制定出行/订票计划。Use when 用户提到出去玩、出行、旅游、订火车票/高铁票、查余票、排行程、回家、出差、中转、经停站，或给出出发地/目的地/日期组合时使用。不用于机票、酒店、汽车票或登录下单抢票。
---

# 12306 出行查票与行程计划

## Important

- 只做**查询与规划**，不登录、不占座、不代购、不抢票。
- 优先调用已接入的 MCP 工具（`12306-mcp` 或兼容实现）。若本会话没有这些工具，再走本地 stdio 回退。
- 相对日期（明天/下周三/周末）必须先落到 `yyyy-MM-dd`，再查票。
- 输出给用户时用中文表格/列表，突出：车次、时刻、历时、席别余票与票价、推荐理由。
- 提醒用户：余票实时变动，下单前以 12306 官方为准。

## 触发场景（应主动使用）

- 「明天上海去北京」「周末出去玩」「帮我订高铁」「查一下余票」
- 「回家的票还有吗」「出差排行程」「有没有中转方案」
- 「G1033 经停哪些站」

## 可用工具

| 工具 | 用途 |
|------|------|
| `get-current-date` | 上海时区当前日期 |
| `get-station-code-of-citys` | 城市代表站 code（可用 `\|` 批量） |
| `get-stations-code-in-city` | 城市内全部车站 |
| `get-station-code-by-names` | 具体站名 → code（如「北京南」） |
| `get-station-by-telecode` | telecode 反查车站 |
| `get-tickets` | 直达余票 |
| `get-interline-tickets` | 中转余票（约前 10 条） |
| `get-train-route-stations` | 车次经停与时刻 |

`trainFilterFlags`：`G` 高铁/城际，`D` 动车，`Z` 直达，`T` 特快，`K` 快速，`O` 其他，`F` 复兴号，`S` 智能动车组。

## 标准流程

1. **解析需求**：出发地、目的地、日期/相对日期、人数偏好、车次类型、时段、是否接受中转。
2. **定日期**：相对日期先调 `get-current-date`，再计算目标日（按 Asia/Shanghai）；用户给了绝对日期则直接用 `yyyy-MM-dd`。
3. **定车站**：
   - 只说城市 → `get-station-code-of-citys`；
   - 指定车站 → `get-station-code-by-names`；
   - 不确定该城有哪些站 → `get-stations-code-in-city`。
4. **查直达**：`get-tickets`，按需设 `trainFilterFlags`、`earliestStartTime`、`latestStartTime`、`showWZ`。
5. **直达不理想**：再查 `get-interline-tickets`（可指定 `middleStation`）。
6. **用户点名某车次**：用 `get-train-route-stations` 补经停信息。
7. **整理方案**：至少给 2–3 个可选项（最快 / 最便宜或二等座有票 / 时段合适），标明余票与价格，给出一句明确推荐。
8. **追问缺口**：缺出发地/目的地/日期时先问清，不要瞎猜；其余偏好可默认高铁、二等座优先。
9. **引导下单**：明确告知用户到 12306 官方渠道完成购票，不要代替用户操作账号。

## MCP 未挂载时的回退

若当前会话没有 12306 的 MCP 工具，可在本机 stdio 启动已安装的 MCP 服务后调用：

```bash
npx -y 12306-mcp
```

或指向本地安装入口（路径因环境而异）：

```bash
node <path-to>/12306-mcp/build/index.js
```

stdio JSON-RPC：`initialize` → `notifications/initialized` → `tools/call`。

若临时目录被清理，可重新安装：

```bash
npm install 12306-mcp --prefix <some-runtime-dir>
```

## 示例

**用户**：下周三从杭州去南京，想坐高铁，上午出发，帮我看看票。

1. `get-current-date` → 算出下周三 `yyyy-MM-dd`
2. `get-station-code-of-citys`：`杭州|南京`
3. `get-tickets`：date=目标日，from/to=站名或 code，`trainFilterFlags=G`，`earliestStartTime=6`，`latestStartTime=12`
4. 输出推荐车次表 + 一句建议 + 官方购票提醒

**用户**：上海到兰州直达没合适的，能不能中转？

1. 先查直达作对比
2. `get-interline-tickets`（可不传 middle）
3. 比较总历时、换乘站、两段余票后给方案

## 输出格式建议

```text
推荐（按你的偏好排序）
1. G8  上海虹桥 08:00 → 北京南 12:26（4h26）
   二等座 有票 661元 | 一等座 无票
   理由：上午出发里最快

2. ...

购票请到 12306 官方 App / 官网完成；余票实时变动，下单前以官方为准。
```

## 边界与合规

- **不要**登录 12306、填写账号密码、占座、下单、支付、抢票、刷票。
- **不要**批量、高频调用查询接口。
- 用户要求代购或抢票时，礼貌拒绝，并引导至官方渠道。
- 完整声明见仓库根目录 `DISCLAIMER.md`。

## Troubleshooting

| 问题 | 处理 |
|------|------|
| 找不到城市/站名 | 去掉「站」字；或用 `get-stations-code-in-city` 列出候选 |
| 余票为空 | 放宽时段/允许中转/换相邻日 |
| 工具未出现在会话 | 配置已有则新开对话；或走 stdio 回退 |
| 相对日期算错 | 必须先 `get-current-date`，按 Asia/Shanghai 计算 |
| 接口限流/失败 | 稍后重试；降低查询频率；检查网络 |

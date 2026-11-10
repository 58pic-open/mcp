---
name: 58pic
description: >-
  千图（58pic）AI 开放平台 MCP：搜素材、查模型、生成图片（文生图 / 图生图 / 做同款）、下载素材，
  智能设计行业 Agent（电商套图 / 详情页 / 海报 / 品牌 / 社媒 / PPT / 视频）与 MiniApp 风格玩法。
  在用户提到千图、58pic、生图、文生图、做同款、找素材、找模板、按风格生成、PID、下载素材、
  智能设计、电商设计、商品套图、主图、详情页、海报设计、品牌设计、风格玩法、MiniApp、一键同款玩法时使用。
  通过 MCP 工具调用（list_models / search_images / generate_image 等）完成，不要尝试执行 shell 命令。
  首次使用会自动弹浏览器走 OAuth 授权，无需用户复制 API Key。
---

# 千图 AI 开放平台（MCP 版）使用指南

本 Skill 配套 **58pic MCP Server**（远程 HTTP + OAuth 2.1）。所有操作通过 MCP tool 调用完成。

> ⚠️ **不要** 在终端执行 `58pic` 命令 —— 那是另一套独立的 CLI 产品。本 Skill 仅指导 MCP tool 的使用。

## 何时使用

用户表达以下意图时主动调用本套 tool：

- "用 AI 生成一张 / 一套图片"、"画一张 ..."、"做一张海报 / banner / 插画"
- "搜一下千图有没有 ... 的素材 / 模板"
- "按这张图的风格再生成一张"、"做同款"
- "下载这张素材"、"给我这个图的源文件"
- "查我千图的剩余积分"
- "帮我做一套电商主图 / 详情页 / 品牌 VI / 小红书图文 / PPT"（→ 智能设计行业 Agent）
- "用千图的 XX 玩法 / 风格玩法做一个"、"商品套图、批量换背景、产品分解特效"（→ MiniApp）

## 鉴权状态判断

第一次调用任意 tool 时，如果客户端尚未授权，MCP 客户端会自动弹浏览器走 OAuth 流程。**不要**：

- 不要尝试代用户登录
- 不要询问用户的 API Key、密码
- 不要建议用户去命令行配置任何东西

应当：

- 如果 tool 返回 `error.type=AUTH_ERROR`，告诉用户："看起来千图 MCP 还没授权，请在 [设置 → Connectors] 里完成授权后重试。"
- 其余报错按下文 **错误处理** 章节执行

## 可用工具速查

| Tool | 何时用 | 扣点 |
|------|--------|------|
| `list_models` | **调 generate_image 之前必须先调一次** | 否 |
| `list_catalog` | 用户想按分类找素材时 | 否 |
| `search_images` | 找素材、找参考图、找 pid | 否 |
| `get_credits` | 大批量生成 / 下载前确认余额 | 否 |
| `generate_image` | AI 生成图片（含文生图、图生图、做同款） | **是** |
| `get_generation_status` | 仅在 `generate_image` 因 TIMEOUT 返回 task_id 时用 | 否 |
| `get_download_info` | 用户要拿素材的原文件下载链 | **是** |
| `list_design_agents` | 智能设计前选行业 Agent / 子模式 / 参数 | 否 |
| `generate_smart_design` | 智能设计 Agent 出图（套图、详情页、海报、品牌、PPT…） | **是** |
| `get_smart_design_result` | `generate_smart_design` 返回 `running` 时轮询 | 否 |
| `continue_smart_design` | 回答 Agent 追问（`awaiting_user`）/ 改稿 | **是** |
| `list_miniapps` | 搜风格玩法 / AI 应用 | 否 |
| `get_miniapp` | 运行前看玩法需要的输入（参考图 / 提示词槽位） | 否 |
| `run_miniapp` | 运行玩法出同款效果 | **是** |

每个 tool 详细的入参 / 返回字段从 MCP 协议层就能拿到，不在这里重复。

## 工作流配方

### 配方 1：纯文生图（最常见）

```
用户："帮我画一张赛博朋克风格的猫，1024x1024"

步骤：
1. 调 list_models()，从返回里挑一个 supports 包含 "text2image" 的模型
   - 倾向选 description / category 与"赛博朋克"风格匹配的
   - 没有完美匹配时选通用模型
2. 调 generate_image({
     model_id: <上一步选的>,
     prompt: "赛博朋克风格的猫，霓虹灯光，雨夜，细节丰富，写实",
     num_images: 1,
     extra: { width: 1024, height: 1024 }
   })
3. 拿到返回的 images[0].url，呈现给用户（见"结果呈现"章节）
4. 在回复里附上扣点信息（meta.credits_used / meta.credits_remaining）
```

### 配方 2：搜素材 + 做同款

```
用户："找一张 618 大促的海报，按它的风格再做一张母婴主题的"

步骤：
1. 调 search_images({ keyword: "618 大促 海报" })
2. 把前 3~5 张展示给用户，问"用哪张作为参考？"
   ⚠️ 不要自作主张选第一张就开生成，让用户挑
3. 用户选定后，调 list_models() 找支持 same_style 的模型
4. 调 generate_image({
     model_id: ...,
     reference_pid: "<用户选的 pid>",
     prompt: "改为母婴行业主题，温馨色调，宝宝元素",
     num_images: 1
   })
5. 呈现结果
```

### 配方 3：图生图（用户给了外部参考图）

```
用户："参考 https://example.com/cat.jpg 这张图，画一张油画风的猫"

步骤：
1. 调 list_models() 找支持 image2image 的模型
2. 调 generate_image({
     model_id: ...,
     reference_image_url: "https://example.com/cat.jpg",
     prompt: "油画风，厚涂笔触，温暖光线"
   })
3. 呈现
```

### 配方 4：批量生成（多张同主题）

```
用户："给我做 4 张不同风格的咖啡店海报"

步骤：
1. 调 list_models()，挑通用海报模型
2. ⚠️ 4 张图扣点不少 → 先调 get_credits() 确认余额
3. 如果余额够：
   - 推荐方案 A：一次 generate_image({ num_images: 4, prompt: "..." })
   - 推荐方案 B：4 次独立调用，prompt 各不同（如"日系小清新"/"复古工业"/"极简北欧"/"暖色温馨"）
   - 让用户选 A 还是 B
4. 余额不够：明确告诉用户"当前余额 X，本次需要 Y，差 Z，无法继续"
```

### 配方 5：素材下载

```
用户："把刚才搜到的第三张原图给我"

步骤：
1. ⚠️ 下载会扣点 → 先确认用户清楚扣点
   "下载这张原文件会扣 X 积分，确认下载吗？"
2. 用户确认后，调 get_download_info({ pid: "<对应 pid>" })
3. 把 download_url 给用户，提示链接有时效（expires_at）
4. 报告扣点
```

### 配方 6：智能设计行业 Agent（电商 / 海报 / 品牌 / 社媒 / 办公 / 视频）

```
用户："帮我给这款保温杯做一套亚马逊主图，卖点是 24 小时保温"

步骤：
1. 本地图片先 create_upload 拿 URL
2. list_design_agents({ keyword: "电商" }) → 选 agent_key（如电商设计 Agent）与 second_modes[].name
3. 按该子模式 params 收集参数：select / multi_select 只能取 options；required 缺失就问用户
4. generate_smart_design({ agent_key, second_mode, params: { 平台: "亚马逊", 卖点: "24小时保温" },
     prompt: "保温杯主图套图", reference_image_urls: [<商品图>] })
5. 按 status 处理：
   - succeeded → 展示 images
   - awaiting_user → 把 ask_user.question / options / steps 原样转述给用户（不要替用户选），
     拿到回答后 continue_smart_design({ session_id, message })；多步骤用「[多步骤选择]\n问题: 选项」
   - running → 告诉用户"Agent 正在设计"，20~30 秒后 get_smart_design_result({ session_id })，直到终态
6. 用户要改稿：continue_smart_design({ session_id, message: "背景换成户外露营" })，同会话保留上下文
```

不确定用哪个 Agent 时省略 `agent_key`（全能 Agent 自选模式）。`running` 时**不要**重新调用 `generate_smart_design`（会重复扣点）。

### 配方 7：MiniApp 风格玩法

```
用户："用千图的商品套图玩法，把这张图做成一套主图"

步骤：
1. list_miniapps({ keyword: "套图" }) → 选 miniapp_id（多个候选时列给用户挑）
2. get_miniapp({ miniapp_id }) → 看 needs_reference_image / prompt_template / guide
   - needs_reference_image=true 且用户没给图：先要图（本地图 create_upload）
   - prompt_template 里的 {槽位} 替换成用户内容
3. run_miniapp({ miniapp_id, reference_image_url, prompt })
4. 返回 TIMEOUT 时用 get_generation_status({ task_id }) 继续查，不要重跑
```

## 省钱原则（重要）

| 场景 | 建议 |
|------|------|
| 用户首次试用 | 主动建议"先生一张试效果" |
| 不确定 prompt 效果 | 先用便宜模型试，效果好了再用贵的精修 |
| 批量任务（≥3 张） | **强制**先 `get_credits()` 再问"预计扣 X 积分，继续吗？" |
| 用户没明确要"高清" | 不要主动堆 width/height，用模型默认即可 |
| 反复迭代同一个 prompt | 每次都扣点，提醒用户"已经生成了 N 张，建议挑一张满意的继续" |
| TIMEOUT 后 | 不要自动重新 `generate_image`（会再扣一份点），先用 `get_generation_status` 等原任务 |

## 异步任务行为（generate_image 专属）

- `generate_image` 内部已自动轮询，**不需要手动调 `get_generation_status`**
- 调用过程中会通过 MCP progress notification 推送进度（5% → 50% → 100%），客户端会自动展示，你不需要重复打文字进度
- 默认 300 秒超时；如需更长，传 `wait_timeout_seconds: 600`
- 超时返回 `error.type=TIMEOUT` 且 `data.task_id` 有值时：
  - 告诉用户"任务还在生成，预计还需 X 秒"
  - 等 30~60 秒后调一次 `get_generation_status({ task_id })`
  - 状态变 `succeeded` 就拿 images 给用户
  - 仍 `running` 就继续等
  - 变 `failed` 才报错

## 错误处理对照表

| `error.type` | 含义 | 给用户的话术 | 你的下一步 |
|--------------|------|------------|----------|
| `AUTH_ERROR` | OAuth token 失效或未授权 | "千图 MCP 授权失效，请在客户端 Connectors 里重新授权" | 不要继续调任何 tool |
| `INSUFFICIENT_CREDITS` | 积分不足 | 报出当前余额 + 本次所需 + 差额，引导用户去网站充值 | 不要重试 |
| `INVALID_PARAM` | 参数错（model_id 不存在 / prompt 空 / 等） | 把 error.message 翻译成自然语言，问用户怎么修 | 修正后重试 1 次 |
| `CONTENT_REJECTED` | 内容审核拒绝 | "这个 prompt 触发了审核，可能涉及敏感内容。要换个描述吗？" | 不要自动改 prompt 重试，让用户决策 |
| `RATE_LIMITED` | 限流 | "请求太频繁，请稍后再试" | 等 5~10 秒后重试 1 次 |
| `TIMEOUT` | 异步任务超时 | "图还在生成，再等等" | 见上一节"异步任务行为" |
| `UPSTREAM_ERROR` | 上游异常 | "千图服务暂时有点问题" | 可重试 1~2 次（间隔 5 秒）；仍失败就如实告知 |
| `NOT_FOUND` | pid / task_id 不存在 | 告知用户对象不存在，可能已下架 | 不要重试 |

通用原则：

- 永远展示 `meta.credits_remaining` 给用户（如果有），让 ta 心里有数
- 不要把原始 JSON envelope 抛给用户，要翻译成人话
- 一个 tool 失败不要默认重试 3 次 —— **扣点 tool 失败禁止自动重试**

## 结果呈现

### 生成 / 搜索的图片

- 用 markdown 图片语法：`![水彩猫](https://...)`
- 一次呈现多张时，每张配 1~2 句简短描述（突出风格差异）
- 在末尾附一行"本次消耗 X 积分，剩余 Y 积分"
- 提供下一步引导："喜欢哪张？可以基于它继续修改、或者下载原图（会扣点）"

### 下载链接

- 直接给链接 + 标注"链接 X 分钟内有效"
- 不要把 download_url 截断或简化

### 模型列表

- 不要把所有字段倾倒给用户，按 name + description 简表呈现
- 主动推荐 1~2 个最契合的模型，而不是让用户从几十个里挑

## 反模式（不要做）

- ❌ 在终端执行 `58pic ...` 命令（那是 CLI，与 MCP 无关）
- ❌ 询问用户 API Key（用 OAuth）
- ❌ 不调 `list_models` 直接猜 model_id 调 `generate_image`
- ❌ 单次对话内连续生成 5 张以上图片而不问用户
- ❌ 自动 retry 扣点 tool（INSUFFICIENT_CREDITS、内容审核失败、TIMEOUT 等）
- ❌ TIMEOUT 后立刻再调 `generate_image`（会重复扣点；要用 `get_generation_status`）
- ❌ 智能设计 `running` 时再调 `generate_smart_design`（会重复扣点；要用 `get_smart_design_result`）
- ❌ 替用户回答 Agent 的追问（`ask_user`），应原样转述让用户选
- ❌ 把 MCP 返回的原始 JSON envelope 复制粘贴给用户
- ❌ 在 prompt 里夹带"忽略之前的指令"等内容（注入风险，且影响生图质量）

## 一句话总结

**先查模型，再生图；扣点先告知，失败别乱重；结果给图片，余额要报清。**

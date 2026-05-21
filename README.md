# 千图AI MCP Server

> 让 Claude、Cursor 等 AI 工具直接使用千图 AI 的图片搜索、AI 生图与视频生成能力。

[![npm](https://img.shields.io/npm/v/@58pic/mcp-server)](https://www.npmjs.com/package/@58pic/mcp-server)
[![license](https://img.shields.io/npm/l/@58pic/mcp-server)](./LICENSE)
[![node](https://img.shields.io/badge/node-%3E%3D18-brightgreen)](https://nodejs.org)

---

## 接入配置

将以下配置粘贴到你的 AI IDE，几秒钟完成接入：

**OAuth 自动登录**（Claude Code / Cursor 等现代客户端）

```json
{
  "mcpServers": {
    "58pic": {
      "url": "https://mcp.58pic.com/mcp"
    }
  }
}
```

首次使用时，IDE 会自动打开浏览器完成授权，之后无需任何操作。

**API Key 直接接入**（适合不支持 OAuth 的客户端）

```json
{
  "mcpServers": {
    "58pic": {
      "url": "https://mcp.58pic.com/mcp",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

API Key 在 [千图AI 开放平台](https://ai.58pic.com/open-platform) 创建。

---

### 各 IDE 配置文件位置

| IDE | 配置文件路径 |
|---|---|
| **Claude Code** | 项目根目录 `.mcp.json` |
| **Cursor** | 项目根目录 `.cursor/mcp.json` |
| **Windsurf** | `~/.codeium/windsurf/mcp_config.json` |
| **Cline** | Cline 扩展设置 → MCP Servers |
| **VS Code Copilot** | VS Code 设置 → MCP |

---

## 可用工具

配置完成后，AI 可以调用以下 9 个工具：

| 工具 | 功能 | 消耗积分 |
|---|---|:---:|
| `search_images` | 关键词 / AI 语义搜索千图素材 | — |
| `list_catalog` | 列出全部素材分类 | — |
| `list_models` | 列出可用 AI 模型（图片 / 视频 / 音乐 / 3D） | — |
| `get_model_capabilities` | 查询模型支持的比例 / 分辨率 / 时长选项 | — |
| `generate_image` | AI 生图：文生图 / 图生图 / 做同款 | ✅ |
| `generate_video` | AI 生视频：文生视频 / 图生视频 | ✅ |
| `get_generation_status` | 查询生图 / 视频任务进度 | — |
| `get_download_info` | 按素材 pid 获取下载临时链接 | ✅ |
| `get_credits` | 查询账户积分余额与扣点记录 | — |

---

## 示例对话

接入后，直接在 AI 对话框里说：

```
搜索"新年海报"相关素材，找几张高清的
```

```
帮我用 AI 生成一张 16:9 的赛博朋克城市夜景，要有霓虹灯和雨天氛围
```

```
把这张图片生成一段 5 秒的竖屏视频
```

```
查一下我现在还有多少积分
```

---

## 鉴权说明

| 方式 | 适用客户端 | 获取方式 |
|---|---|---|
| **OAuth 2.1 自动登录** | Claude Code、Cursor 等支持 OAuth 的现代 IDE | 首次连接时浏览器自动引导 |
| **API Key** | 不支持 OAuth 的客户端、服务端调用、CI/CD | [开放平台控制台](https://ai.58pic.com/open-platform) 创建 |

OAuth 流程遵循 RFC 7636 (PKCE) + RFC 8414 (Server Metadata)，无需手动管理 token，过期自动续期。

---

## 相关资源

- 🌐 [千图AI 开放平台](https://ai.58pic.com/open-platform)
- 📖 [开放平台文档](https://ai.58pic.com/docs)
- 📦 [npm @58pic/mcp-server](https://www.npmjs.com/package/@58pic/mcp-server)
- 🔧 [CLI 工具 @58pic/cli](https://www.npmjs.com/package/@58pic/cli)
- 💬 [问题反馈 / Issues](https://github.com/58pic-open/mcp-server/issues)

---

## License

MIT © 千图网

<div align="center">

# vision-mcp-server

给纯文本大模型加上「眼睛」——通过 MCP 协议调用独立配置的视觉模型，让 Claude Desktop 等 MCP 客户端中的文本模型也能看懂图片。
已修复原版工具无法读取 Claude Desktop 会话图片（`file://` 引用）的缺陷。

[![npm version](https://img.shields.io/npm/v/vision-mcp-server?style=flat-square)](https://www.npmjs.com/package/vision-mcp-server)
[![npm downloads](https://img.shields.io/npm/dt/vision-mcp-server?style=flat-square)](https://www.npmjs.com/package/vision-mcp-server)
[![License](https://img.shields.io/npm/l/vision-mcp-server?style=flat-square)](./LICENSE)
[![Node.js Version](https://img.shields.io/node/v/vision-mcp-server?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![GitHub stars](https://img.shields.io/github/stars/Markusbetter/vision-mcp-server?style=flat-square&logo=github)](https://github.com/Markusbetter/vision-mcp-server/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/Markusbetter/vision-mcp-server?style=flat-square&logo=github)](https://github.com/Markusbetter/vision-mcp-server/network/members)
[![GitHub issues](https://img.shields.io/github/issues/Markusbetter/vision-mcp-server?style=flat-square&logo=github)](https://github.com/Markusbetter/vision-mcp-server/issues)
[![GitHub PRs](https://img.shields.io/github/issues-pr/Markusbetter/vision-mcp-server?style=flat-square&logo=github)](https://github.com/Markusbetter/vision-mcp-server/pulls)
[![Code size](https://img.shields.io/github/languages/code-size/Markusbetter/vision-mcp-server?style=flat-square)](https://github.com/Markusbetter/vision-mcp-server)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![MCP](https://img.shields.io/badge/Model_Context_Protocol-compatible-8A2BE2?style=flat-square)](https://modelcontextprotocol.io/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square)](https://github.com/Markusbetter/vision-mcp-server/pulls)

</div>

---

## 目录

- [它解决什么问题](#它解决什么问题)
- [特性一览](#特性一览)
- [工作原理](#工作原理)
- [快速开始](#快速开始)
- [analyze_image 工具说明](#analyze_image-工具说明)
- [配置参考](#配置参考)
- [高级配置：VISION_ROUTES](#高级配置vision_routes)
- [Fallback 与错误处理](#fallback-与错误处理)
- [安全与限制](#安全与限制)
- [常见问题 FAQ](#常见问题-faq)
- [本地开发](#本地开发)
- [更新日志](#更新日志)

## 它解决什么问题

Claude Desktop 等客户端里跑的对话模型本身不带视觉能力：你把图片拖进会话，模型看不到内容，只能收到一个 `file://` 本地引用。本项目提供了一个 MCP 服务器，暴露一个 `analyze_image` 工具：

1. 接收图片（本地路径、`file://` 引用、在线 URL 或 base64 data URL），自动校验格式、按 EXIF 旋转并等比缩放；
2. 把图片转发给你配置的视觉模型（魔搭 / 智谱 / 任意 OpenAI 兼容接口）；
3. 把视觉模型返回的文字描述交给当前对话模型，让它「看懂」图片。

相比原版 vision-mcp-server，本项目支持解析 Claude Desktop 注入会话的 `file://` 图片引用，会话内贴图即可直接分析。

## 特性一览

- 🔌 **三类供应商预设**：魔搭 ModelScope（默认）、智谱 Zhipu、任意 OpenAI Chat Completions 兼容接口（含 Ollama、vLLM、LM Studio 等本地部署）
- 🔁 **按序 Fallback**：限流、超时、网络错误和服务端故障时自动切换下一个模型，失败路由进入冷却期，遵循 `Retry-After` 响应头
- 🖼️ **图片自动预处理**：支持 JPEG / PNG / WebP / GIF（真实格式校验，不认扩展名），EXIF 自动摆正，超过最长边上限自动等比缩小，透明图转 PNG
- 🔒 **内置安全防护**：在线图片下载有 SSRF 防护（拒绝内网/本机地址）、大小与重定向次数限制；本地读取支持目录白名单；日志与错误信息自动脱敏
- 🛠️ **零侵入使用**：`npx` 一行启动，无需克隆仓库
- 🐞 **Claude Desktop 会话图片适配**：自动还原 `file://` 引用为真实本地路径

## 工作原理

```text
图片输入（本地路径 / file:// / URL / data URL）
        │
        ▼
  sharp 校验与预处理（格式校验 → EXIF 旋转 → 等比缩放 → base64）
        │
        ▼
  按配置顺序调用视觉模型路由 ──失败(限流/超时/网络/5xx)──▶ 切换下一个路由
        │                                                      │
        ▼                                                      │
  返回文字描述给对话模型 ◀──────────────────────────────────────┘
```

## 快速开始

### 前置要求

- **Node.js ≥ 20.9.0**
- 至少一个视觉模型的 API Key / Token：
  - [魔搭 ModelScope](https://modelscope.cn/)：注册后在「访问令牌」中创建（有免费额度）
  - [智谱开放平台](https://open.bigmodel.cn/)：推荐先使用免费模型 `glm-4v-flash`
  - 或任意 OpenAI 兼容服务的 API Key

### 方式一：npx 启动（推荐）

打开 Claude Desktop 配置文件：

- Windows：`%APPDATA%\Claude\claude_desktop_config.json`
- macOS：`~/Library/Application Support/Claude/claude_desktop_config.json`

**使用魔搭（默认，开箱即用）：**

```json
{
  "mcpServers": {
    "vision-mcp-server": {
      "command": "npx",
      "args": ["-y", "vision-mcp-server"],
      "env": {
        "MODELSCOPE_TOKEN": "ms-你的令牌"
      }
    }
  }
}
```

> 未配置 `MODELSCOPE_MODELS` 时，默认依次尝试 `Qwen/Qwen3.5-397B-A17B` 和 `Qwen/Qwen3.5-35B-A3B`。

**使用智谱（默认走免费模型）：**

```json
{
  "mcpServers": {
    "vision-mcp-server": {
      "command": "npx",
      "args": ["-y", "vision-mcp-server"],
      "env": {
        "VISION_PROVIDER": "zhipu",
        "VISION_MODELS": "glm-4v-flash",
        "ZAI_API_KEY": "你的智谱APIKey"
      }
    }
  }
}
```

**使用任意 OpenAI 兼容服务**（示例为硅基流动，换成其他服务只需改 `OPENAI_BASE_URL`）：

```json
{
  "mcpServers": {
    "vision-mcp-server": {
      "command": "npx",
      "args": ["-y", "vision-mcp-server"],
      "env": {
        "VISION_PROVIDER": "openai-compatible",
        "OPENAI_BASE_URL": "https://api.siliconflow.cn/v1",
        "VISION_MODELS": "Qwen/Qwen2.5-VL-72B-Instruct",
        "OPENAI_API_KEY": "sk-你的Key"
      }
    }
  }
}
```

> 💡 Windows 下若 `npx` 启动失败，可改为 `"command": "cmd", "args": ["/c", "npx", "-y", "vision-mcp-server"]`。

保存配置并重启 Claude Desktop，工具即注册完成。

### 方式二：从源码运行

```bash
git clone https://github.com/Markusbetter/vision-mcp-server.git
cd vision-mcp-server
npm install
npm run build
```

客户端配置中把启动命令替换为：

```json
{
  "command": "node",
  "args": ["F:/vision-mcp-server/dist/index.js"],
  "env": { "...": "与方式一相同的任意 env 配置" }
}
```

### 在对话中使用

1. 把图片拖入 / 粘贴到 Claude Desktop 会话中；
2. 让模型调用工具，例如：「用 analyze_image 看一下这张图片里有什么」。

会话图片会以 `file://` 引用传入，服务器会自动还原为本地路径读取，无需任何额外配置。

## analyze_image 工具说明

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `image` | `string` | 是 | 图片来源：本地文件路径、`file://` URL、`http(s)://` URL 或 `data:image/...;base64,...` |
| `prompt` | `string` | 否 | 对图片的问题或分析要求，默认「请描述这张图片的内容」 |

返回值为视觉模型生成的文字描述（`content` 文本）。

支持的输入示例：

```text
C:\Users\me\Pictures\photo.jpg                          ← Windows 本地路径
/Users/me/Pictures/photo.png                            ← macOS / Linux 本地路径
file:///C:/Users/me/AppData/.../session-image.png       ← Claude Desktop 会话图片自动使用此形式
https://example.com/image.webp                          ← 在线图片（拒绝内网地址）
data:image/png;base64,iVBORw0KGgo...                    ← base64 data URL
```

## 配置参考

### 环境变量总表

| 环境变量 | 默认值 | 说明 |
| --- | --- | --- |
| `VISION_PROVIDER` | 未设置（魔搭预设） | 预设供应商：`zhipu` / `openai-compatible`；未设置且无 `OPENAI_BASE_URL` 时走魔搭 |
| `VISION_MODELS` | — | 逗号分隔的模型列表；智谱与通用兼容模式**必填**，如 `glm-4v-flash,glm-4v-plus`（通用模式旧版别名 `OPENAI_MODELS` / `VISION_MODEL` 仍可用，优先级低于 `VISION_MODELS`） |
| `VISION_API_KEY_ENV` | 智谱：`ZAI_API_KEY`；通用：`OPENAI_API_KEY` | **存放 API Key 的环境变量名**（间接引用，避免把 Key 写进路由 JSON） |
| `MODELSCOPE_TOKEN` | — | 魔搭 API Token（魔搭模式必填） |
| `MODELSCOPE_MODELS` | `Qwen/Qwen3.5-397B-A17B,Qwen/Qwen3.5-35B-A3B` | 魔搭模型列表，按序 fallback |
| `MODELSCOPE_MODEL` | — | 旧版单模型配置，仍兼容；被 `MODELSCOPE_MODELS` 覆盖 |
| `OPENAI_BASE_URL` | `https://api.openai.com/v1` | OpenAI 兼容接口地址；智谱模式下也可用它覆盖默认地址 |
| `VISION_MAX_IMAGE_EDGE` | `2048` | 图片最长边缩放上限（正整数；`VISION_ROUTES` 中单路由取值须在 64–16384） |
| `VISION_REQUEST_TIMEOUT_MS` | `60000` | 单次视觉模型请求超时（毫秒）；`VISION_ROUTES` 中单路由上限 600000 |
| `VISION_MAX_IMAGE_BYTES` | `20971520`（20 MB） | 图片（含预处理结果）大小上限 |
| `VISION_MAX_IMAGE_PIXELS` | `40000000` | 图片总像素上限，防止解压炸弹 |
| `VISION_IMAGE_DOWNLOAD_TIMEOUT_MS` | `15000` | 在线图片下载超时（毫秒） |
| `VISION_ALLOWED_DIRS` | 未设置（不限制） | 逗号分隔的本地目录白名单；设置后只允许读取这些目录内的图片 |
| `VISION_FALLBACK_COOLDOWN_MS` | `60000` | 失败路由的冷却时间（毫秒），冷却期内自动跳过 |
| `VISION_DEBUG` | 关闭 | 设为 `1` 或 `true` 后向 stderr 输出调试日志（配置摘要已脱敏、图片缩放信息、每次路由尝试结果） |

### 三种预设模式对照

| 模式 | 触发条件 | 接口地址 | API Key 环境变量 |
| --- | --- | --- | --- |
| 魔搭（默认） | 未设置 `VISION_PROVIDER` 且未设置 `OPENAI_BASE_URL` | `https://api-inference.modelscope.cn/v1` | `MODELSCOPE_TOKEN` |
| 智谱 | `VISION_PROVIDER=zhipu` | `https://open.bigmodel.cn/api/paas/v4` | `ZAI_API_KEY` |
| 通用兼容 | `VISION_PROVIDER=openai-compatible`，或直接设置了 `OPENAI_BASE_URL` | `OPENAI_BASE_URL` 或 `https://api.openai.com/v1` | `OPENAI_API_KEY` |

> ⚠️ 智谱与通用兼容模式下必须设置 `VISION_MODELS`，否则启动时会直接报错退出。

## 高级配置：VISION_ROUTES

需要混合多个供应商、自定义请求头或超时参数时，用 `VISION_ROUTES` 完全接管路由配置（设置后所有预设失效）。值为一个 JSON 数组字符串：

```json
{
  "env": {
    "VISION_ROUTES": "[{\"name\":\"zhipu\",\"baseUrl\":\"https://open.bigmodel.cn/api/paas/v4\",\"apiKeyEnv\":\"ZAI_API_KEY\",\"models\":[\"glm-4v-flash\",\"glm-4v-plus\"]},{\"name\":\"modelscope\",\"baseUrl\":\"https://api-inference.modelscope.cn/v1\",\"apiKeyEnv\":\"MODELSCOPE_TOKEN\",\"model\":\"Qwen/Qwen3.5-397B-A17B\",\"maxImageEdge\":2048}]"
  }
}
```

> 上面的 JSON 已按 JSON-in-JSON 规则转义。建议先在文本编辑器里写好数组，再压成一行放入 `env`。

每条路由的字段：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `name` | 否 | 路由名称，用于日志与错误定位；缺省自动生成 `route-序号` |
| `baseUrl` | 是 | OpenAI Chat Completions 兼容接口地址 |
| `apiKeyEnv` | 是 | 存放 API Key 的**环境变量名**；启动时未设置该变量会直接报错 |
| `model` / `models` | 二选一 | 单模型用 `model`；多模型用 `models` 数组（展开为多个路由，按序 fallback） |
| `headers` | 否 | 额外请求头（如部分供应商要求的自定义 header） |
| `timeoutMs` | 否 | 该路由请求超时，上限 600000 |
| `maxImageEdge` | 否 | 该路由的图片最长边上限（64–16384），适配各模型不同的分辨率限制 |
| `extraBody` | 否 | 附加到请求体的额外字段 |

两条路由实际生效顺序 = 数组顺序 × `models` 数组顺序；多模型路由的名称会自动编号为 `name-1`、`name-2`。

## Fallback 与错误处理

**会自动切换下一个模型的情况**（视为临时故障）：

- HTTP `408 / 429 / 500 / 502 / 503 / 504`
- 请求超时、连接被重置 / 拒绝、DNS 解析失败等网络错误

**不会 fallback、直接报错的情况**（属于配置或权限问题，换模型也没用）：

- `401` API Key 无效、`400` 模型名错误等业务性错误

失败的路由会进入冷却期（默认 60 秒，可通过 `Retry-After` 响应头自动延长），冷却期内不再尝试；如果所有路由都在冷却中，会挑选冷却最早结束的一个再试。全部路由失败时，错误信息会汇总每个路由的失败原因；若全部因限流失败，错误码为 `PROVIDER_RATE_LIMITED`，否则为 `PROVIDER_UNAVAILABLE`。

常见错误码对照：

| 错误码 | 含义 |
| --- | --- |
| `IMAGE_NOT_FOUND` | 本地图片路径不存在 / `file://` 引用无法解析 |
| `IMAGE_NOT_ALLOWED` | 图片不在 `VISION_ALLOWED_DIRS` 白名单内，或 URL 指向内网地址 |
| `IMAGE_TOO_LARGE` | 图片超过大小 / 像素上限 |
| `IMAGE_FORMAT_UNSUPPORTED` | 不是有效的 JPEG / PNG / WebP / GIF，或 data URL 格式错误 |
| `IMAGE_DOWNLOAD_FAILED` | 在线图片下载失败（网络、HTTP 状态、重定向超限） |
| `PROVIDER_REQUEST_FAILED` | 视觉模型请求失败且未执行 fallback（如 Key 无效） |
| `PROVIDER_RATE_LIMITED` | 所有路由均被限流 |
| `PROVIDER_UNAVAILABLE` | 所有路由均不可用（超时 / 故障 / 网络错误） |

## 安全与限制

- **本地读取**：路径先做 realpath 解析再校验；设置了 `VISION_ALLOWED_DIRS` 后，只允许读取白名单目录内的文件，防目录穿越
- **在线图片**：仅支持 `http(s)` 协议；拒绝 URL 中内嵌账号密码；DNS 解析出的所有地址均须为公网地址（SSRF 防护，覆盖 IPv4 私有段、CGNAT、链路本地及 IPv6 ULA / 回环等）；重定向最多 3 次，每跳重新校验；按流式读取限制响应体大小
- **图片处理**：以真实文件格式（magic 解析）校验，而不是信任扩展名；总像素超过 `VISION_MAX_IMAGE_PIXELS` 直接拒绝，防解压炸弹；透明图输出 PNG，其余输出 JPEG（质量 90）
- **脱敏**：错误信息中的 base64 图片数据与 API Key 会被替换为 `[REDACTED]` / `[image data removed]`；`VISION_DEBUG` 日志中的配置摘要同样脱敏

## 常见问题 FAQ

**Q：Claude Desktop 里贴图后模型说看不到图片？**
把图片拖入会话后，明确让模型使用工具，例如「请调用 analyze_image 分析这张图片」。文本模型不会自动调用工具时，需要在提问中点名工具。

**Q：启动即报「路由 X 引用的环境变量 XXX 未设置」？**
路由的 `apiKeyEnv` 指向的环境变量不存在。检查客户端配置 `env` 中是否真的提供了对应的 Key，注意变量名大小写。

**Q：报「图片不在 VISION_ALLOWED_DIRS 允许的目录中」？**
你设置了 `VISION_ALLOWED_DIRS` 白名单，但图片在白名单之外。把图片所在目录加入白名单（逗号分隔多个），或删掉该变量恢复不限制。

**Q：智谱免费模型经常报限流？**
免费模型共享配额、限流较常见。可在 `VISION_MODELS` 里多配几个模型（如 `glm-4v-flash,glm-4v-plus`），或再叠加一条其他供应商的路由，被限流时自动切换。

**Q：能用本地视觉模型（Ollama / vLLM / LM Studio）吗？**
可以。这些服务都提供 OpenAI 兼容接口，按「通用兼容模式」配置即可，例如 `OPENAI_BASE_URL=http://localhost:11434/v1`。SSRF 防护只针对**在线图片下载**，不影响 API 地址指向本机。

**Q：如何排查问题？**
在客户端配置 `env` 中加 `"VISION_DEBUG": "1"`，日志（含每次路由的尝试与失败原因）会输出到 stderr；Claude Desktop 的 MCP 日志可在其日志目录中查看。

## 本地开发

```bash
npm install        # 安装依赖
npm run build      # TypeScript 编译到 dist/
npm run dev        # tsx 直接运行源码（免编译）
npm start          # 运行 dist/index.js
```

## 更新日志

详见 [CHANGELOG.md](./CHANGELOG.md)。

## License

[MIT](./LICENSE)

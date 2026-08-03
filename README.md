# kimi-vision-mcp 🔍

> 让任何 MCP 客户端（Claude Code / OpenCode 等）拥有视觉能力 —— 通过 Kimi Vision API 理解图片内容

**kimi-vision-mcp** 是一个 MCP (Model Context Protocol) 服务器，它让 LLM 编程助手能够"看见"并理解图片 —— 截图、图表、照片、UI 界面、错误弹窗、游戏画面等等。

> ⚠️ **当前仅支持 Kimi 视觉模型**（通过 Moonshot Anthropic 兼容端点）。暂不支持 GPT-4o / Gemini / Claude Vision 等其他模型。详见下方 [模型兼容性](#-模型兼容性)。

## 🎯 它能做什么

| 场景 | 示例 |
|------|------|
| 🖼️ 截图分析 | 把 GitHub Actions 报错截图发给助手，它直接告诉你哪里出了问题 |
| 🎮 游戏画面 | 截取 Unity/Unreal 运行画面，让助手分析渲染 Bug |
| 📊 图表理解 | 发一张流程图/架构图，助手帮你解读 |
| 🐛 错误排查 | 错误弹窗截图直接分析，不用手动抄错误信息 |
| 🌐 网页分析 | 网页截图发给助手，帮你分析布局、内容、问题 |

## 🏗️ 工作原理

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  MCP Client  │────▶│  kimi-vision-mcp│────▶│  Kimi Vision API │
│ (Claude/OC)  │◀────│  (MCP Server)   │◀────│  (Moonshot)      │
└──────────────┘     └─────────────────┘     └──────────────────┘
```

1. 客户端调用 `describe_image` 工具，传入图片路径
2. kimi-vision-mcp 读取图片，缩放压缩后 Base64 编码发给 Kimi Vision API
3. Kimi 返回图片的文字描述
4. LLM 基于描述进行分析、回答、建议

**为什么用 Kimi 而不是其他模型？**
- Kimi 的 Anthropic 兼容端点 → 可直接用 `anthropic` Python SDK
- 中文理解能力强，适合中文用户
- 价格实惠，Moonshot 平台注册即可使用

---

## 🧠 模型兼容性

### 当前支持

| 模型 | 状态 | 提供商 | 端点 |
|------|------|--------|------|
| **Kimi K2.6** | ✅ 默认 | Moonshot | `api.moonshot.cn/anthropic` |

### 为什么不支持其他模型？

本项目利用的是 Kimi 的 **Anthropic 兼容端点** —— 它允许直接用 `anthropic` Python SDK 调用 Kimi 模型，无需额外适配层。

| 模型 | 能否支持 | 原因 |
|------|----------|------|
| GPT-4o / GPT-4V | ❌ 不支持 | OpenAI 格式，需 `openai` SDK 重写 |
| Gemini 2.5 Pro | ❌ 不支持 | Google 格式，需另写适配 |
| Claude Vision | ❌ 不支持 | Anthropic 原生 Vision API 价格高，且多数客户端已有内置图像能力 |
| moonshot-v1-* | ❌ 不支持 | 经实测 `400 Invalid request: Image input not supported`，只有 kimi-k2.6 支持视觉 |
| 其他 Kimi 模型 | 🟡 理论支持 | 可在 `env` 中设置 `VISION_MODEL` 切换，参考 [Moonshot 文档](https://platform.moonshot.cn/docs) |

### 未来计划

- [ ] 支持 OpenAI 兼容端点（GPT-4o 等）
- [ ] 支持自定义端点 URL（自部署模型）
- [ ] 多模型自动 fallback

> 💡 **欢迎 PR！** `server.py` 约 110 行，添加新模型支持只需新增一个 client 分支。

---

## 📋 前置条件

### 1. Python 环境

需要 Python 3.10+：

```bash
python --version  # 应该 >= 3.10
```

### 2. Moonshot API Key

1. 打开 [https://platform.moonshot.cn](https://platform.moonshot.cn)
2. 注册/登录账号
3. 进入控制台 → API Keys → 创建新的 API Key
4. 复制密钥（格式：`sk-xxxxxxxxxxxxxxxxxxxxxxxx`）

> ⚠️ **API Key 是敏感信息，永远不要提交到 Git 或公开分享！**

---

## 📦 安装

### 步骤 1：克隆仓库

```bash
git clone https://github.com/ShaLuuFPS/kimi-vision-mcp.git
cd kimi-vision-mcp
```

### 步骤 2：安装依赖

```bash
pip install -r requirements.txt
```

包含三个依赖：`mcp`（MCP 框架）、`anthropic`（调用 Kimi 兼容端点）、`Pillow`（图片缩放压缩）。

或者用虚拟环境（推荐）：

```bash
python -m venv .venv

# Windows
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
```

### 步骤 3：配置 MCP 客户端

#### 方式 A：OpenCode（已实测）

编辑 OpenCode 配置文件（全局 `~/.config/opencode/opencode.jsonc` 或项目根目录的 `opencode.json`）：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "vision-bridge": {
      "type": "local",
      "command": ["python", "/你的/路径/kimi-vision-mcp/server.py"],
      "environment": {
        "MOONSHOT_API_KEY": "sk-你的密钥"
      },
      "timeout": 120000   // ⚠️ 必加，见下方说明
    }
  }
}
```

#### 方式 B：Claude Code

编辑 `~/.claude/mcp.json`（Windows 上是 `C:\Users\你的用户名\.claude\mcp.json`）：

```json
{
  "mcpServers": {
    "vision-bridge": {
      "type": "stdio",
      "command": "python",
      "args": ["/你的/路径/kimi-vision-mcp/server.py"],
      "env": {
        "MOONSHOT_API_KEY": "sk-你的密钥"
      }
    }
  }
}
```

> ⚠️ **把路径和密钥替换成你自己的！** MCP server 注册名（上例中的 `vision-bridge`）可以自定义，不影响功能。

### 步骤 4：验证安装

重启客户端，然后试试：

```
请描述这张图片：C:\Users\Administrator\Desktop\screenshot.png
```

如果返回了图片描述，说明配置成功 ✅

---

## ⏱️ 超时配置（重要）

Kimi K2.6 的视觉请求较慢，**单张图片通常需要 10~18 秒**。多数 MCP 客户端的默认工具调用超时太短（OpenCode 默认仅 5 秒），会直接报错：

```
MCP error -32001: Request timed out
```

### 各客户端解决方案

| 客户端 | 配置项 | 推荐值 |
|--------|--------|--------|
| **OpenCode** | `mcp.<name>.timeout` | `120000`（ms）|
| **OpenCode（全局）** | `experimental.mcp_timeout` | `120000`（ms）|
| **Claude Code** | 一般无此问题，如遇卡顿参考 [issue](https://github.com/ShaLuuFPS/kimi-vision-mcp/issues) |

OpenCode 完整示例（per-server + 全局双保险）：

```jsonc
{
  "mcp": {
    "vision-bridge": {
      "type": "local",
      "command": ["python", "/path/to/server.py"],
      "environment": { "MOONSHOT_API_KEY": "sk-xxx" },
      "timeout": 120000
    }
  },
  "experimental": {
    "mcp_timeout": 120000
  }
}
```

> 服务端（server.py）内部已设 SDK 超时为 120s，所以客户端只要 ≥ 120s 即可。

---

## 📊 性能实测

以下数据基于 6 张游戏截图（~230KB/张）实测：

| 测试项 | 结果 |
|--------|------|
| 单图请求（max_tokens=512）| ~18.5s |
| 单图请求（max_tokens=16）| ~1.7s —— 瓶颈在图片理解，非文本生成 |
| **6 图并行调用** | **总耗时 ~11.8s，全部成功** |
| 并行单图耗时区间 | 10.4s ~ 11.8s |

**结论：** Moonshot 服务端支持真正的并发，6 图并行只比单图多约 1s。多图场景放心并发调用。

> 图片在发送前会统一缩放到最大边 768px + JPEG quality=85 压缩（见 server.py `_encode_image`），显著减少传输与推理时间。

---

## 🔧 配置说明

### 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `MOONSHOT_API_KEY` | **必填** - Moonshot API 密钥 | 无 |
| `ANTHROPIC_BASE_URL` | API 端点地址 | `https://api.moonshot.cn/anthropic` |
| `VISION_MODEL` | 使用的视觉模型 | `kimi-k2.6` |

### 支持的图片格式

PNG · JPG/JPEG · GIF · WebP · BMP

> 任意格式都会先用 Pillow 打开并转码为 JPEG 发送，所以格式本身不影响 Kimi 侧兼容性。

### 更换模型

在 `env` 中设置 `VISION_MODEL`：

```json
"env": {
  "MOONSHOT_API_KEY": "sk-xxx",
  "VISION_MODEL": "kimi-k2.6"
}
```

可用的 Kimi 视觉模型请查看 [Moonshot 官方文档](https://platform.moonshot.cn/docs)。

---

## 📖 使用示例

### 示例 1：分析错误截图

```
这张 Unity 报错截图是什么意思？怎么修？
C:\Users\Administrator\Desktop\unity-error.png
```

### 示例 2：理解架构图

```
帮我分析这张系统架构图，列出各个模块的职责
C:\projects\docs\architecture.png
```

### 示例 3：检查 UI 设计

```
对比这张 UI 截图和设计稿，找出不一致的地方
C:\projects\screenshots\login-page.png
```

### 示例 4：针对性提问

```
这张图中红色圈出来的按钮是什么文字？
C:\photos\remote-control.jpg
```

### 示例 5：多图并行（OpenCode）

在同一条消息里同时引用多张图片，助手会并发调用，速度近似单图：

```
帮我分别描述这 6 张游戏画面：
C:\pic\boss.png
C:\pic\juese.png
C:\pic\shangcheng.png
...
```

---

## 🐛 常见问题

### Q: 提示 `MCP error -32001: Request timed out`

**最常见问题。** Kimi 视觉请求需 10~18s，超过了客户端默认超时。解决：调大客户端的 MCP 超时到 120000ms，详见 [⏱️ 超时配置](#-超时配置重要)。

### Q: 提示 "MOONSHOT_API_KEY not set"

API Key 没配置或配错了：
- 检查配置文件中 `environment`/`env.MOONSHOT_API_KEY` 是否正确
- 确认密钥格式：以 `sk-` 开头
- 确认 Moonshot 账户余额充足

### Q: 提示 "File not found"

- 图片路径必须是**绝对路径**，不能是相对路径
- 路径中包含中文时确保编码正确

### Q: 提示 "Kimi API error"

- 检查网络是否能访问 `api.moonshot.cn`（国内用户一般没问题）
- 海外用户可能需要配置代理
- 确认 API Key 还有额度
- 若报 `Image input not supported`，说明你把 `VISION_MODEL` 改成了不支持视觉的模型（如 moonshot-v1-*），改回 `kimi-k2.6`

### Q: `ModuleNotFoundError: No module named 'PIL'`

依赖未装全，重新执行 `pip install -r requirements.txt`（确保包含 Pillow）。

### Q: 客户端里看不到 describe_image 工具

1. 确认配置文件路径与 JSON 语法正确
2. 重启客户端
3. OpenCode 用 `opencode mcp list` 查看；Claude Code 输入 `/mcp` 查看

---

## 🛠️ 开发

### 本地测试

```bash
# 直接运行（stdin/stdout JSON-RPC 模式）
python server.py
```

### 项目结构

```
kimi-vision-mcp/
├── server.py          # 主程序 —— 只有一个文件（~110 行）
├── requirements.txt   # Python 依赖（mcp / anthropic / Pillow）
├── .gitignore         # 防止意外提交敏感文件
├── LICENSE            # MIT 许可证
└── README.md          # 本教程
```

### 扩展

想支持更多功能？`server.py` 只有一个文件，很容易修改：

- **调整缩放尺寸** → 修改 `MAX_DIM`（默认 768，越小越快但越糊）
- **调整压缩质量** → 修改 `_encode_image` 中的 `quality=85`
- **调整输出风格** → 修改 `prompt` 变量
- **支持更多模型** → 修改 `MODEL` 环境变量
- **添加新工具** → 用 `@mcp.tool()` 装饰器添加新函数

---

## 📄 许可证

MIT License — 详见 [LICENSE](LICENSE)

## 🙏 鸣谢

- [Anthropic MCP SDK](https://github.com/modelcontextprotocol/python-sdk) — MCP Python 框架
- [Moonshot AI](https://www.moonshot.cn/) — Kimi 视觉模型

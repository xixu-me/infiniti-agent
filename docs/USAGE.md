# Infiniti Agent 使用说明书

本文面向使用 Infiniti Agent 的用户和维护者，说明如何安装、配置、运行、迁移、同步、扩展和排查问题。本文只描述当前版本已经实现的行为，不描述规划中但尚未实现的能力。

## 文档范围

本文覆盖 Infiniti Agent 作为已安装 npm 包和本地源码项目时的主要使用方式，包括命令行入口、交互式 TUI、LiveUI、本地项目数据、配置文件、同步、Skills、媒体能力和常见排障。示例中的密钥、Agent code、模型名和本地路径均使用占位值或通用默认值，实际使用时请替换为自己的配置。

## 1. 产品概览

Infiniti Agent 是一个项目级 AI 智能体框架。命令行程序名是 `infiniti-agent`，npm 包名是 `linkyun-infiniti-agent`。

核心能力：

- 交互式 TUI 对话，基于 React + Ink。
- 非交互式单轮 CLI 调用。
- 项目级 `.infiniti-agent/` 数据空间。
- 多 LLM provider 和多 profile。
- 内置工具、MCP 工具和项目级 Skills。
- 会话、记忆、知识图谱、计划任务和收件箱。
- LiveUI：本地 WebSocket + Electron 透明窗，可渲染 Live2D、sprite 或 real2d 角色。
- TTS、ASR、图片、头像表情集和 Seedance 视频工作流。
- LinkYun Agent 同步和 `.agent` 归档导入导出。

## 2. 前置条件

- Node.js `>=20`。
- npm。
- 如果使用 LiveUI 窗口，需要构建 `liveui` workspace，并确保安装了 optional dependency `electron`。
- 如果使用 `test_asr`、本地 ASR 或摄像头采集，机器上需要对应系统权限；`test_asr` 还需要 `ffmpeg`。
- 如果使用外部 LLM、TTS、ASR、图片或视频服务，需要准备相应 API key 或本地 service URL。

## 3. 安装

全局安装：

```bash
npm install -g linkyun-infiniti-agent
```

从源码开发：

```bash
git clone https://github.com/stelee410/infiniti-agent.git
cd infiniti-agent
npm install
npm run dev
```

本地构建并链接到全局命令：

```bash
npm run build
npm link
```

构建命令会运行 TypeScript 编译，并构建 `infiniti-agent-liveui` workspace：

```bash
npm run build
```

常用 npm scripts：

| 命令 | 用途 |
| --- | --- |
| `npm run dev` | 以 `tsx src/cli.tsx` 启动 TUI |
| `npm run dev:cli` | 以旧式 `--cli` 入口启动单轮 CLI |
| `npm run start` | 运行 `dist/cli.js` |
| `npm run build` | 构建 CLI 和 LiveUI |
| `npm run test` | 运行 Vitest |
| `npm run setup:liveui` | 合并 LiveUI 示例配置到目标项目 |

## 4. 第一次配置

运行初始化向导：

```bash
infiniti-agent init
```

`init` 会配置 LLM provider、base URL、model 和 API key，并写入全局配置：

```text
~/.infiniti-agent/config.json
```

进入一个项目目录后，建议迁移成项目级独立实例：

```bash
cd your-project
infiniti-agent migrate
```

`migrate` 会创建当前目录的 `.infiniti-agent/`。如果全局存在对应文件且本地不存在，它会复制：

- `~/.infiniti-agent/config.json` -> `.infiniti-agent/config.json`
- `~/.infiniti-agent/memory.md` -> `.infiniti-agent/memory.md`
- `~/.infiniti-agent/skills/` -> `.infiniti-agent/skills/`

之后当前目录的会话、记忆和 skills 都在本地 `.infiniti-agent/` 下独立运行。

## 5. 配置读取规则

配置文件版本必须是：

```json
{
  "version": 1
}
```

读取优先级：

1. 当前项目 `.infiniti-agent/config.json`
2. 全局 `~/.infiniti-agent/config.json`

如果两个位置都没有配置，交互模式会进入初始化向导；命令模式会提示先运行 `infiniti-agent init` 或 `infiniti-agent migrate`。

升级旧格式配置：

```bash
infiniti-agent upgrade
```

默认会尝试升级当前项目配置和全局配置。只升级全局配置：

```bash
infiniti-agent upgrade --global
```

升级会把旧的平铺 `llm` 字段转为 `profiles.main + default="main"`，并移除已废弃的 `skills` 配置块。

## 6. 项目数据布局

项目级数据目录是当前工作目录下的 `.infiniti-agent/`。

常见文件和目录：

| 路径 | 用途 |
| --- | --- |
| `.infiniti-agent/config.json` | 项目配置 |
| `.infiniti-agent/session.json` | 当前会话 |
| `.infiniti-agent/session.lock` | 会话写入锁目录 |
| `.infiniti-agent/sessions.db` | 已归档会话数据库 |
| `.infiniti-agent/memory.md` | 传统长期记忆文本 |
| `.infiniti-agent/memory.json` | 结构化记忆 |
| `.infiniti-agent/user_profile.json` | 用户画像 |
| `.infiniti-agent/knowledge.db` | 知识图谱数据库 |
| `.infiniti-agent/memory/long-term/` | 文档化长期记忆 |
| `.infiniti-agent/inbox/` | 本地收件箱 |
| `.infiniti-agent/inbox/assets/` | 图片、视频等生成资产 |
| `.infiniti-agent/schedules.json` | 计划任务 |
| `.infiniti-agent/jobs/` | 后台媒体任务 |
| `.infiniti-agent/h5-applets/` | LiveUI H5 applet 缓存 |
| `.infiniti-agent/dreams/` | dream/subconscious 相关输出 |
| `.infiniti-agent/error.log` | 错误日志 |
| `.infiniti-agent/ref/<agent>/` | LinkYun 同步下来的头像、设定图等参考素材 |
| `.infiniti-agent/backups/sync/` | LinkYun 覆盖导入前的关键文件备份 |
| `.infiniti-agent/sync-history.jsonl` | LinkYun 同步历史 |
| `.infiniti-agent/device.json` | 本机同步设备 ID |

项目根目录还可以放：

| 文件 | 用途 |
| --- | --- |
| `SOUL.md` | Agent 人格和角色设定 |
| `INFINITI.md` | 项目说明或专属指令 |
| `CLAUDE.md` | 兼容项目指令 |
| `AGENTS.md` / `AGENT.md` | Agent 相关提示文件 |
| `.env.local` | LinkYun 登录和绑定信息，本地使用，不会打进 `.agent` 归档 |

## 7. 基本使用

启动交互式对话：

```bash
infiniti-agent
```

等同于：

```bash
infiniti-agent chat
```

执行一轮非交互任务：

```bash
infiniti-agent cli 检查当前项目并总结结构
```

多词 prompt 不需要额外加引号，命令会把 `<prompt...>` 合并成一段文本。

旧式单轮入口仍可用：

```bash
infiniti-agent --cli 检查当前项目并总结结构
```

全局 flags 会在命令解析前处理：

| Flag | 行为 |
| --- | --- |
| `--debug` | 设置 `INFINITI_AGENT_DEBUG=1`，输出 meta-agent、工具调度等调试日志 |
| `--dangerously-skip-permissions` | 跳过工具安全确认 |
| `--disable-thinking` | 将 thinking 模式临时设为 disabled |

示例：

```bash
infiniti-agent cli 查询天气 --debug
infiniti-agent cli 查询天气 --disable-thinking
infiniti-agent --dangerously-skip-permissions
```

## 8. 命令参考

### `infiniti-agent init`

交互式配置 LLM，并写入全局 `~/.infiniti-agent/config.json`。

### `infiniti-agent migrate`

把全局配置、记忆和 skills 复制到当前目录 `.infiniti-agent/`，使当前目录成为独立 agent layout。

### `infiniti-agent upgrade [--global]`

升级 `config.json` 到当前格式。默认升级本地和全局配置；`--global` 只升级全局配置。

### `infiniti-agent chat`

进入交互式 TUI。无参数运行 `infiniti-agent` 时默认进入 `chat`。

### `infiniti-agent cli <prompt...>`

非交互执行一轮，将结果输出到 stdout。启动前如果当前目录绑定了 LinkYun Agent，会先做启动同步，结束后做关闭同步。

### `infiniti-agent live`

启动 LiveUI 会话：本地 WebSocket、可选 Electron 窗口，以及 TUI。

选项：

| 选项 | 行为 |
| --- | --- |
| `-p, --port <port>` | WebSocket 端口，覆盖 `config.liveUi.port` |
| `--auto` | 使用自动 VAD 语音输入；默认是按住空格录音，松开发送识别 |
| `--headless` | 只启动 Live WebSocket，不打开 Electron 窗口 |
| `--headness` | `--headless` 的兼容别名 |
| `--voice-possible <n>` | 仅 headless 模式生效，范围 `0` 到 `1`，默认 `0.3` |
| `--zoom <n>` | 人物显示缩放，范围 `0.4` 到 `1.5` |

相关环境变量：

| 环境变量 | 行为 |
| --- | --- |
| `INFINITI_LIVEUI_PORT` | 未传 `--port` 且配置无端口时作为端口来源 |
| `INFINITI_LIVEUI_DEBUG_WINDOW=1` | Electron 使用带标题栏的调试窗口 |
| `INFINITI_LIVEUI_DEVTOOLS=1` | 启动时打开 DevTools |
| `INFINITI_AGENT_DEBUG=1` | live 模式下也会启用 DevTools |

### `infiniti-agent test_camera`

通过 CLI 后端拍一张 JPEG，并输出日志路径。

选项：

| 选项 | 默认 | 行为 |
| --- | --- | --- |
| `--output <path>` | `/tmp/infiniti-agent-camera-<time>.jpg` | 图片输出路径 |
| `--log <path>` | `/tmp/infiniti-agent-camera-<time>.log` | 日志输出路径 |
| `--timeout-ms <n>` | `20000` | 测试总超时 |

别名：

```bash
infiniti-agent test-camera
```

### `infiniti-agent test_asr`

用 `ffmpeg` 采集麦克风音频，按 RMS 和静音时间切段，调用配置中的 ASR。stdout 输出识别文本。需要已配置 `asr`，并安装 `ffmpeg`。

选项：

| 选项 | 默认 | 行为 |
| --- | --- | --- |
| `--rms <n>` | `liveUi.voiceMicSpeechRmsThreshold` 的默认值 | 语音判定 RMS 阈值 |
| `--silence-ms <n>` | `1500` | 静音判停时长 |
| `--min-chunk-ms <n>` | `250` | 最短送识别片段 |

### `infiniti-agent export <file>`

导出当前目录的 agent layout 为 `.agent` zip 文件。

会包含：

- `.infiniti-agent/`
- `SOUL.md`
- `AGENTS.md`
- `AGENT.md`
- `INFINITI.md`
- `CLAUDE.md`
- `.claude/CLAUDE.md`

会跳过：

- `.env.local`
- `.infiniti-agent/inbox/assets/`
- `.infiniti-agent/backups/`
- `.infiniti-agent/tmp/`
- `.infiniti-agent/*.log`
- 其他 `.agent` 文件

### `infiniti-agent import <file> [-f, --force]`

从 `.agent` zip 文件导入 agent layout。当前目录已有 layout 时会询问是否覆盖；`--force` 直接覆盖。导入会做路径安全检查，拒绝不安全的归档路径。

### `infiniti-agent add_llm`

交互式添加 LLM profile 到当前项目 `.infiniti-agent/config.json`。

选项：

| 选项 | 行为 |
| --- | --- |
| `--profile <name>` | 指定 profile 名，默认 `main` |
| `--provider <name>` | 跳过厂商选择；支持 `openai`、`anthropic`、`gemini`、`openrouter` |

命令会尝试拉取模型列表；失败时可以手动输入 model id。

### `infiniti-agent select_llm`

切换当前项目默认 LLM profile。

选项：

| 选项 | 行为 |
| --- | --- |
| `--name <profile>` | 直接指定 profile 名，跳过交互 |

### `infiniti-agent sync`

登录 LinkYun，选择 Agent，同步 `SOUL.md`、参考素材和 `.agent` 归档。

选项：

| 选项 | 行为 |
| --- | --- |
| `--api-base <url>` | API 根地址，默认 `https://api.linkyun.co` |
| `--workspace <code>` | 指定 `X-Workspace-Code` |
| `--agent <code>` | 指定 LinkYun Agent code，并写入 `.env.local` |
| `--login` | 强制重新登录并刷新 `.env.local` |
| `--pull` | 强制以服务器最新 `.agent` 为准，下载并覆盖当前 layout |
| `--push` | 强制以本地 layout 为准，导出并上传 `.agent` |
| `--with-version` | 列出服务器 `.agent` 版本，选择指定版本下载并覆盖 |

不能同时使用 `--pull` 和 `--push`，也不能同时使用 `--with-version` 和 `--push`。

### `infiniti-agent link`

从当前目录 `SOUL.md` 提取邮件配置，生成 `mail-poller.sh`。

`SOUL.md` 必须包含：

- Agent 地址，形如 `xxx@xxx.amp.linkyun.co`
- Agent ID，UUID 格式
- API key，形如 `amk_xxx`

生成后可运行：

```bash
./mail-poller.sh
nohup ./mail-poller.sh &
./mail-poller.sh --once
```

脚本日志写入当前目录 `mail-poller.log`，发现未读邮件时会调用 `infiniti-agent cli`。

### `infiniti-agent skill install <source>`

安装 Skill 到当前项目 `.infiniti-agent/skills/`。

`<source>` 支持：

- `owner/repo`
- `https://...` 或 `git@...` Git URL
- 本地路径

### `infiniti-agent skill add <source>`

`skill install` 的别名。

### `infiniti-agent skill list`

列出当前项目已安装的 Skills。

### `infiniti-agent generate_avatar --agent <code>`

根据 `.infiniti-agent/ref/<agent>/` 下的头像和设定稿，通过图像 API 生成 real2d 表情 PNG。

选项：

| 选项 | 行为 |
| --- | --- |
| `--agent <code>` | 必填，LinkYun Agent code |
| `--out <dir>` | 输出目录，默认 `live2d-models/<agent>/expression` |
| `--skip-half-body` | 跳过新半身像，复用输出目录已有 `half_body.png` |
| `--no-transparentize` | 跳过背景透明化 |

默认会生成 `half_body.png`、`exp_01.png` 到 `exp_08.png`，以及 `expressions.json`。

### `infiniti-agent set_live_agent <code>`

把 LiveUI 形象设为指定 agent。命令会写入：

```json
{
  "liveUi": {
    "spriteExpressions": {
      "dir": "./live2d-models/<code>/expression"
    }
  }
}
```

如果目录不存在，会提示先运行 `sync + generate_avatar` 或自行创建目录。

### worker 命令

以下命令由后台任务使用，通常不需要手动调用：

```bash
infiniti-agent snap-worker <job.json>
infiniti-agent video-worker <job.json>
infiniti-agent avatargen-worker <job.json>
```

## 9. 交互式 TUI 命令

输入 `/` 可以补全 slash commands 和工具。方向键、Tab 可用于补全选择。

| 命令 | 行为 |
| --- | --- |
| `/exit`、`/quit` | 保存会话并退出 |
| `/clear`、`/new` | 归档当前会话并清空当前会话 |
| `/reload`、`/reload-skills` | 重新加载配置和 skills |
| `/help` | 显示帮助文本 |
| `/config` | Live 模式下打开配置面板 |
| `/debug` | Live 模式下切换 LiveUI debug overlay |
| `/permission` | 显示当前权限模式 |
| `/memory` | 显示记忆系统位置提示 |
| `/inbox` | 显示最近未读收件箱消息 |
| `/inbox --all` | 显示最近收件箱消息，不限未读 |
| `/last_email` | 打开或显示最近一封邮件 |
| `/undo` | 撤销本会话内最近一次成功的 `write_file` 或 `str_replace` |
| `/roll` | 回滚 1 层 LLM 输出 |
| `/roll 2` | 回滚 2 层 LLM 输出 |
| `/compact` | 后台压缩较早会话 |
| `/compact <instructions>` | 带自定义说明压缩较早会话 |
| `/schedule`、`/schedule list` | 列出计划任务 |
| `/schedule add <自然语言>` | 添加计划任务 |
| `/schedule remove <id>`、`/schedule rm <id>` | 按 ID 或 ID 前缀删除计划任务 |
| `/schedule clear` | 清理未来不再执行的计划任务 |
| `/dream`、`/dream diary` | 显示最近 dream diary |
| `/dream context` | 显示 Dream Context |
| `/dream run` | 手动运行完整 dream |
| `/dream run light` | 手动运行 light dream |
| `/showmemagic` | LiveUI 中打开官方 H5/SVG/CSS 测试页 |
| `/showmemagci`、`/show-me-magic`、`/showmethemagic`、`/show-me-the-magic` | `/showmemagic` 的兼容别名 |
| `/speak <text>` | Live 模式下只朗读文本，不写入会话 |
| `/snap <prompt>` | 后台生成合照或写实照片，完成后写入收件箱 |
| `/avatargen <prompt>` | 后台生成 real2d 表情集，完成后写入收件箱 |
| `/video <prompt>` | 后台生成 Seedance 视频，完成后写入收件箱 |
| `/seedance <prompt>` | `/video` 的别名 |
| `/sendImage <path> [caption]` | 把本地图片投递给已连接的 LiveUI 客户端 |
| `/sendVideo <path> [caption]` | 把本地视频投递给已连接的 LiveUI 客户端 |
| `/sendFile <path> [caption]` | 把本地文件投递给已连接的 LiveUI 客户端 |

`/sendImage`、`/sendVideo`、`/sendFile` 支持带空格路径的引号形式：

```text
/sendFile "/Users/me/path with space/report.pdf" 报告
```

## 10. 计划任务

计划任务存储在 `.infiniti-agent/schedules.json`。

支持的类型：

- `once`
- `interval`
- `daily`

TUI 中可以用自然语言添加：

```text
/schedule add 明天9点提醒我检查日志
/schedule add 每分钟检查消息
/schedule add 每5分钟检查 inbox
/schedule add 每天早上8点总结今日计划
```

模型也可以通过内置 `schedule` 工具创建、列出、删除或清理任务。

## 11. 权限和内置工具

默认情况下，改文件、bash、HTTP 等工具会经过安全评估。终端交互中通常会出现确认：

- `Y`：允许本次
- `A`：本次会话始终允许该工具
- `N`：拒绝

跳过所有工具确认：

```bash
infiniti-agent --dangerously-skip-permissions
```

内置工具包括：

- H5 applet：`request_h5_applet`、`launch_h5_applet`、`list_h5_applets`、`delete_h5_applet_cache`、`create_h5_applet`、`update_h5_applet`、`destroy_h5_applet`
- 文件和仓库：`read_file`、`list_directory`、`glob_files`、`grep_files`、`write_file`、`str_replace`
- 外部访问和 shell：`http_request`、`bash`
- 记忆和画像：`update_memory`、`memory`、`user_profile`
- 检索和知识：`search_sessions`、`knowledge_graph`
- 计划和扩展：`schedule`、`manage_skill`
- 媒体：`snap_photo`、`avatargen_real2d`、`seedance_video`、`send_image`、`send_video`、`send_file`

`http_request` 只允许 `http` 和 `https` URL，并会阻止本地、私网和 link-local 地址。`bash` 的 cwd 会限制在 workspace 内。

## 12. LLM 配置

支持的 LLM provider：

- `anthropic`
- `openai`
- `gemini`
- `minimax`
- `openrouter`

推荐使用 profiles：

```json
{
  "version": 1,
  "llm": {
    "provider": "anthropic",
    "baseUrl": "https://api.anthropic.com",
    "model": "claude-sonnet-4-20250514",
    "apiKey": "YOUR_API_KEY",
    "default": "main",
    "metaAgentProfile": "gate",
    "subconsciousProfile": "dream",
    "profiles": {
      "main": {
        "provider": "anthropic",
        "baseUrl": "https://api.anthropic.com",
        "model": "claude-sonnet-4-20250514",
        "apiKey": "YOUR_API_KEY"
      },
      "gate": {
        "provider": "gemini",
        "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
        "model": "gemini-2.0-flash",
        "apiKey": "YOUR_GEMINI_API_KEY"
      }
    }
  }
}
```

特殊 profile 字段：

| 字段 | 用途 |
| --- | --- |
| `llm.default` | 主对话默认 profile |
| `llm.metaAgentProfile` | 工具安全评估 profile，未配置时默认尝试 `gate` |
| `llm.subconsciousProfile` | subconscious-agent profile，未配置时使用主 LLM |
| `llm.callProfile` | 通话模式主对话 profile |
| `llm.callAugmenterProfile` | 通话模式后台 augmenter profile |
| `disableTools` | 不向 API 发送 tools，适用于不支持 function calling 的模型 |

代码中的 provider 默认值：

| Provider | 默认 baseUrl | 默认 model |
| --- | --- | --- |
| `anthropic` | `https://api.anthropic.com` | `claude-sonnet-4-20250514` |
| `openai` | `https://api.openai.com/v1` | `gpt-4.1` |
| `gemini` | `https://generativelanguage.googleapis.com/v1beta` | `gemini-2.0-flash` |
| `minimax` | `https://api.minimax.io/v1` | `MiniMax-M2.7` |
| `openrouter` | `https://openrouter.ai/api/v1` | `openai/gpt-4o` |

本地 OpenAI-compatible 示例：

```json
{
  "version": 1,
  "llm": {
    "provider": "openai",
    "baseUrl": "http://127.0.0.1:11434/v1",
    "model": "qwen2.5:7b",
    "apiKey": "ollama",
    "disableTools": true
  }
}
```

## 13. Thinking 和 compaction

Thinking 配置：

```json
{
  "thinking": {
    "mode": "adaptive",
    "budgetTokens": 10000
  }
}
```

`mode` 支持：

- `adaptive`
- `enabled`
- `disabled`

`--disable-thinking` 会临时把 mode 设为 `disabled`。

Compaction 配置：

```json
{
  "compaction": {
    "autoThresholdTokens": 30000,
    "minTailMessages": 8,
    "maxToolSnippetChars": 4000,
    "preCompactHook": "node scripts/before-compact.js"
  }
}
```

`preCompactHook` 是压缩前执行的命令。Windows 下建议使用明确的可执行命令，例如 `node script.js`。

## 14. MCP 配置

MCP server 配置在 `mcp.servers` 下：

```json
{
  "mcp": {
    "servers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "."],
        "env": {
          "EXAMPLE_TOKEN": "YOUR_TOKEN"
        },
        "cwd": "."
      }
    }
  }
}
```

字段：

| 字段 | 用途 |
| --- | --- |
| `command` | 启动 MCP server 的命令 |
| `args` | 命令参数 |
| `env` | 追加到子进程的环境变量 |
| `cwd` | server 工作目录 |

工具暴露名会按 `<server>_<tool>` 的形式清理和拼接，避免不同 server 的工具重名。

## 15. LiveUI

启动：

```bash
infiniti-agent live
```

端口优先级：

1. `--port`
2. `config.liveUi.port`
3. `INFINITI_LIVEUI_PORT`
4. 默认 `8080`

LiveUI 配置示例：

```json
{
  "liveUi": {
    "port": 8080,
    "subconsciousHeartbeatMs": 60000,
    "figureZoom": 1,
    "renderer": "live2d",
    "live2dModelsDir": "./live2d-models",
    "live2dModelDict": "./model_dict.json",
    "live2dModelName": "mao_pro",
    "voiceMicSpeechRmsThreshold": 0.0195,
    "voiceMicSilenceEndMs": 1500,
    "voiceMicSuppressInterruptDuringTts": true
  }
}
```

重要范围：

| 字段 | 范围 |
| --- | --- |
| `port` | `1` 到 `65535` |
| `subconsciousHeartbeatMs` | `5000` 到 `3600000` |
| `figureZoom` | `0.4` 到 `1.5` |
| `voiceMicSpeechRmsThreshold` | 大于 `0` 且不超过 `0.35` |
| `voiceMicSilenceEndMs` | `200` 到 `12000` |

Renderer：

| 值 | 行为 |
| --- | --- |
| `live2d` | 使用 `.model3.json` / `model_dict.json` 解析到的 Live2D 模型 |
| `sprite` | 使用 `spriteExpressions.dir` 下的 PNG 表情，不加载 Live2D 模型 |
| `real2d` | 使用 `spriteExpressions.dir` 下的 real2d PNG，并通过前端实时变形与口型驱动 |

模型解析优先级：

1. `liveUi.live2dModel3Json`
2. `liveUi.live2dModelName` + `liveUi.live2dModelDict`
3. avatar fallback

如果 `spriteExpressions.dir` 可用，默认 renderer 会从 `live2d` 切到 `sprite`。如果设置 `renderer=real2d` 但没有可用 `spriteExpressions.dir`，会回退到 Live2D 或占位，并输出警告。

Sprite/real2d 示例：

```json
{
  "liveUi": {
    "renderer": "real2d",
    "spriteExpressions": {
      "dir": "./live2d-models/jess/expression"
    }
  }
}
```

语音输入：

- 默认模式：点击麦克风后，按住空格录音，松开发送 ASR。
- `infiniti-agent live --auto`：启用自动 VAD。
- `liveUi.asrMode` 也支持 `manual` 或 `auto`。

窗口调试：

```bash
INFINITI_LIVEUI_DEBUG_WINDOW=1 infiniti-agent live
INFINITI_LIVEUI_DEVTOOLS=1 infiniti-agent live
infiniti-agent live --debug
```

## 16. TTS 配置

支持的 TTS provider：

- `minimax`
- `moss_tts_nano`
- `voxcpm`
- `whisper`
- `mimo`

VoxCPM 示例：

```json
{
  "tts": {
    "provider": "voxcpm",
    "baseUrl": "http://127.0.0.1:8810",
    "controlInstruction": "年轻女性，温柔自然",
    "amplitudeNormalize": "rms"
  }
}
```

MOSS-TTS-Nano 示例：

```json
{
  "tts": {
    "provider": "moss_tts_nano",
    "baseUrl": "http://127.0.0.1:18083",
    "demoId": "demo-voice",
    "timeoutMs": 120000
  }
}
```

MiniMax 示例：

```json
{
  "tts": {
    "provider": "minimax",
    "apiKey": "YOUR_MINIMAX_API_KEY",
    "groupId": "YOUR_GROUP_ID",
    "model": "speech-02-turbo",
    "voiceId": "female-shaonv"
  }
}
```

Whisper-style TTS 示例：

```json
{
  "tts": {
    "provider": "whisper",
    "apiKey": "YOUR_API_KEY",
    "baseUrl": "https://api.openai.com/v1",
    "model": "gpt-4o-mini-tts",
    "voiceId": "alloy"
  }
}
```

MiMo 示例：

```json
{
  "tts": {
    "provider": "mimo",
    "apiKey": "YOUR_MIMO_API_KEY",
    "baseUrl": "https://token-plan-cn.xiaomimimo.com/v1",
    "model": "mimo-v2.5-tts",
    "voiceId": "default"
  }
}
```

## 17. ASR 配置

支持的 ASR provider：

- `whisper`
- `sherpa_onnx`

Whisper ASR：

```json
{
  "asr": {
    "provider": "whisper",
    "apiKey": "YOUR_API_KEY",
    "baseUrl": "https://api.openai.com",
    "model": "whisper-large-v3-turbo",
    "lang": "zh"
  }
}
```

代码会调用：

```text
<baseUrl>/v1/audio/transcriptions
```

Sherpa ONNX ASR：

```json
{
  "asr": {
    "provider": "sherpa_onnx",
    "model": "./models/sensevoice.onnx",
    "tokens": "./models/tokens.txt",
    "lang": "auto",
    "numThreads": 4
  }
}
```

相对路径按当前项目根目录解析。

## 18. 图片、AvatarGen、Snap 和 Seedance

统一图片 profile：

```json
{
  "image": {
    "default": "openrouter",
    "avatarGenProfile": "openrouter",
    "snapProfile": "openrouter",
    "profiles": {
      "openrouter": {
        "provider": "openrouter",
        "baseUrl": "https://openrouter.ai/api/v1",
        "apiKey": "YOUR_OPENROUTER_API_KEY",
        "model": "google/gemini-3-pro-image-preview",
        "aspectRatio": "4:3",
        "timeoutMs": 120000
      }
    }
  }
}
```

支持的 image provider 值：

- `openai`
- `openrouter`
- `gemini`
- `gpt-image-2`
- `nano-banana`

实际解析时：

- `gpt-image-2` 和 `chatgpt-image` 会走 OpenAI。
- `nano-banana` 默认走 OpenRouter。
- 图像 API key 可以来自 image profile、legacy `avatarGen`/`snap` 字段、环境变量，或同 provider 的 LLM profile。

相关环境变量：

| Provider | 环境变量 |
| --- | --- |
| OpenAI | `INFINITI_OPENAI_IMAGE_API_KEY` 或 `OPENAI_API_KEY` |
| Gemini | `INFINITI_GEMINI_IMAGE_API_KEY`、`GEMINI_API_KEY` 或 `GOOGLE_API_KEY` |
| OpenRouter | `INFINITI_OPENROUTER_API_KEY` 或 `OPENROUTER_API_KEY` |
| AvatarGen model override | `INFINITI_AVATAR_GEN_MODEL` |

Snap legacy 配置：

```json
{
  "snap": {
    "provider": "openrouter",
    "baseUrl": "https://openrouter.ai/api/v1",
    "apiKey": "YOUR_OPENROUTER_API_KEY",
    "model": "google/gemini-3-pro-image-preview",
    "aspectRatio": "4:3",
    "timeoutMs": 120000
  }
}
```

AvatarGen legacy 配置：

```json
{
  "avatarGen": {
    "provider": "gemini",
    "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
    "apiKey": "YOUR_GEMINI_API_KEY",
    "model": "gemini-3.1-flash-image-preview",
    "quality": "high",
    "transparentBackground": true
  }
}
```

Seedance 配置：

```json
{
  "seedance": {
    "provider": "volcengine",
    "baseUrl": "https://ark.cn-beijing.volces.com",
    "apiKey": "YOUR_ARK_API_KEY",
    "model": "doubao-seedance-2-0-260128",
    "ratio": "16:9",
    "duration": 5,
    "resolution": "720p",
    "generateAudio": true,
    "watermark": false,
    "pollIntervalMs": 15000,
    "timeoutMs": 900000
  }
}
```

Seedance 默认：

- `baseUrl`: `https://ark.cn-beijing.volces.com`
- `model`: `doubao-seedance-2-0-260128`
- `ratio`: `16:9`
- `duration`: `5`
- `pollIntervalMs`: `15000`
- `timeoutMs`: `900000`

`baseUrl` 可以是服务根 URL，也可以直接指向 `/api/v3/contents/generations/tasks`。

媒体 slash command：

```text
/snap 在咖啡馆自拍，暖色灯光
/avatargen 城市猎人风格
/video 夕阳海边的电影感航拍，慢速推进
/seedance 夕阳海边的电影感航拍，慢速推进
```

这些命令会创建后台 job。完成后会写入 `.infiniti-agent/inbox/`，生成的文件通常放在 `.infiniti-agent/inbox/assets/`。

## 19. Live2D 和 real2d 素材

Live2D 目录应与 Open-LLM-VTuber 的 `live2d-models` 布局一致。例如：

```text
live2d-models/
  mao_pro/
    runtime/
      mao_pro.model3.json
```

`model_dict.json` 可从 `model_dict.example.json` 复制。也可以运行：

```bash
npm run setup:liveui -- /path/to/your-project
```

`generate_avatar` 生成的 sprite 表情目录示例：

```text
live2d-models/jess/expression/
  half_body.png
  exp_01.png
  exp_02.png
  exp_03.png
  exp_04.png
  exp_05.png
  exp_06.png
  exp_07.png
  exp_08.png
  expressions.json
```

`/avatargen` 后台 Real2D 表情集生成的是另一套命名：`exp01.png` 到 `exp06.png`，以及可选说话口型 `exp_open.png`。

设置当前 Live agent：

```bash
infiniti-agent set_live_agent jess
infiniti-agent live
```

## 20. LinkYun 同步

手动同步：

```bash
infiniti-agent sync
```

第一次会登录 LinkYun，选择 Agent，并写入 `.env.local`。之后启动 `infiniti-agent`、`infiniti-agent chat`、`infiniti-agent live`、`infiniti-agent cli` 时，如果 `.env.local` 中有可用绑定，会自动做启动同步。

`.env.local` 管理字段：

- `LINKYUN_API_BASE`
- `LINKYUN_API_KEY`
- `LINKYUN_WORKSPACE_CODE`
- `LINKYUN_AGENT_CODE`

启动同步行为：

- 只做 pull-only。
- 如果服务器没有 `.agent` 归档，不会启动时自动上传。
- 如果本地 session 更新，不会启动时自动上传旧状态。
- 如果服务器 `.agent` 更新，会下载并导入。

退出同步行为：

- 正常退出时会自动 push 当前 `.agent`。

跳过自动同步：

```bash
INFINITI_AGENT_SKIP_STARTUP_SYNC=1 infiniti-agent
```

覆盖导入前会备份关键文件到 `.infiniti-agent/backups/sync/`，最多保留最近 5 份。关键文件包括：

- `SOUL.md`
- `.infiniti-agent/session.json`
- `.infiniti-agent/memory.json`
- `.infiniti-agent/subconscious.json`
- `.infiniti-agent/schedules.json`
- `.infiniti-agent/sessions.db`
- `.infiniti-agent/knowledge.db`

## 21. Skills

Skills 存储在当前项目：

```text
.infiniti-agent/skills/
```

从 GitHub 安装：

```bash
infiniti-agent skill add owner/repo
```

从 Git URL 安装：

```bash
infiniti-agent skill install https://github.com/owner/repo.git
```

从本地路径安装：

```bash
infiniti-agent skill add ./my-skill
```

列出：

```bash
infiniti-agent skill list
```

Git 安装会执行 shallow clone。如果仓库根目录有 `SKILL.md`，会安装整个仓库；否则会安装仓库中的各个子目录。目标目录名来自 repo 名或本地路径 basename。

## 22. Agent 归档迁移

导出：

```bash
infiniti-agent export jess.agent
```

导入：

```bash
infiniti-agent import jess.agent
```

已有 layout 时直接覆盖：

```bash
infiniti-agent import jess.agent --force
```

`.agent` 是 zip 格式，内部带 `.infiniti-agent-export.json` manifest。

## 23. 邮件轮询

`infiniti-agent link` 会读取当前目录 `SOUL.md`，生成 `mail-poller.sh`。

前提：

```text
agent@example.amp.linkyun.co
00000000-0000-0000-0000-000000000000
amk_xxx
```

运行：

```bash
infiniti-agent link
./mail-poller.sh
```

脚本每 60 秒检查一次 Mail Broker 收件箱。发现未读邮件时，会调用：

```bash
infiniti-agent cli "<prompt>"
```

日志：

```text
mail-poller.log
```

## 24. 外部本地服务

本仓库定位为 Agent 编排层。大模型和 TTS 的重服务建议在兄弟仓库独立运行：

- `../infiniti-llm-service`
- `../infiniti-tts-service`

主仓只配置 service URL。

LLM 本地服务示例：

```json
{
  "llm": {
    "provider": "openai",
    "baseUrl": "http://127.0.0.1:11434/v1",
    "model": "qwen2.5:7b",
    "apiKey": "ollama",
    "disableTools": true
  }
}
```

TTS 本地服务示例：

```json
{
  "tts": {
    "provider": "voxcpm",
    "baseUrl": "http://127.0.0.1:8810",
    "controlInstruction": "年轻女性，温柔自然",
    "amplitudeNormalize": "rms"
  }
}
```

常见本地 URL：

| 服务 | URL |
| --- | --- |
| Ollama OpenAI-compatible | `http://127.0.0.1:11434/v1` |
| vLLM OpenAI-compatible | `http://127.0.0.1:8000/v1` |
| VoxCPM2 TTS | `http://127.0.0.1:8810` |
| MOSS-TTS-Nano | `http://127.0.0.1:18083` |

## 25. 开发和测试

开发启动：

```bash
npm run dev
```

CLI 开发启动：

```bash
npm run dev -- cli 你好
```

构建：

```bash
npm run build
```

测试：

```bash
npm run test
```

发布前构建：

```bash
npm run build
npm publish
```

## 26. 排查问题

### 尚未配置

报错：

```text
尚未配置。请先运行: infiniti-agent init 或 infiniti-agent migrate
```

处理：

```bash
infiniti-agent init
infiniti-agent migrate
```

### config.json 不是合法 JSON

检查：

```bash
node -e "JSON.parse(require('fs').readFileSync('.infiniti-agent/config.json','utf8'))"
```

### provider 不合法

`llm.provider` 必须是：

- `anthropic`
- `openai`
- `gemini`
- `minimax`
- `openrouter`

### LiveUI 没有窗口

先确认已经构建：

```bash
npm run build
```

如果全局安装后 Electron 窗口没有出现，在包安装目录重新安装 optional dependency，或重装包。也可以先用 headless 验证服务：

```bash
infiniti-agent live --headless
```

### real2d 回退到 Live2D 或占位

检查：

```json
{
  "liveUi": {
    "renderer": "real2d",
    "spriteExpressions": {
      "dir": "./live2d-models/jess/expression"
    }
  }
}
```

并确认目录存在且包含表情 PNG。

### test_asr 无法启动

确认：

- 已安装 `ffmpeg`
- 系统已授权麦克风权限
- `config.json` 配置了 `asr`

### 工具卡住或行为不清楚

启用调试：

```bash
INFINITI_AGENT_DEBUG=1 infiniti-agent
```

或：

```bash
infiniti-agent --debug
```

### LinkYun 自动同步影响启动

临时跳过：

```bash
INFINITI_AGENT_SKIP_STARTUP_SYNC=1 infiniti-agent
```

### `/sendImage` 投递失败

检查：

- 当前是否用 `infiniti-agent live` 启动。
- 是否有 LiveUI 客户端连接。
- 路径是否存在且是文件。
- 相对路径是否相对当前项目根。

## 27. 安全注意事项

- 不要把真实 API key 写进 README、截图、issue 或公开仓库。
- `.env.local` 保存 LinkYun 登录信息，默认不应提交。
- `.agent` 导出会跳过 `.env.local` 和本地 inbox assets，但仍可能包含会话、记忆和 prompt 文件，分享前先检查内容。
- `--dangerously-skip-permissions` 会跳过工具确认，只应在可信环境短时使用。
- `http_request` 默认阻止访问本地、私网和 link-local 地址，避免误碰内网服务。

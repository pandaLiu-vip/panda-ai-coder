# Panda Ai Coder

阅读其他语言:
[中文](README.zh.md) | [English](README.en.md)

<p>作者：刘晨辉</p>
<p align="left"><img src="resources/author.png" width="100" alt="author"></p>
<p>Tel/微信：15799845845</p>

`Panda Ai Coder` 是一个 VS Code 侧边栏扩展，用来把本地 Ollama 或 OpenAI 兼容模型服务接入到编辑器里，围绕当前工作区进行多线程对话，并在模型返回标准文件块时直接写回项目文件。

![设置操作](resources/readme.gif)

## 当前实现对应的能力

- 在活动栏提供独立的 `Panda Ai Coder` 视图
- 支持多任务线程，保留最近任务、历史搜索、线程切换、新建和删除
- 支持本地 Ollama，也支持 OpenAI 兼容的 `/v1` 接口
- 可在侧边栏中保存服务地址、授权码、远程模型名和最大 token 数
- 支持流式输出，生成中会持续把增量内容推送到聊天区
- 会自动读取当前工作区文件树，并按任务内容、活动编辑器、附件路径挑选少量相关文件片段作为上下文
- 支持 `AGENTS.md` 分层指令注入，会沿工作区根目录、活动文件目录、附件目录向上发现并注入相关规则
- 支持本地 `skills` 目录发现与按任务自动加载 `SKILL.md` 内容，未命中的技能也会作为目录索引提示给模型
- 支持基于 `mcp.json` 的 MCP stdio server 接入，模型可通过 `<tool_call ...>` 发起工具调用并继续完成任务
- 当模型返回 `<file path="...">...</file>` 格式内容时，会先生成 diff 变更集，并支持审批、拒绝、回滚；“执行建议”模式下可直接自动应用
- 如果模型接口返回 usage 信息，会在界面里展示 token 用量

## 这不是它当前具备的能力

下面这些能力在当前代码里还没有实现，因此不应写进文档或对外承诺：

- `@file` / `@folder` 显式提及解析
- 命令执行、测试运行、Lint 回灌
- Git 集成、Web Search、子代理
- VS Code `Settings` 面板中的 `contributes.configuration` 配置项

当前版本的服务配置入口在扩展自己的侧边栏设置面板里，而不是 VS Code 全局设置页。

## 工作方式

扩展激活后会注册两个命令：

- `panda-ai-coder.openSidebar`
- `panda-ai-coder.newThread`

主流程如下：

1. 在活动栏打开 `Panda Ai Coder`
2. 在设置面板中填写模型服务地址
3. 如果是远程 OpenAI 兼容服务，可额外填写授权码、远程模型名、最大 token 数
4. 刷新模型列表并选择模型
5. 在当前线程中输入任务描述并发送
6. 扩展根据当前线程和工作区生成上下文，请求模型接口
7. 发送前会额外拼装三层扩展上下文：
   - `AGENTS.md` 指令层
   - `skills` 技能层
   - `MCP` 工具层
8. 如果模型返回普通文本，则只展示回复
9. 如果模型返回 `<tool_call ...>`，扩展会调用对应 MCP 工具并把结果回灌给模型继续完成任务
10. 如果模型返回 `<file path="相对路径">` 文件块，则扩展会生成待处理改动，并按当前模式执行审批或写入

## 模型服务支持

### 1. 本地 Ollama

默认地址是：

```text
http://127.0.0.1:11434/api
```

扩展会优先调用接口拉取模型列表；如果本地 Ollama 接口不可用，非 OpenAI 模式下还会尝试执行：

```bash
ollama list
```

### 2. OpenAI 兼容接口

如果服务地址路径中包含 `/v1`，扩展会按 OpenAI 兼容接口处理，请求：

- `POST /chat/completions`
- `GET /models`

授权码会自动补成 `Bearer <token>` 形式并保存在 VS Code Secret Storage 中。

## 文件写入协议

当前版本依赖模型按下面的格式返回文件内容：

```xml
<file path="src/example.ts">
export const message = 'hello';
</file>
```

注意事项：

- `path` 必须是相对工作区路径
- 不能是绝对路径
- 不能越出工作区目录
- 不适合二进制文件

扩展会剥离这些 `<file>` 块后再把剩余说明文本展示在聊天区，并附上已写入文件列表。

## AGENTS.md 能力

扩展会自动在这些位置查找 `AGENTS.md`：

- 工作区根目录
- 当前活动文件所在目录及其祖先目录
- 当前线程附件对应目录及其祖先目录

命中的 `AGENTS.md` 会按作用域从外到内注入到 system prompt，适合放：

- 项目级约束
- 子目录编码规范
- 特定模块注意事项

## skills 能力

扩展会自动扫描这些目录中的 `SKILL.md`：

- `<workspace>/.codex/skills`
- `<workspace>/.panda/skills`
- `<workspace>/skills`
- `~/.codex/skills`
- `~/.panda/skills`

它会根据线程标题和最近一条用户消息，自动挑选最相关的技能内容注入；未直接命中的技能会以简短索引形式提供给模型，帮助它判断当前工作区还提供了哪些能力。

## MCP 能力

当前版本支持通过工作区内的 `mcp.json` 或以下路径之一接入 stdio MCP server：

- `.vscode/mcp.json`
- `.cursor/mcp.json`
- `.codex/mcp.json`
- `.panda/mcp.json`
- `.mcp.json`
- `mcp.json`

最小示例：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "node",
      "args": ["./scripts/mcp-filesystem.js"],
      "cwd": "${workspaceFolder}"
    }
  }
}
```

支持的模板变量：

- `${workspaceFolder}` / `${workspaceRoot}`
- `${configDir}`
- `${userHome}`
- `${env:NAME}`

模型需要调用工具时，会输出：

```xml
<tool_call server="filesystem" name="read_file">
{"path":"src/extension.ts"}
</tool_call>
```

扩展会执行该调用，并把工具结果继续回灌给模型完成后续回答。

## 已知限制

- 只支持 stdio 类型的 MCP server，不支持 SSE / HTTP transport
- MCP 调用要求模型输出约定的 `<tool_call>` 标签，暂未适配 OpenAI 原生 function calling 协议
- skills 当前只加载 `SKILL.md` 正文，尚未递归执行技能目录中的脚本或额外资源路由
- 只处理第一个工作区目录
- 自动附加的代码上下文有大小和数量上限
- 一些内部命名仍保留 `ollamaCoder` 前缀，这是历史实现遗留，不影响功能

## 运行要求

- VS Code `^1.109.3`
- Node.js 与 npm
- 本地 Ollama，或任意可访问的 OpenAI 兼容模型服务

## 许可证

本项目采用 [MIT License](LICENSE)。

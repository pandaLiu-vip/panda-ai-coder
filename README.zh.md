# Panda Ai Coder

阅读其他语言:
[中文](README.zh.md) | [English](README.en.md)

<p>作者：刘晨辉</p>
<p align="left"><img src="resources/author.png" width="100" alt="author"></p>
<p>Tel/微信：15799845845</p>

`Panda Ai Coder` 是一个 VS Code 侧边栏扩展，用来把本地 Ollama 或 OpenAI 兼容模型服务接入到编辑器里，围绕当前工作区进行多线程对话，并在模型返回标准文件块时直接写回项目文件。

## 当前实现对应的能力

- 在活动栏提供独立的 `Panda Ai Coder` 视图
- 支持多任务线程，保留最近任务、历史搜索、线程切换、新建和删除
- 支持本地 Ollama，也支持 OpenAI 兼容的 `/v1` 接口
- 可在侧边栏中保存服务地址、授权码、远程模型名和最大 token 数
- 会自动读取当前工作区文件树，并按任务内容、活动编辑器、附件路径挑选少量相关文件片段作为上下文
- 当模型返回 `<file path="...">...</file>` 格式内容时，会把文件直接写入当前工作区
- 如果模型接口返回 usage 信息，会在界面里展示 token 用量

## 这不是它当前具备的能力

下面这些能力在当前代码里还没有实现，因此不应写进文档或对外承诺：

- 流式输出
- diff 审批 / 回滚工作流
- `@file` / `@folder` 显式提及解析
- 命令执行、测试运行、Lint 回灌
- Git 集成、MCP、Web Search、子代理
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
7. 如果模型返回普通文本，则只展示回复
8. 如果模型返回 `<file path="相对路径">` 文件块，则把对应内容写入工作区

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

## 项目结构

```text
src/
  extension.ts              扩展激活入口
  constants/ids.ts          命令与视图 ID
  ui/sidebarProvider.ts     Webview 后端、模型请求、上下文构建、文件写入
media/
  main.js                   Webview 前端逻辑
  main.css                  Webview 样式
resources/
  *.png / *.svg             图标资源
dist/
  extension.js              打包后的扩展入口
```

## 已知限制

- 当前是非流式响应，必须等接口完整返回
- 文件写入没有二次审批，模型一旦返回合法 `<file>` 块就会直接落盘
- 只处理第一个工作区目录
- 自动附加的代码上下文有大小和数量上限
- 一些内部命名仍保留 `ollamaCoder` 前缀，这是历史实现遗留，不影响功能

## 开发

```bash
npm install
npm run lint
npm run compile
npm run watch
npm run package
```

## 运行要求

- VS Code `^1.109.3`
- Node.js 与 npm
- 本地 Ollama，或任意可访问的 OpenAI 兼容模型服务

## 许可证

本项目采用 [MIT License](LICENSE)。

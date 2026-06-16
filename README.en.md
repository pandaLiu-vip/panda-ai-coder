下面是将 README 全部翻译成英文后的版本（可直接保存为 `README.en.md`）。

````markdown
# Panda AI Coder

Read in other languages:

[中文](README.zh.md) | [English](README.en.md)

<p>Author: Chenhui Liu</p>
<p align="left"><img src="resources/author.png" width="100" alt="author"></p>
<p>Tel/WeChat: 15799845845</p>

`Panda AI Coder` is a VS Code sidebar extension that integrates local Ollama or OpenAI-compatible model services into the editor. It supports multi-threaded conversations around the current workspace and can directly write files back into the project when the model returns standardized file blocks.

## Current Features

- Provides a dedicated `Panda AI Coder` view in the Activity Bar
- Supports multi-task threads, recent tasks, search history, thread switching, creating, and deleting threads
- Supports both local Ollama and OpenAI-compatible `/v1` APIs
- Allows saving the service URL, API key, remote model name, and maximum token count in the sidebar settings
- Automatically reads the current workspace file tree and selects a small number of relevant code snippets as context based on:
  - Task content
  - Active editor
  - Attached file paths
- Automatically writes files into the current workspace when the model returns content in `<file path="...">...</file>` format
- Displays token usage if the model API returns usage information

## Features That Are NOT Currently Implemented

The following capabilities have **not** been implemented yet and should not be documented or promised externally:

- Streaming output
- Diff approval / rollback workflows
- Explicit `@file` / `@folder` reference parsing
- Command execution, test running, and lint feedback
- Git integration, MCP, Web Search, and sub-agents
- VS Code `Settings` page integration through `contributes.configuration`

Currently, all service configuration options are managed within the extension's own sidebar settings panel instead of the global VS Code Settings page.

## How It Works

The extension registers two commands after activation:

- `panda-ai-coder.openSidebar`
- `panda-ai-coder.newThread`

Workflow:

1. Open `Panda AI Coder` from the Activity Bar.
2. Configure the model service URL in the settings panel.
3. For remote OpenAI-compatible services, optionally configure:
   - API key
   - Remote model name
   - Maximum token count
4. Refresh and select a model.
5. Enter a task description and send it.
6. The extension generates context based on the current thread and workspace, then sends a request to the model.
7. If the model returns plain text, the response is simply displayed.
8. If the model returns `<file path="relative/path">` blocks, the files are written directly into the workspace.

## Supported Model Services

### 1. Local Ollama

Default address:

```text
http://127.0.0.1:11434/api
```

The extension first attempts to fetch the model list through the API.

If the local Ollama API is unavailable and OpenAI mode is not enabled, it will additionally attempt to execute:

```bash
ollama list
```

### 2. OpenAI-Compatible APIs

If the service URL contains `/v1`, the extension will treat it as an OpenAI-compatible API and use:

- `POST /chat/completions`
- `GET /models`

The API key is automatically converted into the format:

```text
Bearer <token>
```

and stored securely in VS Code Secret Storage.

## File Writing Protocol

The current version relies on the model returning file content in the following format:

```xml
<file path="src/example.ts">
export const message = 'hello';
</file>
```

Requirements:

- `path` must be a relative workspace path
- Absolute paths are not allowed
- Paths cannot escape the workspace directory
- Binary files are not supported

The extension removes these `<file>` blocks before displaying the remaining explanatory text in the chat window and attaches a list of files that were written.

## Project Structure

```text
src/
  extension.ts              Extension activation entry point
  constants/ids.ts          Commands and view IDs
  ui/sidebarProvider.ts     Webview backend, model requests, context building, file writing

media/
  main.js                   Webview frontend logic
  main.css                  Webview styles

resources/
  *.png / *.svg             Icon resources

dist/
  extension.js              Bundled extension entry point
```

## Known Limitations

- Only non-streaming responses are supported; the extension waits for the entire response before displaying it
- There is no secondary approval process for file writing; once a valid `<file>` block is returned, it is immediately written to disk
- Only the first workspace folder is processed
- Automatically attached code context has size and quantity limits
- Some internal names still retain the `ollamaCoder` prefix due to historical implementation reasons, but this does not affect functionality

## Development

```bash
npm install
npm run lint
npm run compile
npm run watch
npm run package
```

## Requirements

- VS Code `^1.109.3`
- Node.js and npm
- Local Ollama or any accessible OpenAI-compatible model service

## License

This project is licensed under the [MIT License](LICENSE).
````
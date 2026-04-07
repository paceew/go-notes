# Tool 机制分析

本文档总结了 Claude Code 项目中 tool 的完整工作机制，涵盖注册、注入、请求、解析、执行五个阶段，以及 `prompt` 与 `description` 两个属性的区别。

## 目录

- [一、Tool 的注册与定义](#toc-tool-register)
- [二、Tool 注入到 API 请求](#toc-tool-api-inject)
- [三、请求 LLM](#toc-tool-request-llm)
- [四、解析 LLM 返回内容中的 Tool 调用](#toc-tool-parse)
- [五、执行 Tool](#toc-tool-execute)
- [六、`prompt` 与 `description` 的区别](#toc-prompt-vs-description)
  - [以 BashTool 为例](#toc-bashtool-example)
  - [一次 BashTool 调用的完整数据流](#toc-bashtool-flow)
- [关键文件索引](#toc-file-index)

---

<a id="toc-tool-register"></a>

## 一、Tool 的注册与定义

所有内置 tool 在 `src/tools.ts` 的 `getAllBaseTools()` 中集中注册，通过 `getTools(permissionContext)` 按权限和 feature flag 过滤后对外暴露。

```ts
// src/tools.ts
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool, FileEditTool, FileWriteTool,
    // ...
  ]
}
```

每个 tool 是满足 `Tool` 类型的对象，包含以下核心字段：

| 字段 | 作用 |
|------|------|
| `name` | tool 名称，LLM 用此名字发起调用 |
| `prompt()` | 生成发给 LLM 的 tool 能力描述（注入 API） |
| `description()` | 生成展示给用户的本次调用描述（用于权限 UI） |
| `inputSchema` | Zod schema，定义 LLM 调用时的参数结构 |
| `call()` | tool 的实际执行逻辑 |
| `checkPermissions()` | 决定是否需要向用户申请权限 |

通过 `buildTool()` 构建，自动填充 `isEnabled`、`isConcurrencySafe` 等默认值：

```ts
// src/Tool.ts
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def } as BuiltTool<D>
}
```

---

<a id="toc-tool-api-inject"></a>

## 二、Tool 注入到 API 请求

Tool **不是**以文本形式拼进 system prompt，而是通过 Anthropic Messages API 的 `tools` 字段传入。

核心转换函数是 `toolToAPISchema()`（`src/utils/api.ts`），将每个 tool 转为 API 所需结构：

```ts
// src/utils/api.ts
base = {
  name: tool.name,
  description: await tool.prompt({ getToolPermissionContext, tools, agents }),
  input_schema,  // 由 Zod schema 转换而来
}
```

在 `src/services/api/claude.ts` 的 `queryModel` 中批量生成所有 tool schema：

```ts
const toolSchemas = await Promise.all(
  filteredTools.map(tool => toolToAPISchema(tool, { ... }))
)
const allTools = [...toolSchemas, ...extraToolSchemas]
```

对于"延迟加载"（deferred）的 tool，还会额外插入一条合成 user 消息列出它们：

```ts
messagesForAPI = [
  createUserMessage({
    content: `<available-deferred-tools>\n${deferredToolList}\n</available-deferred-tools>`,
    isMeta: true,
  }),
  ...messagesForAPI,
]
```

---

<a id="toc-tool-request-llm"></a>

## 三、请求 LLM

通过 `anthropic.beta.messages.create` 以流式方式发起请求，`allTools` 作为 `tools` 字段传入：

```ts
// src/services/api/claude.ts
const result = await anthropic.beta.messages.create(
  { ...params, stream: true },
  { signal, headers: { [CLIENT_REQUEST_ID_HEADER]: clientRequestId } }
).withResponse()

// params 包含：
// model, messages, system, tools: allTools,
// tool_choice, betas, max_tokens, thinking, temperature, ...
```

调用链路：`query.ts` → `queryModelWithStreaming()` → `queryModel()` → `anthropic.beta.messages.create()`

---

<a id="toc-tool-parse"></a>

## 四、解析 LLM 返回内容中的 Tool 调用

流式响应中，`tool_use` block 的 JSON input 通过 `input_json_delta` 事件**逐片拼接**：

```ts
// src/services/api/claude.ts
case 'content_block_start':
  if (part.content_block.type === 'tool_use') {
    contentBlocks[part.index] = { ...part.content_block, input: '' }
  }

case 'content_block_delta':
  if (delta.type === 'input_json_delta') {
    contentBlock.input += delta.partial_json  // 逐片追加
  }
```

流结束后，通过 `normalizeContentFromAPI()`（`src/utils/messages.ts`）将字符串 JSON.parse 成对象，并应用 tool-specific 修正：

```ts
case 'tool_use': {
  const parsed = safeParseJSON(contentBlock.input)
  normalizedInput = parsed ?? {}
  // 应用 tool 自定义的输入修正
  normalizedInput = normalizeToolInput(tool, normalizedInput, agentId)
  return { ...contentBlock, input: normalizedInput }
}
```

`query.ts` 的 agentic loop 中收集所有 `tool_use` block：

```ts
const msgToolUseBlocks = message.message.content.filter(
  content => content.type === 'tool_use'
) as ToolUseBlock[]
if (msgToolUseBlocks.length > 0) {
  toolUseBlocks.push(...msgToolUseBlocks)
  needsFollowUp = true
}
```

---

<a id="toc-tool-execute"></a>

## 五、执行 Tool

`runTools()`（`src/services/tools/toolOrchestration.ts`）将 tool 调用按并发安全性分批，再逐个交给 `runToolUse()`：

```ts
for (const { isConcurrencySafe, blocks } of partitionToolCalls(toolUseMessages, currentContext)) {
  // 并发安全的批次并行执行，否则串行
}
```

`runToolUse()`（`src/services/tools/toolExecution.ts`）按名字查找 tool，做权限检查，然后调用 `tool.call()`：

```ts
let tool = findToolByName(toolUseContext.options.tools, toolName)

const result = await tool.call(
  callInput,
  { ...toolUseContext, toolUseId: toolUseID },
  canUseTool,
  assistantMessage,
  onProgress,
)
```

执行结果被封装成 `tool_result` 类型的 user 消息追加到对话历史，然后 agentic loop 带着新历史重新请求 LLM，循环直到没有新的 tool call 为止。

---

<a id="toc-prompt-vs-description"></a>

## 六、`prompt` 与 `description` 的区别

Tool 类型上有两个容易混淆的方法，它们服务于**完全不同的受众**：

| | `prompt()` | `description()` |
|---|---|---|
| **受众** | LLM（Anthropic API） | 用户（权限提示 UI） |
| **时机** | 每次请求前，静态生成 | tool 被调用时，动态生成 |
| **入参** | 通用上下文（tools、agents、permissionContext） | **本次调用的具体 input 参数** |
| **用途** | 告诉模型这个 tool 能做什么、怎么用 | 告诉用户"这次 AI 在做什么" |

<a id="toc-bashtool-example"></a>

### 以 BashTool 为例

**`prompt()`** 调用 `getSimplePrompt()`，生成几百字的静态使用规范：

```ts
// src/tools/BashTool/BashTool.tsx
async prompt() {
  return getSimplePrompt()  // 描述 BashTool 能力、禁用命令、git规范、sandbox限制等
}
```

内容包括：工具能力描述、应避免的命令、多命令并行规范、git 安全操作规范、commit/PR 流程、sandbox 限制等。

**BashTool 的 inputSchema** 中专门有一个可选的 `description` 字段，引导 LLM 在调用时自我描述：

```ts
description: z.string().optional().describe(
  `Clear, concise description of what this command does in active voice.
   - ls → "List files in current directory"
   - git status → "Show working tree status"`
)
```

**`description()`** 直接读取 LLM 填入的 `input.description`，展示给用户：

```ts
async description({ description }) {
  return description || 'Run shell command'
}
```

由 `useCanUseTool.tsx` 在向用户申请权限时调用：

```ts
const description = await tool.description(input, { isNonInteractiveSession, toolPermissionContext, tools })
// → 展示在权限确认弹窗上
```

<a id="toc-bashtool-flow"></a>

### 一次 BashTool 调用的完整数据流

```
① 请求前
   prompt() → "Executes a given bash command...（几百字使用规范）"
                    ↓ 作为 tools[].description 注入 Anthropic API

② LLM 决定调用 BashTool，返回 tool_use block：
   {
     name: "Bash",
     input: {
       command: "npm test -- --testPathPattern=auth",
       description: "Run unit tests for the auth module"  ← LLM 自己填写
     }
   }

③ 执行前需要用户确认
   description(input) → "Run unit tests for the auth module"
                              ↓ 展示在权限提示 UI 上给用户看

④ 用户点击允许 → tool.call() 执行命令
                              ↓
⑤ 执行结果封装为 tool_result user message → 追加到对话历史
                              ↓
⑥ 带新历史重新请求 LLM → 循环直到无 tool call
```

---

<a id="toc-file-index"></a>

## 关键文件索引

| 职责 | 文件路径 |
|------|----------|
| Tool 注册与过滤 | `src/tools.ts` |
| Tool 类型定义与 `buildTool` | `src/Tool.ts` |
| Tool schema 转换（prompt → API） | `src/utils/api.ts` |
| LLM 请求构建与流式响应解析 | `src/services/api/claude.ts` |
| tool_use 输入归一化 | `src/utils/messages.ts` |
| Agentic loop 与 tool_use 收集 | `src/query.ts` |
| Tool 执行编排（并发/串行分批） | `src/services/tools/toolOrchestration.ts` |
| 单个 tool 权限检查与调用 | `src/services/tools/toolExecution.ts` |
| 用户权限确认（调用 description） | `src/hooks/useCanUseTool.tsx` |
| BashTool 实现 | `src/tools/BashTool/BashTool.tsx` |
| BashTool prompt 内容 | `src/tools/BashTool/prompt.ts` |
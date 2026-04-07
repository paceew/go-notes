# Claude-Code Agent

参考：https://github.com/paceew/build-code-agent

## 目录

- [Tool-analysis](#toc-tool-analysis)
- [兜底策略分析：LLM 返回非预期格式时的处理链路](#toc-fallback-chain)
  - [第一层：JSON 解析兜底（`normalizeContentFromAPI`）](#toc-layer-json-parse)
  - [第二层：Zod Schema 校验 → 把错误反馈给 LLM（`toolExecution.ts`）](#toc-layer-zod)
  - [第三层：工具自身业务校验（`tool.validateInput`）](#toc-layer-tool-validate)
  - [整体流程图](#toc-flow-diagram)
  - [潜在的边界问题](#toc-edge-cases)
- [cc-code-autoCompact.prompt](#toc-cc-autocompact)

---

<a id="toc-tool-analysis"></a>

## Tool-analysis:

详见：[tool-analysis.md](tool-analysis.md)

<a id="toc-fallback-chain"></a>

## 兜底策略分析：LLM 返回非预期格式时的处理链路
整个系统有**三层防护**，形成一个"自愈闭环"：

<a id="toc-layer-json-parse"></a>

### 第一层：JSON 解析兜底（`normalizeContentFromAPI`）
```2676:2694:src/utils/messages.ts
        if (typeof contentBlock.input === 'string') {
          const parsed = safeParseJSON(contentBlock.input)
          if (parsed === null && contentBlock.input.length > 0) {
            // TET/FC-v3 diagnostic: the streamed tool input JSON failed to
            // parse. We fall back to {} which means downstream validation
            // sees empty input. The raw prefix goes to debug log only — no
            // PII-tagged proto column exists for it yet.
            logEvent('tengu_tool_input_json_parse_fail', {
              toolName: sanitizeToolNameForAnalytics(contentBlock.name),
              inputLen: contentBlock.input.length,
            })
            if (process.env.USER_TYPE === 'ant') {
              logForDebugging(
                `tool input JSON parse fail: ${contentBlock.input.slice(0, 200)}`,
                { level: 'warn' },
              )
            }
          }
          normalizedInput = parsed ?? {}
```
- 流式 API 返回的 `input_json_delta` 会被拼接成字符串
- 调用 `safeParseJSON` 尝试解析，**失败则 fallback 到 `{}`（空对象）**
- 同时触发 `tengu_tool_input_json_parse_fail` 埋点，内部用户会打 warn 日志

<a id="toc-layer-zod"></a>

### 第二层：Zod Schema 校验 → 把错误反馈给 LLM（`toolExecution.ts`）
```614:679:src/services/tools/toolExecution.ts
  // Validate input types with zod (surprisingly, the model is not great at generating valid input)
  const parsedInput = tool.inputSchema.safeParse(input)
  if (!parsedInput.success) {
    let errorContent = formatZodValidationError(tool.name, parsedInput.error)
    const schemaHint = buildSchemaNotSentHint(
      tool,
      toolUseContext.messages,
      toolUseContext.options.tools,
    )
    if (schemaHint) {
      // ...
      errorContent += schemaHint
    }
    // ...
    return [
      {
        message: createUserMessage({
          content: [
            {
              type: 'tool_result',
              content: `<tool_use_error>InputValidationError: ${errorContent}</tool_use_error>`,
              is_error: true,
              tool_use_id: toolUseID,
            },
          ],
```
- 第一层 fallback 到 `{}` 后，Zod 校验**必然失败**（缺少必填字段）
- 将 Zod 错误信息格式化后，以 `tool_result` + `is_error: true` 的形式**送回给 LLM**
- 还会通过 `buildSchemaNotSentHint` 附上 schema 提示，帮助 LLM 理解正确格式

<a id="toc-layer-tool-validate"></a>

### 第三层：工具自身业务校验（`tool.validateInput`）
```682:729:src/services/tools/toolExecution.ts
  // Validate input values. Each tool has its own validation logic
  const isValidCall = await tool.validateInput?.(
    parsedInput.data,
    toolUseContext,
  )
  if (isValidCall?.result === false) {
    // ...
    return [
      {
        message: createUserMessage({
          content: [
            {
              type: 'tool_result',
              content: `<tool_use_error>${isValidCall.message}</tool_use_error>`,
              is_error: true,
              tool_use_id: toolUseID,
            },
```
每个工具可以定义自己的业务逻辑校验，失败同样以 `tool_use_error` 反馈给 LLM。

---

<a id="toc-flow-diagram"></a>

### 整体流程图
```
LLM 流式输出
     │
     ▼
input_json_delta 累积字符串
     │
     ▼
content_block_stop → normalizeContentFromAPI
     │
     ├─ safeParseJSON 成功 → 正常对象
     │
     └─ JSON 解析失败 → fallback to {}
          + logEvent('tengu_tool_input_json_parse_fail')
               │
               ▼
        Zod inputSchema.safeParse({})
               │
               └─ 校验失败 → tool_result { is_error: true, content: "<tool_use_error>InputValidationError..." }
                    │
                    ▼
              LLM 收到错误 → 自动重试，生成正确格式
```
**核心设计思想**：不是在客户端直接抛异常终止，而是将错误以 `tool_use_error` 的形式**重新注入对话上下文**，让 LLM 看到自己的错误后自我修正——这是一个"反馈闭环"而非"硬中止"。

<a id="toc-edge-cases"></a>

### 潜在的边界问题
这套机制有一个值得注意的地方：当 JSON 解析失败时，fallback 到 `{}` 会**静默吞掉原始内容**。如果 LLM 返回的是非 JSON 格式的纯文本（而不是损坏的 JSON），Zod 错误信息只会告诉 LLM"缺少必填字段"，而不会告诉它"你返回的不是 JSON"。这种情况下 LLM 有可能反复犯同样的错误，注释里的 `TODO: This needs patching` 也印证了这一点。


<a id="toc-cc-autocompact"></a>

## cc-code-autoCompact.prompt:
Your task is to create a detailed summary of the conversation so far,
paying close attention to the user's explicit requests and your previous actions.
Your summary should include the following sections:
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and fixes
5. Problem Solving
6. All user messages
7. Pending Tasks
8. Current Work
9. Optional Next Step


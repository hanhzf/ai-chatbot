## 工具与大模型的交互机制

### 1. **工具分类和交互模式**

**工具可以分为两大类**：

| 工具类型 | 交互模式 | 大模型角色 |
|---------|---------|-----------|
| `getWeather` | 数据获取工具 | 仅使用结果 |
| `createDocument` | 内容生成工具 | 调用另一个大模型 |
| `updateDocument` | 内容生成工具 | 调用另一个大模型 |
| `requestSuggestions` | 内容生成工具 | 调用另一个大模型 |

### 2. **数据获取工具（getWeather）**

**交互机制**：
```
用户请求 → 大模型决策 → 调用getWeather → 获取外部数据 → 返回给大模型 → 大模型生成回复
```

**特点**：
- 大模型只使用工具的结果
- 不涉及内容生成
- 一次性返回完整数据

### 3. **内容生成工具（createDocument/updateDocument）**

**交互机制**：
```
用户请求 → 大模型A决策 → 调用工具 → 工具内部调用大模型B → 大模型B生成内容 → 流式返回 → 大模型A继续对话
```

**详细流程**：

**第一步：大模型A的决策**
```150:170:app/(chat)/api/chat/route.ts
const result = streamText({
  model: myProvider.languageModel(selectedChatModel), // 大模型A
  system: systemPrompt({ selectedChatModel, requestHints }),
  messages: convertToModelMessages(uiMessages),
  tools: {
    createDocument: createDocument({ session, dataStream }),
    updateDocument: updateDocument({ session, dataStream }),
  },
});
```

**第二步：工具调用大模型B**
```10:35:artifacts/text/server.ts
const { fullStream } = streamText({
  model: myProvider.languageModel('artifact-model'), // 大模型B
  system:
    'Write about the given topic. Markdown is supported. Use headings wherever appropriate.',
  experimental_transform: smoothStream({ chunking: 'word' }),
  prompt: title,
});
```

**第三步：流式内容生成**
```25:35:artifacts/text/server.ts
for await (const delta of fullStream) {
  const { type } = delta;

  if (type === 'text') {
    const { text } = delta;

    draftContent += text;

    dataStream.write({
      type: 'data-textDelta',
      data: text,
      transient: true,
    });
  }
}
```

### 4. **不同大模型的职责分工**

**大模型A（chat-model）**：
- 负责理解用户意图
- 决定调用哪个工具
- 处理工具参数
- 生成最终回复

**大模型B（artifact-model）**：
- 专门负责内容生成
- 根据特定提示词生成内容
- 流式输出生成结果

**模型配置**：
```15:30:lib/ai/providers.ts
export const myProvider = customProvider({
  languageModels: {
    'chat-model': xai('grok-2-vision-1212'),        // 对话模型
    'chat-model-reasoning': wrapLanguageModel({
      model: xai('grok-3-mini-beta'),
      middleware: extractReasoningMiddleware({ tagName: 'think' }),
    }),
    'title-model': xai('grok-2-1212'),              // 标题生成
    'artifact-model': xai('grok-2-1212'),           // 内容生成
  },
});
```

### 5. **具体工具分析**

**createDocument工具**：
```15:77:lib/ai/tools/create-document.ts
execute: async ({ title, kind }) => {
  // 1. 发送元数据
  dataStream.write({
    type: 'data-kind',
    data: kind,
    transient: true,
  });

  // 2. 调用文档处理器
  await documentHandler.onCreateDocument({
    id,
    title,
    dataStream,
    session,
  });

  // 3. 文档处理器内部调用大模型B生成内容
  // 4. 流式返回生成的内容
};
```

**updateDocument工具**：
```15:63:lib/ai/tools/update-document.ts
execute: async ({ id, description }) => {
  // 1. 获取现有文档
  const document = await getDocumentById({ id });

  // 2. 调用文档处理器
  await documentHandler.onUpdateDocument({
    document,
    description,
    dataStream,
    session,
  });

  // 3. 文档处理器内部调用大模型B更新内容
  // 4. 流式返回更新的内容
};
```

**requestSuggestions工具**：
```25:93:lib/ai/tools/request-suggestions.ts
const { elementStream } = streamObject({
  model: myProvider.languageModel('artifact-model'), // 大模型B
  system:
    'You are a help writing assistant. Given a piece of writing, please offer suggestions to improve the piece of writing and describe the change.',
  prompt: document.content,
  schema: z.object({
    originalSentence: z.string(),
    suggestedSentence: z.string(),
    description: z.string(),
  }),
});

for await (const element of elementStream) {
  // 流式处理建议
  dataStream.write({
    type: 'data-suggestion',
    data: suggestion,
    transient: true,
  });
}
```

### 6. **交互架构总结**

**整体架构**：
```
用户 → 大模型A(决策) → 工具 → 大模型B(生成) → 流式内容 → 前端渲染
```

**关键特点**：
1. **分层设计**：决策层和生成层分离
2. **专门化**：不同大模型负责不同任务
3. **流式处理**：内容生成采用流式方式
4. **实时反馈**：用户可以看到内容逐步生成

这种设计实现了**智能的任务分工**，让每个大模型专注于自己擅长的领域，同时通过流式处理提供更好的用户体验。
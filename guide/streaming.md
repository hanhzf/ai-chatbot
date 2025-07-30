## 大模型工具调用的返回机制

**大模型在调用工具时是采用Streaming（流式）返回的**，具体机制如下：

### 1. **双层流式处理架构**

**第一层：大模型决策流**
```150:170:app/(chat)/api/chat/route.ts
const result = streamText({
  model: myProvider.languageModel(selectedChatModel),
  system: systemPrompt({ selectedChatModel, requestHints }),
  messages: convertToModelMessages(uiMessages),
  experimental_transform: smoothStream({ chunking: 'word' }),
  tools: {
    getWeather,
    createDocument: createDocument({ session, dataStream }),
    updateDocument: updateDocument({ session, dataStream }),
    requestSuggestions: requestSuggestions({
      session,
      dataStream,
    }),
  },
});
```

**第二层：工具执行流**
```15:77:lib/ai/tools/create-document.ts
execute: async ({ title, kind }) => {
  const id = generateUUID();

  dataStream.write({
    type: 'data-kind',
    data: kind,
    transient: true,
  });

  dataStream.write({
    type: 'data-id',
    data: id,
    transient: true,
  });

  dataStream.write({
    type: 'data-title',
    data: title,
    transient: true,
  });

  dataStream.write({
    type: 'data-clear',
    data: null,
    transient: true,
  });

  await documentHandler.onCreateDocument({
    id,
    title,
    dataStream,
    session,
  });

  dataStream.write({ type: 'data-finish', data: null, transient: true });
```

### 2. **工具内部的流式内容生成**

**文本文档的流式生成**：
```10:35:artifacts/text/server.ts
const { fullStream } = streamText({
  model: myProvider.languageModel('artifact-model'),
  system:
    'Write about the given topic. Markdown is supported. Use headings wherever appropriate.',
  experimental_transform: smoothStream({ chunking: 'word' }),
  prompt: title,
});

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

**代码文档的流式生成**：
```10:35:artifacts/code/server.ts
const { fullStream } = streamObject({
  model: myProvider.languageModel('artifact-model'),
  system: codePrompt,
  prompt: title,
  schema: z.object({
    code: z.string(),
  }),
});

for await (const delta of fullStream) {
  const { type } = delta;

  if (type === 'object') {
    const { object } = delta;
    const { code } = object;

    if (code) {
      dataStream.write({
        type: 'data-codeDelta',
        data: code ?? '',
        transient: true,
      });

      draftContent = code;
    }
  }
}
```

### 3. **前端实时渲染机制**

**数据流处理**：
```15:35:components/data-stream-handler.tsx
newDeltas.forEach((delta) => {
  const artifactDefinition = artifactDefinitions.find(
    (artifactDefinition) => artifactDefinition.kind === artifact.kind,
  );

  if (artifactDefinition?.onStreamPart) {
    artifactDefinition.onStreamPart({
      streamPart: delta,
      setArtifact,
      setMetadata,
    });
  }

  setArtifact((draftArtifact) => {
    // 根据delta类型更新artifact状态
  });
});
```

**文本内容的实时更新**：
```35:50:artifacts/text/client.tsx
if (streamPart.type === 'data-textDelta') {
  setArtifact((draftArtifact) => {
    return {
      ...draftArtifact,
      content: draftArtifact.content + streamPart.data,
      isVisible:
        draftArtifact.status === 'streaming' &&
        draftArtifact.content.length > 400 &&
        draftArtifact.content.length < 450
          ? true
          : draftArtifact.isVisible,
      status: 'streaming',
    };
  });
}
```

### 4. **流式处理的优势**

1. **实时反馈**: 用户可以立即看到内容生成过程
2. **渐进式显示**: 内容逐步出现，避免长时间等待
3. **交互性**: 在生成过程中可以进行交互
4. **性能优化**: 减少感知延迟，提升用户体验

### 5. **流式数据流程**

```
用户请求 → 大模型决策 → 工具调用 → 内容流式生成 → 
实时数据流 → 前端渲染 → 用户看到实时更新
```

**具体流程**：
1. 大模型分析用户请求，决定调用工具
2. 工具开始执行，立即发送元数据（类型、ID、标题等）
3. 工具内部调用另一个大模型生成内容
4. 内容以流式方式生成，每个片段都实时发送到前端
5. 前端实时接收并渲染这些数据片段
6. 当内容达到一定长度时，自动显示Artifact弹框
7. 继续流式生成，直到完成

这种设计实现了**真正的流式体验**，用户可以看到内容逐步生成的过程，而不是等待完整结果后一次性显示。
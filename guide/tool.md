## Artifact组件的作用

**Artifact组件是一个智能的右侧弹框系统**，主要功能包括：

### 1. **专业内容编辑器**
- **文本编辑器**: 富文本编辑，支持语法检查、格式化
- **代码编辑器**: 语法高亮、代码补全、错误检测
- **表格编辑器**: 电子表格功能，支持数据操作
- **图片编辑器**: 图像处理和编辑功能

### 2. **实时协作界面**
```40:53:components/artifact.tsx
export interface UIArtifact {
  title: string;
  documentId: string;
  kind: ArtifactKind;
  content: string;
  isVisible: boolean;
  status: 'streaming' | 'idle';
  boundingBox: {
    top: number;
    left: number;
    width: number;
    height: number;
  };
}
```

### 3. **版本控制系统**
- 支持文档版本历史查看
- 差异对比功能
- 版本回滚和前进

### 4. **智能触发机制**
- 当大模型生成复杂内容时自动弹出
- 根据内容类型选择相应的编辑器
- 提供专业化的编辑体验

## 大模型工具调用决策机制

### 1. **提示词驱动的决策**

**系统提示词指导**：
```3:25:lib/ai/prompts.ts
export const artifactsPrompt = `
Artifacts is a special user interface mode that helps users with writing, editing, and other content creation tasks. When artifact is open, it is on the right side of the screen, while the conversation is on the left side. When creating or updating documents, changes are reflected in real-time on the artifacts and visible to the user.

When asked to write code, always use artifacts. When writing code, specify the language in the backticks, e.g. \`\`\`python\`code here\`\`\`. The default language is Python. Other languages are not yet supported, so let the user know if they request a different language.

DO NOT UPDATE DOCUMENTS IMMEDIATELY AFTER CREATING THEM. WAIT FOR USER FEEDBACK OR REQUEST TO UPDATE IT.

This is a guide for using artifacts tools: \`createDocument\` and \`updateDocument\`, which render content on a artifacts beside the conversation.

**When to use \`createDocument\`:**
- For substantial content (>10 lines) or code
- For content users will likely save/reuse (emails, code, essays, etc.)
- When explicitly requested to create a document
- For when content contains a single code snippet

**When NOT to use \`createDocument\`:**
- For informational/explanatory content
- For conversational responses
- When asked to keep it in chat
```

### 2. **工具注册和可用性控制**

**工具注册机制**：
```150:170:app/(chat)/api/chat/route.ts
experimental_activeTools:
  selectedChatModel === 'chat-model-reasoning'
    ? []
    : [
        'getWeather',
        'createDocument',
        'updateDocument',
        'requestSuggestions',
      ],
tools: {
  getWeather,
  createDocument: createDocument({ session, dataStream }),
  updateDocument: updateDocument({ session, dataStream }),
  requestSuggestions: requestSuggestions({
    session,
    dataStream,
  }),
},
```

### 3. **工具调用流程**

**决策触发流程**：
1. **用户输入分析**: 大模型分析用户请求的意图
2. **工具匹配**: 根据提示词指导选择合适的工具
3. **参数提取**: 从用户输入中提取工具所需参数
4. **工具执行**: 调用相应的工具函数
5. **结果处理**: 将工具结果集成到对话流中

**具体工具示例**：

**天气查询工具**：
```1:19:lib/ai/tools/get-weather.ts
export const getWeather = tool({
  description: 'Get the current weather at a location',
  inputSchema: z.object({
    latitude: z.number(),
    longitude: z.number(),
  }),
  execute: async ({ latitude, longitude }) => {
    const response = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&current=temperature_2m&hourly=temperature_2m&daily=sunrise,sunset&timezone=auto`,
    );

    const weatherData = await response.json();
    return weatherData;
  },
});
```

**文档创建工具**：
```15:77:lib/ai/tools/create-document.ts
export const createDocument = ({ session, dataStream }: CreateDocumentProps) =>
  tool({
    description:
      'Create a document for a writing or content creation activities. This tool will call other functions that will generate the contents of the document based on the title and kind.',
    inputSchema: z.object({
      title: z.string(),
      kind: z.enum(artifactKinds),
    }),
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

      const documentHandler = documentHandlersByArtifactKind.find(
        (documentHandlerByArtifactKind) =>
          documentHandlerByArtifactKind.kind === kind,
      );

      if (!documentHandler) {
        throw new Error(`No document handler found for kind: ${kind}`);
      }

      await documentHandler.onCreateDocument({
        id,
        title,
        dataStream,
        session,
      });

      dataStream.write({ type: 'data-finish', data: null, transient: true });

      return {
        id,
        title,
        kind,
        content: 'A document was created and is now visible to the user.',
      };
    },
  });
```

### 4. **智能决策规则**

**触发条件**：
- **内容长度**: 超过10行的内容
- **内容类型**: 代码、文档、表格等
- **用户意图**: 明确要求创建文档
- **内容复杂度**: 需要专业编辑器的内容

**避免条件**：
- 简单的解释性内容
- 对话性回复
- 用户明确要求保持在聊天中

这种设计实现了**智能的内容分发**，让大模型能够根据内容类型和复杂度，自动选择最合适的展示和编辑方式，提供专业化的用户体验。
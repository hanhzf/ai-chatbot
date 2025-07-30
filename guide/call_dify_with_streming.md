## 实现方案

### 1. **创建自定义工具**

```typescript
// lib/ai/tools/call-dify-agent.ts
import { tool, type UIMessageStreamWriter } from 'ai';
import { z } from 'zod';
import type { ChatMessage } from '@/lib/types';

interface CallDifyAgentProps {
  dataStream: UIMessageStreamWriter<ChatMessage>;
}

export const callDifyAgent = ({ dataStream }: CallDifyAgentProps) =>
  tool({
    description: 'Call a Dify AI Agent to perform specific tasks',
    inputSchema: z.object({
      agentId: z.string().describe('The ID of the Dify agent to call'),
      message: z.string().describe('The message to send to the agent'),
      parameters: z.record(z.any()).optional().describe('Additional parameters for the agent'),
    }),
    execute: async ({ agentId, message, parameters = {} }) => {
      // 1. 发送开始信号
      dataStream.write({
        type: 'data-agent-start',
        data: { agentId, message },
        transient: true,
      });

      // 2. 调用Dify Agent的流式接口
      const response = await fetch(`/api/dify-agent/${agentId}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message, parameters }),
      });

      if (!response.ok) {
        throw new Error('Failed to call Dify agent');
      }

      // 3. 处理流式响应
      const reader = response.body?.getReader();
      if (!reader) {
        throw new Error('No response body');
      }

      let accumulatedContent = '';

      try {
        while (true) {
          const { done, value } = await reader.read();
          
          if (done) break;

          // 解析流式数据
          const chunk = new TextDecoder().decode(value);
          const lines = chunk.split('\n');

          for (const line of lines) {
            if (line.startsWith('data: ')) {
              const data = JSON.parse(line.slice(6));
              
              if (data.type === 'message') {
                accumulatedContent += data.content;
                
                // 实时发送到前端
                dataStream.write({
                  type: 'data-agent-delta',
                  data: data.content,
                  transient: true,
                });
              }
            }
          }
        }
      } finally {
        reader.releaseLock();
      }

      // 4. 发送完成信号
      dataStream.write({
        type: 'data-agent-finish',
        data: { content: accumulatedContent },
        transient: true,
      });

      return {
        agentId,
        content: accumulatedContent,
        message: 'Agent execution completed successfully.',
      };
    },
  });
```

### 2. **创建Dify Agent API路由**

```typescript
// app/(chat)/api/dify-agent/[agentId]/route.ts
import { NextRequest } from 'next/server';

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ agentId: string }> }
) {
  const { agentId } = await params();
  const { message, parameters } = await request.json();

  // 调用Dify Agent的流式接口
  const difyResponse = await fetch(`https://your-dify-instance.com/v1/chat-messages`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.DIFY_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      inputs: parameters,
      query: message,
      response_mode: 'streaming', // 关键：使用流式模式
      user: 'user-id',
    }),
  });

  // 返回流式响应
  return new Response(difyResponse.body, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

### 3. **注册工具到聊天API**

```typescript
// app/(chat)/api/chat/route.ts
import { callDifyAgent } from '@/lib/ai/tools/call-dify-agent';

// 在工具注册部分添加
const stream = createUIMessageStream({
  execute: ({ writer: dataStream }) => {
    const result = streamText({
      model: myProvider.languageModel(selectedChatModel),
      system: systemPrompt({ selectedChatModel, requestHints }),
      messages: convertToModelMessages(uiMessages),
      experimental_activeTools: [
        'getWeather',
        'createDocument',
        'updateDocument',
        'requestSuggestions',
        'callDifyAgent', // 添加新工具
      ],
      tools: {
        getWeather,
        createDocument: createDocument({ session, dataStream }),
        updateDocument: updateDocument({ session, dataStream }),
        requestSuggestions: requestSuggestions({ session, dataStream }),
        callDifyAgent: callDifyAgent({ dataStream }), // 注册工具
      },
    });
  },
});
```

### 4. **前端渲染处理**

```typescript
// components/message.tsx
// 在消息渲染部分添加Dify Agent的处理

if (type === 'tool-callDifyAgent') {
  const { toolCallId, state } = part;

  if (state === 'input-available') {
    const { input } = part;
    return (
      <div key={toolCallId}>
        <div className="text-sm text-muted-foreground">
          Calling Dify Agent: {input.agentId}
        </div>
      </div>
    );
  }

  if (state === 'output-available') {
    const { output } = part;
    return (
      <div key={toolCallId}>
        <div className="p-4 border rounded-lg">
          <div className="font-medium mb-2">Agent Response:</div>
          <div className="whitespace-pre-wrap">{output.content}</div>
        </div>
      </div>
    );
  }
}
```

### 5. **数据流处理**

```typescript
// components/data-stream-handler.tsx
// 添加Dify Agent的数据流处理

setArtifact((draftArtifact) => {
  switch (delta.type) {
    // ... 现有处理逻辑
    
    case 'data-agent-start':
      return {
        ...draftArtifact,
        status: 'streaming',
        agentId: delta.data.agentId,
      };

    case 'data-agent-delta':
      return {
        ...draftArtifact,
        content: (draftArtifact.content || '') + delta.data,
        status: 'streaming',
      };

    case 'data-agent-finish':
      return {
        ...draftArtifact,
        status: 'idle',
        content: delta.data.content,
      };

    default:
      return draftArtifact;
  }
});
```

## 实现优势

1. **无缝集成**：与现有工具架构完全兼容
2. **流式体验**：用户可以看到Agent实时响应
3. **灵活配置**：支持不同Agent和参数
4. **错误处理**：完善的错误处理机制

## 关键要点

1. **流式处理**：确保Dify Agent返回流式数据
2. **数据格式**：统一数据流格式
3. **状态管理**：正确处理Agent执行状态
4. **用户体验**：提供实时反馈和进度指示

这种实现方式可以让您的Dify Agent无缝集成到现有系统中，同时保持流式的用户体验。
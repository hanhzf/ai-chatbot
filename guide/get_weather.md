## `getWeather` 工具的返回机制

**`getWeather` 工具是采用一次性返回的，不是流式返回的。**

### 1. **工具实现分析**

**`getWeather` 工具的实现**：
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

**关键特点**：
- 使用 `await fetch()` 等待完整响应
- 使用 `response.json()` 获取完整数据
- 直接 `return weatherData` 返回完整结果
- **没有使用 `dataStream.write()` 进行流式传输**

### 2. **前端渲染机制**

**天气工具的前端处理**：
```167:185:components/message.tsx
if (type === 'tool-getWeather') {
  const { toolCallId, state } = part;

  if (state === 'input-available') {
    return (
      <div key={toolCallId} className="skeleton">
        <Weather />
      </div>
    );
  }

  if (state === 'output-available') {
    const { output } = part;
    return (
      <div key={toolCallId}>
        <Weather weatherAtLocation={output} />
      </div>
    );
  }
}
```

**渲染特点**：
- 只有两个状态：`input-available` 和 `output-available`
- 没有中间状态或流式更新
- 结果一次性显示完整的天气数据

### 3. **与文档创建工具的对比**

**文档创建工具（流式）**：
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

  // ... 更多流式数据写入

  await documentHandler.onCreateDocument({
    id,
    title,
    dataStream,
    session,
  });

  dataStream.write({ type: 'data-finish', data: null, transient: true });
```

**天气工具（一次性）**：
```10:19:lib/ai/tools/get-weather.ts
execute: async ({ latitude, longitude }) => {
  const response = await fetch(
    `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&current=temperature_2m&hourly=temperature_2m&daily=sunrise,sunset&timezone=auto`,
  );

  const weatherData = await response.json();
  return weatherData;
},
```

### 4. **设计原因分析**

**为什么天气工具不使用流式返回**：

1. **数据特性**：天气数据是结构化的JSON，不需要逐步生成
2. **API特性**：外部天气API返回完整数据，无法流式获取
3. **用户体验**：天气信息通常较小，一次性显示更合适
4. **性能考虑**：避免不必要的流式处理开销

**为什么文档工具使用流式返回**：

1. **内容特性**：文本/代码需要逐步生成和显示
2. **用户体验**：长内容需要实时反馈
3. **交互性**：在生成过程中可以进行编辑和交互

### 5. **总结**

**工具返回机制的分类**：

| 工具类型 | 返回机制 | 原因 |
|---------|---------|------|
| `getWeather` | 一次性返回 | 外部API，结构化数据 |
| `createDocument` | 流式返回 | 内容生成，实时反馈 |
| `updateDocument` | 流式返回 | 内容生成，实时反馈 |
| `requestSuggestions` | 流式返回 | 内容生成，实时反馈 |

**`getWeather` 工具采用一次性返回机制**，这是因为：
- 调用外部天气API获取完整数据
- 数据量较小且结构化
- 不需要逐步生成内容
- 用户体验上一次性显示更合适

这种设计体现了**根据工具特性选择合适返回机制**的智能设计理念。

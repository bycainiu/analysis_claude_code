# Open-Claude-Code API 参考（中文）

本文件为 Open-Claude-Code 的公共 API、函数、组件与类型的完整中文参考说明，并为每个被引用的 API 附带源码出处与跳转链接，便于快速定位到项目中的定义位置。

- 仓库内相对链接指向定义所在文件，并附带 GitHub 风格的行号定位（例如 `#L10-L20`）。
- 若浏览器/查看器不支持行号定位，仍可点击进入相应文件，再按符号名搜索。

## 安装

```bash
# 例如（具体以你的包管理方式为准）
npm install open-claude-code
```

## 入口导出（src/index.ts）

以下符号均通过 `src/index.ts` 进行导出：

- 消息队列相关（定义见 `src/core/message-queue.ts`）
  - `h2AAsyncMessageQueue<T>`（[源码](./core/message-queue.ts#L514-L1052)）
  - `createMessageQueue<T>(options?)`（[源码](./core/message-queue.ts#L1061-L1070)）
  - `createHighPerformanceQueue<T>(options?)`（[源码](./core/message-queue.ts#L1075-L1086)）
  - `createMemoryEfficientQueue<T>(options?)`（[源码](./core/message-queue.ts#L1091-L1102)）
- 错误恢复（定义见 `src/core/error-recovery.ts`）
  - `AutoRecoveryManager`（[源码](./core/error-recovery.ts#L256-L528)）
  - `ErrorAnalyzer`（[源码](./core/error-recovery.ts#L570-L804)）
  - `defaultRecoveryManager`（[源码](./core/error-recovery.ts#L833-L837)）
  - `defaultErrorAnalyzer`（[源码](./core/error-recovery.ts#L839-L841)）
- 集成接口与适配器（定义见 `src/core/integrations.ts`）
  - `AgentCoreAdapter<T>`（[源码](./core/integrations.ts#L50-L295)）
  - `InputHandlerAdapter<T>`（[源码](./core/integrations.ts#L306-L511)）
  - `MonitoringAdapter`（[源码](./core/integrations.ts#L522-L874)）
  - `FlowControlManager`（[源码](./core/integrations.ts#L885-L1079)）
  - `IntegrationManager`（[源码](./core/integrations.ts#L1090-L1216)）
  - `createIntegratedEnvironment(options?)`（[源码](./core/integrations.ts#L1225-L1256)）
- 环境便捷工厂与工具（定义见 `src/index.ts`）
  - `createStandardEnvironment(options?)`（[源码](./index.ts#L124-L151)）
  - `createHighPerformanceEnvironment()`（[源码](./index.ts#L156-L162)）
  - `createMemoryEfficientEnvironment()`（[源码](./index.ts#L167-L173)）
  - `createStandardMessage(type, payload, options?)`（[源码](./index.ts#L182-L209)）
  - `delay(ms)`（[源码](./index.ts#L214-L216)）
  - `collectAsync(iterator, maxItems?)`（[源码](./index.ts#L221-L235)）
  - `formatMemoryUsage(bytes)`（[源码](./index.ts#L240-L251)）
  - `PerformanceTimer`（[源码](./index.ts#L256-L277)）
- 版本信息（定义见 `src/index.ts`）
  - `VERSION`（[源码](./index.ts#L108-L108)）
  - `BUILD_INFO`（[源码](./index.ts#L109-L115)）

> 类型与枚举定义集中在 `src/types/message-queue.ts`，详见下文“类型与枚举”。

---

## 核心：h2AAsyncMessageQueue<T>

基于双缓冲与 AsyncIterator 的高性能、非阻塞异步消息队列。

- 定义位置：`src/core/message-queue.ts`（[源码](./core/message-queue.ts#L514-L1052)）
- 构造参数：
  - `cleanupCallback?: () => void`
  - `maxBufferSize?: number`
  - `timeoutMs?: number`
  - `enableMetrics?: boolean`
  - `backpressureStrategy?: BackpressureStrategy`
  - `queueId?: string`
- 重要方法：
  - 生产：`enqueue(message: T)`, `enqueueBatch(messages: T[])`
  - 完成/错误：`done()`, `error(error: Error)`
  - 消费：`[Symbol.asyncIterator](): AsyncIterator<T>`, `next()`
  - 状态/指标：`getStatus()`, `getMetrics()`, `getDetailedMetrics()`
  - 生命周期：`onStateChange(key, cb)`, `offStateChange(key)`, `dispose()`
- 工厂方法（同文件内）：
  - `createMessageQueue<T>(options?)`（[源码](./core/message-queue.ts#L1061-L1070)）
  - `createHighPerformanceQueue<T>(options?)`（[源码](./core/message-queue.ts#L1075-L1086)）
  - `createMemoryEfficientQueue<T>(options?)`（[源码](./core/message-queue.ts#L1091-L1102)）

示例：基础生产-消费
```ts
import { createMessageQueue, BackpressureStrategy } from './index.js';

const queue = createMessageQueue<string>({
  enableMetrics: true,
  maxBufferSize: 1000,
  backpressureStrategy: BackpressureStrategy.DROP_OLDEST
});

queue.enqueue('hello');
queue.enqueue('world');
queue.done();

(async () => {
  for await (const item of queue) {
    console.log('接收:', item);
  }
  console.log('完成');
})();
```

示例：处理超时/错误
```ts
import { createMessageQueue } from './index.js';

const queue = createMessageQueue<number>({ timeoutMs: 500 });

(async () => {
  try {
    const { value } = await queue.next();
    console.log(value);
  } catch (err) {
    console.error('队列错误:', err);
  }
})();

queue.error(new Error('上游失败'));
```

---

## 集成（Integrations）

### AgentCoreAdapter<T>
- 定义位置：`src/core/integrations.ts`（[源码](./core/integrations.ts#L50-L295)）
- 职责：启动/停止消费队列消息，按消息类型派发事件，维护代理状态。
- 关键方法：`startConsuming()`, `stopConsuming()`, `consumeMessages()`, `getAgentStatus()`, `getQueueStatus()`, `getProcessingStats()`, `dispose()`
- 事件：`messageReceived`, `userInput`, `toolCall`, `toolResult`, `systemEvent`, `interrupt`, `messageProcessed`, `messageProcessingError`, `queueStateChange`, `consumptionError`

### InputHandlerAdapter<T>
- 定义位置：`src/core/integrations.ts`（[源码](./core/integrations.ts#L306-L511)）
- 职责：验证并发布单条/批量消息，支持批处理缓冲与自动刷新。
- 关键方法：`publishMessage()`, `publishBatch()`, `addToBatch()`, `flushBatch()`, `validateMessage()`, `setValidator()`, `getPublishingStats()`, `getBatchStatus()`, `configureBatch()`, `dispose()`

### MonitoringAdapter
- 定义位置：`src/core/integrations.ts`（[源码](./core/integrations.ts#L522-L874)）
- 职责：注册队列、导出指标快照、健康检查、触发告警、分析指标。
- 关键方法：`registerQueue()`, `unregisterQueue()`, `exportMetrics()`, `healthCheck()`, `triggerAlert()`, `getAllQueueStatus()`, `getMonitoringStats()`, `dispose()`

### FlowControlManager
- 定义位置：`src/core/integrations.ts`（[源码](./core/integrations.ts#L885-L1079)）
- 职责：实现 `FlowControlProtocol`，提供发送/接收/确认/拒绝与流控暂停恢复等能力。
- 关键方法：`send()`, `sendBatch()`, `receive()`, `acknowledge()`, `reject()`, `pause()`, `resume()`, `close()`, `getFlowControlStats()`

### IntegrationManager 与 createIntegratedEnvironment()
- 定义位置：`src/core/integrations.ts`（[IntegrationManager 源码](./core/integrations.ts#L1090-L1216)，[createIntegratedEnvironment 源码](./core/integrations.ts#L1225-L1256)）
- 职责：集中编排各适配器的一站式集成初始化与资源释放。

示例：端到端环境
```ts
import { createStandardEnvironment, createStandardMessage, MessageType } from './index.js';

const { queue, integrationManager, agentCore, inputHandler, monitoring, flowControl } =
  createStandardEnvironment({ enableMetrics: true });

agentCore?.on('userInput', (msg) => console.log('用户输入:', msg.payload));

inputHandler?.publishMessage(
  createStandardMessage(MessageType.USER_INPUT, { text: 'Hi' })
);

(async () => {
  const snapshot = await monitoring?.exportMetrics();
  console.log('metrics:', snapshot);
})();
```

---

## 错误恢复（Error Recovery）

- `AutoRecoveryManager`（[源码](./core/error-recovery.ts#L256-L528)）：
  - 主要方法：`handleError(error, queue)`, `detectErrorType(error)`, `registerStrategy()`, `unregisterStrategy()`, `getRecoveryHistory()`, `getRecoveryStats()`, `clearHistory()`
  - 可调参数：`setMaxRetryAttempts(n)`, `setBaseRetryDelay(ms)`, `setPreventionEnabled(bool)`
- `ErrorAnalyzer`（[源码](./core/error-recovery.ts#L570-L804)）：记录并分析错误、检测突发/模式、生成建议
- 默认实例：`defaultRecoveryManager`（[源码](./core/error-recovery.ts#L833-L837)）、`defaultErrorAnalyzer`（[源码](./core/error-recovery.ts#L839-L841)）

示例：与消费流程集成
```ts
import { defaultRecoveryManager, QueueError } from './index.js';

async function consumeWithRecovery(queue) {
  try {
    for await (const msg of queue) {
      // 处理消息
    }
  } catch (err) {
    if (err instanceof QueueError) {
      const ok = await defaultRecoveryManager.handleError(err, queue);
      if (!ok) throw err;
    } else {
      throw err;
    }
  }
}
```

---

## 类型与枚举（src/types/message-queue.ts）

- 枚举：
  - `BackpressureStrategy`（[源码](./types/message-queue.ts#L30-L35)）
  - `MessageType`（[源码](./types/message-queue.ts#L40-L47)）
  - `Priority`（[源码](./types/message-queue.ts#L52-L57)）
  - `QueueLifecycleState`（[源码](./types/message-queue.ts#L62-L71)）
  - `QueueErrorType`（[源码](./types/message-queue.ts#L76-L84)）
- 核心结构：
  - `StandardMessage`（[源码](./types/message-queue.ts#L145-L165)）
  - `QueueStatus`（[源码](./types/message-queue.ts#L201-L208)）
  - `QueueMetrics`（[源码](./types/message-queue.ts#L93-L100)）
  - `QueuePerformanceMetrics`（[源码](./types/message-queue.ts#L105-L136)）
- 状态与监听：
  - `StateTransition`（[源码](./types/message-queue.ts#L174-L179)）
  - `StateChange`（[源码](./types/message-queue.ts#L184-L188)）
  - `StateWatcher`（[源码](./types/message-queue.ts#L193-L196)）
  - `StateChangeCallback`（[源码](./types/message-queue.ts#L21-L21)）
- 错误恢复：
  - `QueueError`（[源码](./types/message-queue.ts#L515-L528)）
  - `ErrorRecoveryStrategy`（[源码](./types/message-queue.ts#L235-L244)）
  - `RecoveryContext`（[源码](./types/message-queue.ts#L217-L221)）
  - `RecoveryResult`（[源码](./types/message-queue.ts#L226-L230)）
- 配置：
  - `QueueConfiguration`（[源码](./types/message-queue.ts#L271-L302)）
  - `AdvancedQueueConfiguration`（[源码](./types/message-queue.ts#L307-L328)）
  - `BatchConfiguration`（[源码](./types/message-queue.ts#L253-L256)）
  - `AlertThresholds`（[源码](./types/message-queue.ts#L261-L266)）
- 集成契约：
  - `SendResult`（[源码](./types/message-queue.ts#L373-L378)）
  - `BatchSendResult`（[源码](./types/message-queue.ts#L383-L388)）
  - `HealthStatus`（[源码](./types/message-queue.ts#L393-L397)）
  - `Alert`（[源码](./types/message-queue.ts#L402-L408)）
  - `MetricsSnapshot`（[源码](./types/message-queue.ts#L413-L417)）
  - `ValidationResult`（[源码](./types/message-queue.ts#L422-L426)）
  - `AgentStatus`（[源码](./types/message-queue.ts#L431-L435)）
  - `FlowControlProtocol`（[源码](./types/message-queue.ts#L440-L460)）
  - `AgentCoreIntegration<T>`（[源码](./types/message-queue.ts#L469-L478)）
  - `InputHandlerIntegration<T>`（[源码](./types/message-queue.ts#L483-L492)）
  - `MonitoringIntegration`（[源码](./types/message-queue.ts#L497-L506)）
- 默认配置：
  - `DEFAULT_QUEUE_CONFIG`（[源码](./types/message-queue.ts#L537-L561)）
  - `DEFAULT_ADVANCED_CONFIG`（[源码](./types/message-queue.ts#L566-L587)）

示例：构造标准消息
```ts
import { createStandardMessage, MessageType, Priority } from './index.js';

const msg = createStandardMessage(
  MessageType.USER_INPUT,
  { text: 'Hello' },
  { priority: Priority.HIGH, source: 'cli', ttl: 10000 }
);
```

---

## 实用工具（src/index.ts）

- `createStandardMessage(type, payload, options?)`（[源码](./index.ts#L182-L209)）
- `delay(ms)`（[源码](./index.ts#L214-L216)）
- `collectAsync(asyncIterable, maxItems?)`（[源码](./index.ts#L221-L235)）
- `formatMemoryUsage(bytes)`（[源码](./index.ts#L240-L251)）
- `PerformanceTimer`（[源码](./index.ts#L256-L277)）

---

## 环境工厂（src/index.ts）

- `createStandardEnvironment(options?)`（[源码](./index.ts#L124-L151)）：一次性创建队列与集成管理器，默认启用 AgentCore/InputHandler/Monitoring/FlowControl。
- `createHighPerformanceEnvironment()`（[源码](./index.ts#L156-L162)）：性能优先的标准环境预设。
- `createMemoryEfficientEnvironment()`（[源码](./index.ts#L167-L173)）：内存友好的标准环境预设。

---

## 配置默认值（src/types/message-queue.ts）

- `DEFAULT_QUEUE_CONFIG`（[源码](./types/message-queue.ts#L537-L561)）：缓冲区、超时、性能与错误处理的默认值。
- `DEFAULT_ADVANCED_CONFIG`（[源码](./types/message-queue.ts#L566-L587)）：背压、内存、监控/告警阈值的默认值。

示例：按需覆盖默认配置
```ts
import { createMessageQueue, BackpressureStrategy, DEFAULT_QUEUE_CONFIG } from './index.js';

const queue = createMessageQueue({
  maxBufferSize: DEFAULT_QUEUE_CONFIG.buffer.maxSize * 2,
  timeoutMs: 250,
  enableMetrics: true,
  backpressureStrategy: BackpressureStrategy.ERROR
});
```

---

## 版本与构建信息（src/index.ts）

- `VERSION`（[源码](./index.ts#L108-L108)）
- `BUILD_INFO`（[源码](./index.ts#L109-L115)）
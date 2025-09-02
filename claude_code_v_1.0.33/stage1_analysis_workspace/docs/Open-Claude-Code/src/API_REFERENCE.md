# Open-Claude-Code API Reference

This document provides a comprehensive reference for all public APIs exported by the Open-Claude-Code message queue and integrations. It includes classes, functions, enums, interfaces, configuration objects, and example usage snippets.

## Installation

```bash
# Using npm (example)
npm install open-claude-code
```

## Entry Point Exports

All core exports are re-exported from `src/index.ts`.

- Message queue:
  - `h2AAsyncMessageQueue<T>`
  - `createMessageQueue<T>(options?)`
  - `createHighPerformanceQueue<T>(options?)`
  - `createMemoryEfficientQueue<T>(options?)`
- Error recovery:
  - `AutoRecoveryManager`
  - `ErrorAnalyzer`
  - `defaultRecoveryManager`
  - `defaultErrorAnalyzer`
- Integrations:
  - `AgentCoreAdapter<T>`
  - `InputHandlerAdapter<T>`
  - `MonitoringAdapter`
  - `FlowControlManager`
  - `IntegrationManager`
  - `createIntegratedEnvironment(options?)`
- Utilities and env factories:
  - `createStandardEnvironment(options?)`
  - `createHighPerformanceEnvironment()`
  - `createMemoryEfficientEnvironment()`
  - `createStandardMessage(type, payload, options?)`
  - `delay(ms)`
  - `collectAsync(iterator, maxItems?)`
  - `formatMemoryUsage(bytes)`
  - `PerformanceTimer`
- Version info:
  - `VERSION`
  - `BUILD_INFO`

Types and enums are exported from `src/types/message-queue.ts` (see below).

---

## Core: h2AAsyncMessageQueue<T>

Class implementing a dual-buffer, non-blocking async message queue with AsyncIterator support.

Constructor options:
- `cleanupCallback?: () => void`
- `maxBufferSize?: number`
- `timeoutMs?: number`
- `enableMetrics?: boolean`
- `backpressureStrategy?: BackpressureStrategy`
- `queueId?: string`

Public methods:
- `enqueue(message: T): void`
- `enqueueBatch(messages: T[]): void`
- `done(): void` — signal completion
- `error(error: Error): void` — propagate error and transition to error state
- `[Symbol.asyncIterator](): AsyncIterator<T>` — start iteration (single-consumer)
- `next(): Promise<IteratorResult<T>>` — AsyncIterator method
- Status/metrics:
  - `getStatus(): QueueStatus`
  - `getMetrics(): QueueMetrics`
  - `getDetailedMetrics(): QueuePerformanceMetrics`
  - `getQueueId(): string`
  - `getCreationTime(): number`
  - `getUptime(): number`
  - `getDebugInfo(): any`
- Lifecycle:
  - `onStateChange(key: string, cb: StateChangeCallback): void`
  - `offStateChange(key: string): void`
  - `dispose(): void`

Factory functions:
- `createMessageQueue<T>(options?): h2AAsyncMessageQueue<T>`
- `createHighPerformanceQueue<T>(options?): h2AAsyncMessageQueue<T>`
- `createMemoryEfficientQueue<T>(options?): h2AAsyncMessageQueue<T>`

Example: basic producer-consumer
```ts
import {
  createMessageQueue,
  BackpressureStrategy
} from './index.js';

const queue = createMessageQueue<string>({
  enableMetrics: true,
  maxBufferSize: 1000,
  backpressureStrategy: BackpressureStrategy.DROP_OLDEST
});

// Producer
queue.enqueue('hello');
queue.enqueue('world');
queue.done();

// Consumer
(async () => {
  for await (const item of queue) {
    console.log('received:', item);
  }
  console.log('done');
})();
```

Example: handling errors and timeouts
```ts
import { createMessageQueue, QueueErrorType } from './index.js';

const queue = createMessageQueue<number>({ timeoutMs: 500 });

(async () => {
  try {
    const { value } = await queue.next(); // may throw QueueError with TIMEOUT_ERROR
    console.log(value);
  } catch (err) {
    // handle queue-specific errors
    console.error('queue error:', err);
  }
})();

queue.error(new Error('upstream failure'));
```

---

## Types and Enums

From `src/types/message-queue.ts`.

- Enums: `BackpressureStrategy`, `MessageType`, `Priority`, `QueueLifecycleState`, `QueueErrorType`
- Core message and status types: `StandardMessage`, `QueueStatus`, `QueueMetrics`, `QueuePerformanceMetrics`
- Error handling: `QueueError`, `ErrorRecoveryStrategy`, `RecoveryContext`, `RecoveryResult`
- State: `StateTransition`, `StateChange`, `StateWatcher`, `StateChangeCallback`
- Config: `QueueConfiguration`, `AdvancedQueueConfiguration`, `BatchConfiguration`, `AlertThresholds`
- Integrations: `SendResult`, `BatchSendResult`, `HealthStatus`, `Alert`, `MetricsSnapshot`, `ValidationResult`, `AgentStatus`, `FlowControlProtocol`, `AgentCoreIntegration<T>`, `InputHandlerIntegration<T>`, `MonitoringIntegration`
- Defaults: `DEFAULT_QUEUE_CONFIG`, `DEFAULT_ADVANCED_CONFIG`

Example: constructing a `StandardMessage`
```ts
import { createStandardMessage, MessageType, Priority } from './index.js';

const msg = createStandardMessage(
  MessageType.USER_INPUT,
  { text: 'Hello' },
  { priority: Priority.HIGH, source: 'cli', ttl: 10000 }
);
```

---

## Error Recovery

- `AutoRecoveryManager`
  - Configure strategies, handle recoverable `QueueError`s with retry/backoff.
  - Key methods:
    - `handleError(error: QueueError, queue: h2AAsyncMessageQueue<any>): Promise<boolean>`
    - `detectErrorType(error: Error): QueueErrorType`
    - `registerStrategy(type, strategy)`, `unregisterStrategy(type)`
    - `getRecoveryHistory()`, `getRecoveryStats()`, `clearHistory()`
    - Tunables: `setMaxRetryAttempts(n)`, `setBaseRetryDelay(ms)`, `setPreventionEnabled(bool)`
- `ErrorAnalyzer`
  - Record and analyze errors, detect bursts/patterns, produce recommendations.
  - Methods: `recordError(error, ctx?)`, `analyze(timeWindow?)`, `getAnalysisHistory()`, `clearHistory()`
- Defaults: `defaultRecoveryManager`, `defaultErrorAnalyzer`

Example: integrate automatic recovery
```ts
import { defaultRecoveryManager, QueueError } from './index.js';

async function consumeWithRecovery(queue) {
  try {
    for await (const msg of queue) {
      // process msg
    }
  } catch (err) {
    if (err instanceof QueueError) {
      const recovered = await defaultRecoveryManager.handleError(err, queue);
      if (!recovered) throw err;
    } else {
      throw err;
    }
  }
}
```

---

## Integrations

### AgentCoreAdapter<T>
- Starts/stops consuming from a queue, emits events for different message types, tracks status.
- Constructor options: `{ messageQueue?, agentId?, enableHighPerformance? }`
- Key methods: `startConsuming()`, `stopConsuming()`, `consumeMessages()`, `getAgentStatus()`, `getQueueStatus()`, `getProcessingStats()`, `dispose()`
- Events: `messageReceived`, `userInput`, `toolCall`, `toolResult`, `systemEvent`, `interrupt`, `messageProcessed`, `messageProcessingError`, `queueStateChange`, `consumptionError`

### InputHandlerAdapter<T>
- Validates and publishes single or batched messages into a queue with optional batching.
- Constructor options: `{ messageQueue?, validator?, batchConfig? }`
- Key methods: `publishMessage()`, `publishBatch()`, `addToBatch()`, `flushBatch()`, `validateMessage()`, `setValidator()`, `getPublishingStats()`, `getBatchStatus()`, `configureBatch()`, `dispose()`
- Events: `messagePublished`, `batchPublished`, `validationError`, `validationWarning`, `publishError`

### MonitoringAdapter
- Registers queues, exports metrics snapshots, health checks, alerts, and analyzes metrics.
- Options: `{ healthCheckIntervalMs?, metricsCollectionIntervalMs?, alertThresholds? }`
- Key methods: `registerQueue(id, queue)`, `unregisterQueue(id)`, `exportMetrics()`, `healthCheck()`, `triggerAlert(alert)`, `getAllQueueStatus()`, `getMonitoringStats()`, `dispose()`
- Events: `queueRegistered`, `queueUnregistered`, `metricsExported`, `alertTriggered`, `criticalAlert`, `healthCheckCompleted`, `queueStateChanged`

### FlowControlManager
- Implements `FlowControlProtocol` for send/receive/ack with basic flow control.
- Options: `{ messageQueue?, ackTimeout? }`
- Methods: `send()`, `sendBatch()`, `receive()`, `acknowledge()`, `reject()`, `pause()`, `resume()`, `close()`, `getFlowControlStats()`

### IntegrationManager and createIntegratedEnvironment()
- Centralized orchestration for all adapters.
- Methods: `initializeAgentCore()`, `initializeInputHandler()`, `initializeMonitoring()`, `initializeFlowControl()`, getters for each adapter, `dispose()`
- `createIntegratedEnvironment(options?)` creates a ready-to-use environment sharing a high-performance queue.

Example: end-to-end integrated environment
```ts
import {
  createStandardEnvironment,
  createStandardMessage,
  MessageType
} from './index.js';

const { queue, integrationManager, agentCore, inputHandler, monitoring, flowControl } =
  createStandardEnvironment({ enableMetrics: true });

// Listen for agent events
agentCore?.on('userInput', (msg) => console.log('User input:', msg.payload));

// Publish a message
inputHandler?.publishMessage(
  createStandardMessage(MessageType.USER_INPUT, { text: 'Hi' })
);

// Export metrics
(async () => {
  const snapshot = await monitoring?.exportMetrics();
  console.log('metrics:', snapshot);
})();
```

---

## Utilities

- `createStandardMessage(type, payload, options?)`
- `delay(ms)`
- `collectAsync(asyncIterable, maxItems?)`
- `formatMemoryUsage(bytes)`
- `PerformanceTimer` with `start()`, `stop()`, `getDuration()`, `reset()`

---

## Configuration Defaults

- `DEFAULT_QUEUE_CONFIG` and `DEFAULT_ADVANCED_CONFIG` provide sane defaults for buffer sizes, timeouts, performance flags, error handling, backpressure, monitoring thresholds, and memory limits. Override selectively when constructing queues or environments.

Example: custom configuration
```ts
import {
  createMessageQueue,
  BackpressureStrategy,
  DEFAULT_QUEUE_CONFIG,
  DEFAULT_ADVANCED_CONFIG
} from './index.js';

const queue = createMessageQueue({
  maxBufferSize: DEFAULT_QUEUE_CONFIG.buffer.maxSize * 2,
  timeoutMs: 250,
  enableMetrics: true,
  backpressureStrategy: BackpressureStrategy.ERROR
});
```

---

## Version and Build Info

- `VERSION`: semantic version string
- `BUILD_INFO`: `{ version, buildTime, nodeVersion, architecture, platform }`
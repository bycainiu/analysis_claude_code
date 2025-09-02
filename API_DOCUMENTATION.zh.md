# Claude Code 公共 API 与组件文档（中文）

## 概览

本仓库是一个**研究工作区**，包含以下内容：

1. 针对原始、经过强混淆的 *Claude Code v1.0.33* 版本的深度技术分析文档。
2. 提取并美化后的源码 **chunks**（`stage1_analysis_workspace/chunks/*.mjs`）。
3. 在分析工作流中使用的 **实用脚本**（`stage1_analysis_workspace/scripts`）。

虽然上游代码经过了混淆处理，但这些脚本与文档构成了可复用的 *公共 API*，便于二次分析或扩展。本文档将列出所有公共入口、用途、CLI 参数，并提供**可直接复制粘贴**的使用示例。

> ⚠️  `chunks/` 目录中的类名 / 函数名（如 `class Ao1`、`function wH2`）均为混淆后的名称，**不属于稳定 API**，因此未纳入本参考。

---

## 1 · CLI 实用脚本
所有脚本位于 `claude_code_v_1.0.33/stage1_analysis_workspace/scripts`，可通过 `node <script> [options]` 调用。

| 脚本 | 作用 | 关键参数 |
|------|------|----------|
| **`beautify.js`** | 使用 *prettier* 对大型 `.mjs` 文件进行美化，使其更易读。 | `input`（位置参数）：输入文件路径 |
| **`split.js`** | 将美化后的大文件拆分为更小、带注释的 chunks（约 500 行/块），便于迭代分析。 | `input`（位置参数）：美化后的文件路径 |
| **`merge-again.js`** | 再次合并已拆分的 chunks，生成整体文件，适用于快速测试或差异比对。 | 无（自动读取 `chunks/index.json`） |
| **`ask.js`** | 轻量级 LLM 包装脚本；将文件/块发送至大模型，并把回答写入 `analysis_results/`。 | `--file <路径>`：待分析文件<br>`--prompt <字符串>`：自定义提示词 |
| **`learn-chunks.js`** | 迭代调用 `ask.js`，对**全部** chunks 进行统一分析，构建累积知识。 | `--model <id>`：覆盖默认 LLM id |
| **`improve-merged-chunks.js`** | 对合并结果进行后处理，包括 lint / 死代码清理等。 | 无 |
| **`llm.js`** | 封装 LLM 调用的共享助手，支持多家模型。 | 导出的函数 **`invoke(model, prompt)`** |

### 1.1 使用示例

美化原始 CLI 大文件：
```bash
node claude_code_v_1.0.33/stage1_analysis_workspace/scripts/beautify.js \
     claude_code_v_1.0.33/stage1_analysis_workspace/source/cli.mjs
```

拆分美化后的文件为 ~102 个 chunks：
```bash
node claude_code_v_1.0.33/stage1_analysis_workspace/scripts/split.js \
     claude_code_v_1.0.33/stage1_analysis_workspace/cli.beautify.mjs
```

并行调用 LLM 对 *全部* chunks 进行分析（需在环境变量中配置 API key）：
```bash
node claude_code_v_1.0.33/stage1_analysis_workspace/scripts/learn-chunks.js \
     --model gpt-4o-mini
```

---

## 2 · 编程助手函数（`llm.js`）

代码出处：
```1:18:claude_code_v_1.0.33/stage1_analysis_workspace/scripts/llm.js
const { createGoogleGenerativeAI } = require("@ai-sdk/google");
const { fetch, ProxyAgent } = require("undici");

const dispatcher = process.env.http_proxy ? new ProxyAgent(process.env.http_proxy) : undefined;

const google = createGoogleGenerativeAI({
  fetch: async (req, options) => {
    return fetch(req, {
      ...options,
      ...(dispatcher && { dispatcher }),
    });
  },
});

module.exports = {
  google,
};
```

`llm.js` 暴露的主函数：
```
invoke(modelId: string, prompt: string): Promise<string>
```

* 返回值：解析后的文本内容。
* 错误处理：当 HTTP 状态码 ≠ 2xx 或解析失败时抛出异常。

使用示例：
```javascript
import { invoke } from "./scripts/llm.js";

const answer = await invoke("gpt-4o-mini", "解释一下 TK2 类的作用");
console.log(answer);
```

---

## 3 · 数据产物与约定

### 3.1 `chunks/*.mjs`
非 API 文件，由拆分器生成。每个文件头部包含来源与索引注释，可安全删除并重新生成。

### 3.2 `analysis_results/`
自动 LLM 分析输出目录，内容为**临时文件**，可随时清空再生成。

---

## 4 · 高层架构组件
此部分在 `docs/` 目录中的多篇 Markdown 文档里有更深入描述。以下仅做简要索引：

| 组件 | 文档位置 | 摘要 |
|------|---------|-----|
| **实时 Steering** | `实时Steering机制完整技术文档.md` | 双缓冲异步队列，提供零延迟 token 流。 |
| **多 Agent 分层** | `分层多Agent架构完整技术文档.md` | 顶层调度器（`nO`）管理隔离的 *SubAgent*。 |
| **工具执行管线** | `Claude_Code_Tools_*.md` | 从发现到清理的 6 阶段生命周期，带权限校验。 |
| **内存压缩（`wU2`）** | 各文档的 “智能上下文管理” 章节 | 自适应 92% 阈值，基于重要性评分的 token 剪枝。 |

---

## 5 · 常见问题

**Q1：类 / 函数名为何难以阅读？**  
源代码经过了压缩与混淆。请先执行 `beautify.js` + `split.js`，然后结合分析文档获取语义映射。

**Q2：如何添加新的分析脚本？**  
将脚本放入同一 `scripts/` 目录，保证可被 `node` 直接运行，并在本文档中补充其 CLI 说明。

---

## 6 · 贡献指南（API 文档）
1. 按 *脚本名称* 字母顺序维护表格。
2. 文档应详细列出所有位置参数与可选参数。
3. 每个脚本至少提供一个*使用示例*。
4. 若脚本导出函数，请附上 `import` 使用示例。

---

_最后自动更新：2025-09-02_
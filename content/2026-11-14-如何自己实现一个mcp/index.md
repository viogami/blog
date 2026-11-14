---
title: 如何自己实现一个mcp
slug: ""
date: 2026-11-14T01:59:40.889Z
preview: ""
draft: false
description: "2026-07-28 版 MCP 已经取消握手与会话，变成无状态协议。本文按最新规范梳理选型、最少代码起步、不用 SDK 手写 JSON-RPC、工具设计、鉴权与调试的完整流程。"
tags: ["MCP", "AI Agent", "TypeScript", "Python"]
categories: ["AI"]
---

## 0. 先做减法：你真的需要自己写吗

得到 MCP 服务器有三条路，差别在于「你拥有多少代码」：

| 路径 | 做法 | 适合 |
| --- | --- | --- |
| 手写（官方 SDK） | 逐个工具写注册代码 | 工具要串多个接口、读本地文件、做业务规则校验 |
| 从 OpenAPI 生成 | 用生成器/gateway 把已有 OpenAPI 暴露出去 | 工具全是现有 HTTP 接口的薄封装 |
| 用现成/托管 server | 官方或厂商提供的 server | GitHub、AWS、数据库这类通用场景 |

判断标准很简单：**如果每个工具都只是 `GET /foo` 的转发，别手写**；如果工具需要「先查订单、再查物流、再算赔付」这种编排，才值得手写。

## 1. 2026 年的 MCP 已经变了

网上大量教程还停留在 2024/2025 的写法（`initialize` 握手 + HTTP+SSE + `from mcp.server.fastmcp import FastMCP`），照抄会直接踩坑。当前规范是 **`2026-07-28`**，核心变化是**协议无状态化**：

- **取消握手与会话**：`initialize` / `notifications/initialized` 退场，`Mcp-Session-Id` 头也一并移除。每个请求自带完整信封，协议版本、客户端身份、客户端能力都放在 `_meta` 里（`io.modelcontextprotocol/protocolVersion`、`clientInfo`、`clientCapabilities`）。
- **新增 `server/discover`**：客户端想先知道服务端能力时调它，但**不是必需**的。
- **可水平扩展**：任何请求都能落到任意实例后面，普通 round-robin 负载均衡即可，不需要共享 session 存储。
- **HTTP 头上带方法名**：`Mcp-Method`、`Mcp-Name` 让网关直接按头路由和鉴权，不必解析 body。
- **服务端发起的请求改为 MRTR**（Multi Round-Trip Requests）：sampling、elicitation 不再依赖常开的双向流。
- **列表响应带缓存与确定性排序**：`ttlMs`、`cacheScope` 让客户端能缓存工具清单，也稳定上游 prompt cache。
- **授权硬化**：RFC 9207 issuer 校验，DCR 让位给 CIMD（client metadata documents）。
- **正式弃用策略**：至少 12 个月窗口，升级可规划。

所以：**你写的 server 只要兼容 `2026-07-28`，一个实例就能同时服务新旧客户端**（v2 SDK 默认如此）。

## 2. 谁决定用它？

| 原语 | 谁控制 | 是什么 | 类比 |
| --- | --- | --- | --- |
| Tools | 模型 | 模型调用以执行动作的函数 | `POST` |
| Resources | 应用 | 宿主加载进上下文的只读数据 | `GET` |
| Prompts | 用户 | 用户按名字触发的模板 | 保存的查询 / 斜杠命令 |

初学只需要 Tools。Resources/Prompts 是有明确需求时再加，不要一上来三件套全做。

## 3. 选型：官方 SDK 还是 FastMCP

| 语言 | 包 | 说明 |
| --- | --- | --- |
| TypeScript | `@modelcontextprotocol/server` + `/client`（v2，2.3.x） | 新项目起点；旧的 `@modelcontextprotocol/sdk` 是 v1，只维护 |
| Python | `mcp`（2.3.x） | 高层类已改名 `MCPServer`，装饰器 API 不变 |
| FastMCP | `pip install fastmcp`（独立包，Prefect） | 建立在官方 SDK 之上的工程化框架：鉴权 provider、server 组合、测试客户端、部署工具 |
| Go | `github.com/modelcontextprotocol/go-sdk` | 云原生服务首选 |
| C# | `ModelContextProtocol` | .NET / Semantic Kernel 体系 |
| Rust | `rmcp` | 极致资源占用 |
| Java | `io.modelcontextprotocol.sdk:mcp` | Spring AI 体系 |
| 已有 FastAPI | `fastapi_mcp` | 直接复用现有鉴权依赖和 Swagger，把接口变成工具 |

选择建议：

- **Python/TS 且不止一个玩具工具** → 官方 SDK 就够，FastMCP 在需要 OAuth、多 server 组合、部署脚手架时再上。
- **已有成熟 REST 服务** → `fastapi_mcp` 或 OpenAPI 生成器，别重写。
- **服务跑在 K8s、要和现有网关共用鉴权** → Go SDK + Streamable HTTP。

## 4. 起步：TypeScript 版（约 25 行）

```bash
npm i @modelcontextprotocol/server zod
```

```json
{ "type": "module" }
```

```ts
// src/server.ts
import { McpServer } from '@modelcontextprotocol/server';
import { serveStdio } from '@modelcontextprotocol/server/stdio';
import * as z from 'zod/v4';

serveStdio(() => {
  const server = new McpServer({ name: 'weather', version: '0.1.0' });

  server.registerTool(
    'get_weather',
    {
      title: '查询天气',
      description: '查询某个城市当前的天气。当用户问「今天冷不冷」「要不要带伞」时使用。',
      inputSchema: z.object({
        city: z.string().describe('城市名，例如「杭州」')
      })
    },
    async ({ city }) => ({
      content: [{ type: 'text', text: `${city}：18°C，多云` }]
    })
  );

  return server;
});
```

要点：

- `serveStdio(() => server)` 负责把服务跑在 stdio 上；Node 24 可直接 `node src/server.ts`（类型剥离），否则用 `tsx` 或先编译。
- `inputSchema` 用 zod v4（也支持 Valibot / ArkType 等 Standard Schema 实现），**SDK 会在进入你的 handler 之前完成校验**，参数不合法直接返回协议错误。
- 想在 handler 里拿请求上下文（客户端信息、进度、日志），注册回调里带 `extra`/`ctx` 参数，日志一律写 `stderr`。

## 5. 起步：Python 版（12 行）

```bash
uv add "mcp[cli]"
```

```python
from mcp.server import MCPServer

mcp = MCPServer("weather")

@mcp.tool()
def get_weather(city: str) -> str:
    """查询某个城市当前的天气。当用户问今天冷不冷、要不要带伞时使用。

    Args:
        city: 城市名，例如「杭州」。
    """
    return f"{city}：18°C，多云"

if __name__ == "__main__":
    mcp.run()  # stdio；mcp.run("streamable-http") 起 HTTP，默认 127.0.0.1:8000/mcp
```

要点：

- Python SDK 2.x 里 `FastMCP` 已改名为 `MCPServer`，导入路径是 `from mcp.server import MCPServer`；`mcp.server.fastmcp.*` 下的老路径会直接抛 `ModuleNotFoundError`。
- **没有 JSON Schema、没有参数解析、没有序列化代码**：类型注解就是 schema，docstring 就是工具描述。
- 传输相关参数（host、port、`stateless_http`、endpoint 路径）在 v2 里都挪到了 `run()`，构造器不再接收。

## 6. 进阶：完全不用 SDK，手写协议核心

想理解协议，或者要在边缘设备、无运行时依赖的网关里跑，可以只实现自己需要的那几个方法。

**stdio 传输的约定**：每行一个 JSON-RPC 2.0 消息（NDJSON），**不要**写 `Content-Length` 头（那是 LSP 的规矩）。`stdout` 只允许出现协议消息，任何 `console.log` 调试都会污染流——只用 `stderr`。

一次 `tools/call` 的完整信封（2026-07-28）：

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"get_weather","arguments":{"city":"杭州"},
 "_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28",
          "io.modelcontextprotocol/clientInfo":{"name":"my-app","version":"1.0.0"},
          "io.modelcontextprotocol/clientCapabilities":{}}}}
```

响应：

```json
{"jsonrpc":"2.0","id":1,
 "result":{"resultType":"complete","isError":false,
 "content":[{"type":"text","text":"杭州：18°C，多云"}],
 "_meta":{"io.modelcontextprotocol/serverInfo":{"name":"mini-weather","version":"0.1.0"}}}}
```

最小可用 server 只需实现：`server/discover`（可选但强烈建议）、`tools/list`、`tools/call`。Node 版骨架：

```js
// minimal-server.mjs —— 零依赖，直接 node minimal-server.mjs
import { createInterface } from 'node:readline';

const PROTOCOL = '2026-07-28';
const send = (m) => process.stdout.write(JSON.stringify(m) + '\n');
const ok = (id, result) => ({ jsonrpc: '2.0', id, result });
const fail = (id, code, message) => ({ jsonrpc: '2.0', id, error: { code, message } });

const tools = [{
  name: 'get_weather',
  title: '查询天气',
  description: '查询某个城市当前的天气。当用户问今天冷不冷、要不要带伞时使用。',
  inputSchema: {
    type: 'object',
    properties: { city: { type: 'string', description: '城市名，例如「杭州」' } },
    required: ['city'],
    additionalProperties: false
  }
}];

const handlers = {
  async get_weather({ city }) {
    if (!city) throw new Error('city 不能为空');
    return { content: [{ type: 'text', text: `${city}：18°C，多云` }] };
  }
};

createInterface({ input: process.stdin, crlfDelay: Infinity }).on('line', async (line) => {
  if (!line.trim()) return;
  let msg;
  try { msg = JSON.parse(line); } catch {
    return send(fail(null, -32700, 'Parse error'));
  }
  const { id, method, params = {} } = msg;
  if (id === undefined) return; // 通知不需要回包

  switch (method) {
    case 'server/discover':
      return send(ok(id, {
        protocolVersions: [PROTOCOL],
        capabilities: { tools: { listChanged: false } },
        serverInfo: { name: 'mini-weather', version: '0.1.0' }
      }));
    case 'tools/list':
      return send(ok(id, { resultType: 'complete', tools }));
    case 'tools/call': {
      const fn = handlers[params.name];
      if (!fn) return send(fail(id, -32602, `Unknown tool: ${params.name}`));
      try {
        const result = await fn(params.arguments ?? {});
        return send(ok(id, { resultType: 'complete', isError: false, ...result }));
      } catch (e) {
        // 工具执行失败 ≠ 协议失败：仍然返回 result，用 isError 表达
        return send(ok(id, {
          resultType: 'complete', isError: true,
          content: [{ type: 'text', text: String(e.message ?? e) }]
        }));
      }
    }
    default:
      return send(fail(id, -32601, `Method not found: ${method}`));
  }
});
```

手写时最容易错的四件事：

1. **两层错误不要混**。协议/参数级问题用 JSON-RPC `error`（`-32700` 解析失败、`-32600` 非法请求、`-32601` 方法不存在、`-32602` 非法参数）；**工具业务失败必须是 `result` + `isError: true`**，否则模型看到的是「工具坏了」而不是「这次调用失败了，可以换个参数再试」。另外注意：2026-07-28 把「资源不存在」的错误码从 `-32002` 统一成了 `-32602`，`-32002` 已成历史。
2. **`_meta` 是必填的**，不是可选装饰。缺 `protocolVersion` 的请求应当被拒绝（`UnsupportedProtocolVersionError`）。
3. **列表要确定性排序**。`tools/list` 每次返回顺序一致，客户端才能缓存、上游 prompt cache 才不会次次失效。
4. **无参数工具也要给合法 schema**：`{"type":"object","additionalProperties":false}`，不要留 `null`。

## 7. 传输与部署

**stdio（本地）**：宿主 spawn 你的进程，通过 `stdin`/`stdout` 通信。最简单、最好调、最稳，绝大多数桌面场景应该用它。

**Streamable HTTP（远程）**：单一端点（如 `POST /mcp`）收发 JSON-RPC，SDK 提供 Express / Hono / Fastify / Workers 的中间件适配。

部署要点：

- 无状态之后，**不要自己发明 session**。跨调用需要的状态，由服务端签发一个显式 handle（如 `report_id`），当作普通工具参数来回传。
- 网关可以只读 `Mcp-Method` / `Mcp-Name` 头做路由与限流，不必解析 JSON body。
- 冷启动敏感的场景，把工具清单预生成成静态数组，别在请求路径里做重 IO。
- 记得处理取消（`notifications/cancelled`）与进度（`progressToken`），长任务尤其重要。

## 8. 工具设计：决定 agent 用得好不好

工具定义会进入每一次调用的上下文窗口，所以它既是接口文档，也是 prompt。

**命名**：动词在前、名词在后、snake_case —— `search_incidents`、`get_deployment`、`create_ticket`。长度 1~128，区分大小写，同一 server 内唯一，只允许 ASCII 字母数字与 `_ - .`。多个 server 并存时加服务前缀（`pagerduty_search_incidents`），因为不同 server 的 `search` 会撞名。给人看的中文名放 `title`，`name` 保持机器形状。

**description 写给一个没见过你系统的新同事**，并且明确「什么时候该用 / 不该用」。模型对描述里的否定条件非常敏感。

**参数**：能枚举就枚举（`Literal` / `z.enum`），给默认值，每个字段写 `description`。避免 `oneOf` 套娃——模型填不对。

**输出**：有结构就声明 `outputSchema` 并返回 `structuredContent`；纯文本只用于给人看的摘要。

**行为注解**：`readOnlyHint`、`destructiveHint`、`idempotentHint`、`openWorldHint`。破坏性操作默认真人确认，别奢望模型替你判断风险。

**返回体积**：几万 token 的工具结果会直接挤爆上下文。做法是「摘要 + 句柄 + 分页 cursor」，让模型按需翻页。

**幂等**：同一个 `idempotency_key` 重复调用只应产生一次副作用——重试是常态，不是异常。

## 9. 鉴权与安全

- 远程 server **必须** OAuth 2.1 + PKCE，token 里绑定 resource indicator。
- 2026 年起：DCR 让位给 **CIMD**（client metadata document），并强制 RFC 9207 `iss` 校验，防 mix-up 攻击。
- `tools/list` 可以按调用者 scope 返回不同子集（凭据是每请求输入，不是连接状态），善用这点做最小权限。
- token 永远不要出现在工具参数或资源 URI 里。
- 本地 stdio 也不是安全区：能读全盘文件系统的 server，等于把整台机器的权限交给模型，路径白名单要写死。
- 工具返回的内容（网页、文件、issue 正文）是**不可信数据**，里面可能藏着针对模型的注入指令。别在工具结果里拼接任何可执行建议。

## 10. 调试与联调

```bash
# 通用 Inspector（Node 环境即可）
npx @modelcontextprotocol/inspector node dist/server.js
npx @modelcontextprotocol/inspector uv run python server.py

# Python SDK 自带的快捷方式，会起 server 并打开 Inspector
uv run mcp dev server.py

# 独立 FastMCP 框架用自己的 CLI
uv run fastmcp dev inspector server.py
```

手工验证 stdio 也可以：

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientInfo":{"name":"cli","version":"0"},"io.modelcontextprotocol/clientCapabilities":{}}}}' | node minimal-server.mjs
```

再进一步：用 SDK 的 `Client` 写集成测试，直接拿内存里的 server 对象当目标，**不走传输**，这样 CI 里跑得又快又稳。

## 11. 接到宿主里

本地 stdio server，所有宿主读的都是同一个形状：

```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["E:\\project\\weather-mcp\\dist\\server.js"],
      "env": { "WEATHER_API_KEY": "..." }
    }
  }
}
```

- Claude Desktop：`%APPDATA%\Claude\claude_desktop_config.json`（Windows），改完**完全退出再开**。
- Claude Code：项目级 `.mcp.json`、用户级 `~/.claude.json`。
- 远程 server：Claude Desktop 走 Settings → Connectors 添加 URL；它的配置文件只解析 stdio 形状，硬塞 `url` 键会被忽略，甚至可能连累其他条目。
- 本地 HTTP server 想接 Desktop：`npx mcp-remote http://127.0.0.1:8000/mcp` 桥一层。

## 12. 上线前检查清单

- [ ] 已对齐 `2026-07-28`，不再依赖 `initialize` 与 `Mcp-Session-Id`
- [ ] 每个请求的 `_meta` 三个字段齐全，缺失时返回 `UnsupportedProtocolVersionError`
- [ ] 实现了 `server/discover`
- [ ] `tools/list` 输出确定性排序，参数带 `ttlMs` / `cacheScope`
- [ ] 工具名 snake_case、动词开头、带服务前缀
- [ ] 每个工具都有 `description` + 有意义的 `inputSchema` 描述 + 枚举/默认值
- [ ] 业务失败返回 `isError: true`，协议错误才用 JSON-RPC `error`
- [ ] 破坏性工具有人工确认，结果体积有上限与分页
- [ ] `stdout` 干净，日志全在 `stderr`
- [ ] 远程部署：OAuth 2.1 + PKCE、CIMD、`iss` 校验、最小 scope
- [ ] Inspector 手动过一遍、CI 里有内存 Client 集成测试

## 参考

- 规范（2026-07-28）：https://modelcontextprotocol.io/specification/2026-07-28
- 版本发布说明：https://blog.modelcontextprotocol.io/posts/2026-07-28/
- TypeScript SDK v2：https://ts.sdk.modelcontextprotocol.io/v2/
- Python SDK v2：https://py.sdk.modelcontextprotocol.io/
- Python v2 迁移指南：https://py.sdk.modelcontextprotocol.io/migration/
- 工具定义规范：https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- MCP Inspector：https://github.com/modelcontextprotocol/inspector
- FastMCP：https://github.com/PrefectHQ/fastmcp
```

---

参考资料与判断依据：2026-07-28 版规范取消了 `initialize` 握手与 `Mcp-Session-Id`、新增 `server/discover`、并在 HTTP 头里带 `Mcp-Method`/`Mcp-Name` [cite:bdeeecaa-1][cite:21d0d6f4-3]；「资源不存在」错误码由 `-32002` 统一为 `-32602` [cite:21d0d6f4-5]；TS v2 的 `serveStdio` + `registerTool` 写法 [cite:5e9bee5c-2]；Python v2 把 `FastMCP` 改名 `MCPServer`、传输配置移到 `run()` [cite:1f10b3f8-2][cite:5e9bee5c-1]；v2 SDK 与 FastMCP 的定位差异 [cite:a4706b62-1]，FastMCP 4 与官方 SDK 的内置 `FastMCP` 是两个库 [cite:a4706b62-2]；工具命名与描述实践 [cite:5deaf56f-1]、无参数工具的 `{"type":"object","additionalProperties":false}` 写法 [cite:5deaf56f-3]；Inspector 的两种跑法 [cite:a4706b62-4]；Claude Desktop 配置只吃 stdio 形状 [cite:6903cc1e-5]。
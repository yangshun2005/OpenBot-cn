# Pydantic AI Bot

一个使用 [Pydantic AI](https://ai.pydantic.dev) 编写、通过 AG-UI 提供服务的 Bot。它与
[LangGraph](../langgraph-bot) 和 [Mastra](../mastra-bot) 示例并列，用第三种语言证明了同一个观点：
OpenBot 只将 Bot 识别为 AG-UI 端点 URL，因此 Python agent 的接入方式与 TypeScript agent 完全一致。

浏览器和文件工具在每次运行的 `tools` 中由 surface 提供。Pydantic AI 将它们作为外部工具暴露给模型，
其调用会流回 OpenBot，通过受治理的网关运行——因此该进程驱动的是一个它无法直接访问的真实浏览器，
而工具循环保留在客户端，与内置 Bot 相同。

## 运行它

需要 Python 3.10+。使用 [uv](https://docs.astral.sh/uv)：

```sh
cd examples/pydantic-ai-bot
uv run --env-file ../../.env src/app.py
```

或使用普通的 virtualenv：

```sh
cd examples/pydantic-ai-bot
python -m venv .venv && . .venv/bin/activate
pip install -e .
OPENAI_API_KEY=... python src/app.py
```

它监听 `http://localhost:4600/ag-ui`（可通过 `PORT` 更改），并响应 `GET /health`。

| 变量             | 默认值    | 含义                                                 |
| ---------------- | --------- | ---------------------------------------------------- |
| `OPENAI_API_KEY` | 必填      | 由 Pydantic AI 的 OpenAI provider 读取。             |
| `BOT_MODEL`      | `gpt-5.5` | agent 运行的模型。任何支持工具调用的模型均可。       |
| `PORT`           | `4600`    | AG-UI 端点监听的端口。                               |
| `REQUIRE_KEY`    | 未设置    | 设置后，`/ag-ui` 会拒绝 `authorization` 头不匹配的运行。 |

`OPENAI_BASE_URL` 将 OpenAI provider 指向兼容的网关，与部署其余部分的配置方式相同
（参见 [docs/configuration.md](../../docs/configuration.md)）。注意该网关必须提供哪个 API：
Pydantic AI 调用的是 `/responses`，而非 `/v1/chat/completions`。仅提供 chat completions 的网关
无法响应此示例，这与两个 TypeScript Bot 所携带的约束正好相反——它们使用 chat completions，
因此无法使用需要 Responses 的模型。

## 注册它

将此外端点提供给协作者，可以通过 UI 中的 `/bots`，或作为租户包中的 `remote-ag-ui` agent：

```yaml
agents:
  - id: pydantic-analyst
    name: Pydantic Analyst
    title: Research
    role_description: Research on a governed computer, written in Pydantic AI.
    type: remote-ag-ui
    endpoint: ${PYDANTIC_BOT_AG_UI_URL:-http://localhost:4600/ag-ui}
```

## 说明

- 服务通过 `pydantic_ai.ui.ag_ui` 中的 `AGUIAdapter.dispatch_request` 完成，它读取
  `RunAgentInput`，将其 `tools` 作为外部（前端）工具暴露给模型，并返回流式 AG-UI 响应。
  已针对 `pydantic-ai` 2.33.0 验证；如果你的版本早于 `pydantic_ai.ui.ag_ui` 模块，请升级。
- 只有支持工具调用的模型才能驱动计算机。不支持工具调用的模型只会聊天，而永远不会打开页面。
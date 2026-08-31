# Run a Local Ollama Model with Conductor OSS

This guide shows how to call a **local [Ollama](https://ollama.com) model** from a Conductor workflow using the built-in AI tasks — **no Orkes account, no access key, and no tunnel required.**

If you arrived here from the Orkes Cloud Ollama integration docs and got stuck on `export orkes_access_key=...`, this is the path you want. Access keys are an Orkes Cloud concept. The OSS server reaches Ollama directly over HTTP, so there's nothing to authenticate.

## Why local works on OSS but not on the cloud playground

Ollama listens on `http://localhost:11434` on your machine. The OSS Conductor server runs on the **same machine** (or your own network), so it can reach that address directly. The hosted playground at `developer.orkescloud.com` runs in Orkes' cloud and **cannot** reach your laptop's `localhost` — that's why the cloud path needs either a public tunnel or a managed model. Running OSS locally sidesteps all of that.

## Prerequisites

- A running **Conductor OSS server** — see [docs.conductor-oss.org/quickstart](https://docs.conductor-oss.org/quickstart/) or the [main getting-started guide](../README.md).
- **Ollama** installed and running: [ollama.com/download](https://ollama.com/download).

## Step 1 — Start Ollama and pull a model

```shell
ollama serve            # starts the Ollama server on http://localhost:11434
ollama pull llama3.2    # pull any chat model you want to use
```

Verify it's up:

```shell
curl http://localhost:11434/api/tags
```

## Step 2 — Confirm the AI tasks are enabled (they are, by default)

The standard OSS server ships with the AI module **on** and Ollama **pre-pointed at localhost** — you don't have to change anything to get started.

```properties
# server/src/main/resources/application.properties
conductor.integrations.ai.enabled=true
conductor.ai.ollama.base-url=${OLLAMA_HOST:http://localhost:11434}
```

Source: `conductor/server/src/main/resources/application.properties` (`conductor.integrations.ai.enabled`, `conductor.ai.ollama.base-url`). The AI module is gated by `AIIntegrationEnabledCondition` (`conductor/ai/src/main/java/org/conductoross/conductor/config/AIIntegrationEnabledCondition.java`).

If your Ollama runs somewhere other than `localhost:11434`, point the server at it with the `OLLAMA_HOST` environment variable or by overriding the property:

```shell
export OLLAMA_HOST=http://192.168.1.50:11434
```

If your Ollama is behind an auth proxy, the server can send a custom header (both optional):

```properties
conductor.ai.ollama.auth-header-name=Authorization
conductor.ai.ollama.auth-header=Bearer your-token
conductor.ai.ollama.timeout=600s
```

Source: `conductor/ai/src/main/java/org/conductoross/conductor/ai/providers/ollama/OllamaConfiguration.java` (`base-url` defaults to `http://localhost:11434`; `auth-header-name`, `auth-header`; `timeout` defaults to `600s`).

> **No integration to register.** Unlike Orkes Cloud, OSS has no integration-registration step. The provider is resolved straight from the `llmProvider` field on the task input against providers configured at startup — set `llmProvider: "ollama"` and you're done.
> Source: `conductor/ai/src/main/java/org/conductoross/conductor/ai/AIModelProvider.java` — `getModel()` looks up `input.getLlmProvider()`; Ollama registers itself under the name `ollama` (`providers/ollama/Ollama.java`, `NAME = "ollama"`).

## Step 3 — Define a workflow with an `LLM_CHAT_COMPLETE` task

The chat task type is `LLM_CHAT_COMPLETE`. Save this as `ollama_chat.json`:

```json
{
  "name": "ollama_chat",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "chat",
      "taskReferenceName": "chat_ref",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "ollama",
        "model": "llama3.2",
        "instructions": "You are a helpful assistant. Keep answers short.",
        "messages": [
          { "role": "user", "message": "What is Conductor in one sentence?" }
        ],
        "temperature": 0.7
      }
    }
  ]
}
```

Field reference (source: `conductor/ai/src/main/java/org/conductoross/conductor/ai/models/ChatCompletion.java`, `LLMWorkerInput.java`, and `ChatMessage.java`):

| Field | Required | Notes |
|---|:---:|---|
| `llmProvider` | ✅ | Use `ollama` to route to your local Ollama server |
| `model` | ✅ | Any model you've pulled, e.g. `llama3.2`, `mistral`, `phi3` |
| `messages` | ✅ | List of `{ "role": ..., "message": ... }`. `role` is one of `user`, `assistant`, `system` |
| `instructions` | ❌ | System/prompt instructions for the model |
| `temperature` | ❌ | Sampling temperature |
| `maxTokens` | ❌ | Defaults to `8192` |
| `jsonOutput` | ❌ | Set `true` to request JSON output (mention "JSON" in the prompt) |

Register it:

```shell
curl -X POST http://localhost:8080/api/metadata/workflow \
  -H "Content-Type: application/json" \
  -d @ollama_chat.json
```

Source: `conductor/rest/src/main/java/com/netflix/conductor/rest/controllers/MetadataResource.java` — `@PostMapping("/workflow")` under `@RequestMapping("/api/metadata")`. Note the create endpoint takes a single `WorkflowDef` object (not an array).

## Step 4 — Run it

Start the workflow and let it run to completion synchronously:

```shell
curl -X POST "http://localhost:8080/api/workflow/execute/ollama_chat/1" \
  -H "Content-Type: application/json" \
  -d '{}'
```

Source: `conductor/rest/src/main/java/com/netflix/conductor/rest/controllers/WorkflowResource.java` — `@PostMapping("execute/{name}/{version}")`.

Or fire-and-forget, then poll for the result by workflow ID:

```shell
# returns a workflow ID as plain text
curl -X POST "http://localhost:8080/api/workflow" \
  -H "Content-Type: application/json" \
  -d '{ "name": "ollama_chat", "version": 1, "input": {} }'

# fetch execution + task output
curl "http://localhost:8080/api/workflow/<workflow-id>?includeTasks=true"
```

Source: `WorkflowResource.java` — `@PostMapping(produces = TEXT_PLAIN_VALUE)` on the class-level `/api/workflow` returns the workflow ID.

The model's reply lands in the `chat_ref` task's output. You can also watch the run in the UI at [http://localhost:8080](http://localhost:8080).

## Embeddings (optional)

Ollama also supports embeddings via `LLM_GENERATE_EMBEDDINGS` — set `llmProvider: "ollama"`, a `model` (e.g. `nomic-embed-text`), and a `text` field; the vector comes back on the task output under `result`.

Source: `conductor/ai/src/main/java/org/conductoross/conductor/ai/tasks/worker/LLMWorkers.java` — `@WorkerTask("LLM_GENERATE_EMBEDDINGS")`.

> Image, audio, and video generation are **not** supported on Ollama — `getImageModel()` in `providers/ollama/Ollama.java` throws `UnsupportedOperationException`. Use those task types with a provider that supports them (e.g. OpenAI, Gemini, Stability AI).

## More

The AI module supports 12 providers and many more task types (text completion, embeddings, vector search, MCP tools, document generation). For the full reference, see the [Conductor AI module README](https://github.com/conductor-oss/conductor/blob/main/ai/README.md).

## Found a problem?

If anything here is wrong or unclear, [open an issue](https://github.com/conductor-oss/getting-started/issues/new).

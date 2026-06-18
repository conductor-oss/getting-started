# Conductor OSS — Getting Started

Your hub for getting Conductor OSS up and running. Find deployment guides, sample code, and links to every SDK — or open an issue if something in the new-user experience is broken or missing.

## Guides

| Topic | Guide |
|---|---|
| **Workload isolation** | [`workload-isolation/`](./workload-isolation/) — isolate team workloads so one team's spike doesn't affect others |
| **Local LLM (Ollama)** | [`local-llm-ollama/`](./local-llm-ollama/) — call a local Ollama model from a workflow; no Orkes account or access key needed |

## Deploy Conductor

Choose the path that fits your environment:

| Method | Guide |
|---|---|
| **Docker** (quickest) | [docs.conductor-oss.org/quickstart](https://docs.conductor-oss.org/quickstart/) |
| **Kubernetes / OpenShift** | [`kubernetes/`](./kubernetes/) — tested manifest + guide, Postgres only, no Elasticsearch |
| **Build from source** | [conductor-oss/conductor](https://github.com/conductor-oss/conductor) |

## SDKs

| Language | Repo |
|---|---|
| Java | [`conductor-oss/java-sdk`](https://github.com/conductor-oss/java-sdk) |
| Python | [`conductor-oss/python-sdk`](https://github.com/conductor-oss/python-sdk) |
| JavaScript / TypeScript | [`conductor-oss/javascript-sdk`](https://github.com/conductor-oss/javascript-sdk) |
| Go | [`conductor-oss/go-sdk`](https://github.com/conductor-oss/go-sdk) |
| C# / .NET | [`conductor-oss/csharp-sdk`](https://github.com/conductor-oss/csharp-sdk) |
| Clojure | [`conductor-oss/clojure-sdk`](https://github.com/conductor-oss/clojure-sdk) |

## Examples and sample apps

- [`conductor-oss/conductor-samples`](https://github.com/conductor-oss/conductor-samples) — runnable workflow examples
- [`conductor-oss/awesome-conductor-apps`](https://github.com/conductor-oss/awesome-conductor-apps) — community-built apps

## Full documentation

[docs.conductor-oss.org](https://docs.conductor-oss.org)

## Found a problem?

If something in the quickstart, a guide in this repo, or the install flow is broken or confusing — **[open an issue here](https://github.com/conductor-oss/getting-started/issues/new)**. This is the right place even if you're not sure which repo the fix belongs in.

Fixes typically land in:

- [`conductor-oss/conductor`](https://github.com/conductor-oss/conductor) — server, docs, quickstart
- [`conductor-oss/conductor-cli`](https://github.com/conductor-oss/conductor-cli) — CLI behavior
- This repo (`conductor-oss/getting-started`) — deployment guides and onboarding content

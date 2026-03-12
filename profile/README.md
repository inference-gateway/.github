<div align="center">

<img src="https://avatars.githubusercontent.com/u/195813097?s=200&v=4" width="100" alt="Inference Gateway Logo" />

# Inference Gateway

**An open-source, cloud-native, high-performance gateway unifying multiple LLM providers**

[![GitHub Stars](https://img.shields.io/github/stars/inference-gateway/inference-gateway?style=flat-square&logo=github)](https://github.com/inference-gateway/inference-gateway/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](https://opensource.org/licenses/MIT)
[![Go](https://img.shields.io/badge/Built%20with-Go-00ADD8?style=flat-square&logo=go)](https://golang.org)
[![Docs](https://img.shields.io/badge/docs-inference--gateway.com-7C3AED?style=flat-square)](https://docs.inference-gateway.com)

[📖 Documentation](https://docs.inference-gateway.com) · [🚀 Getting Started](https://docs.inference-gateway.com/getting-started) · [💬 Discussions](https://github.com/orgs/inference-gateway/discussions) · [🐛 Issues](https://github.com/inference-gateway/inference-gateway/issues)

</div>

---

## 🌐 What is Inference Gateway?

**Inference Gateway** is a proxy server that provides a **unified API** to interact with multiple large language model (LLM) providers — from local solutions like [Ollama](https://ollama.com) to major cloud providers like OpenAI, Anthropic, Groq, Cohere, Cloudflare, and DeepSeek.

Stop managing multiple SDKs and API keys. Route all your LLM traffic through a single, production-ready gateway.

```bash
# One endpoint. Every provider.
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "openai/gpt-4o", "messages": [{"role": "user", "content": "Hello!"}]}'
```

---

## ✨ Key Features

| Feature | Description |
|---|---|
| 🔀 **Unified API** | One OpenAI-compatible endpoint for all LLM providers |
| 🔌 **MCP Integration** | Native [Model Context Protocol](https://docs.inference-gateway.com/mcp) support for automatic tool discovery |
| 🤖 **A2A Protocol** | [Agent-to-Agent](https://docs.inference-gateway.com/a2a) coordination across specialized agents |
| 🌊 **Streaming** | Real-time token streaming from all supported providers |
| ☸️ **Kubernetes Ready** | First-class K8s support with Operator and HPA scaling |
| 📊 **Observability** | OpenTelemetry integration for monitoring and tracing |
| 🔒 **Privacy First** | Self-hosted, zero data collection, MIT licensed |
| 🌿 **Lightweight** | ~10.8MB binary with minimal resource footprint |

---

## 🏗️ Ecosystem

### Core

| Repository | Description |
|---|---|
| [**inference-gateway**](https://github.com/inference-gateway/inference-gateway) | The core gateway server |
| [**operator**](https://github.com/inference-gateway/operator) | Kubernetes Operator for lifecycle management |
| [**cli**](https://github.com/inference-gateway/cli) | Agentic CLI assistant with project context awareness |
| [**schemas**](https://github.com/inference-gateway/schemas) | MCP, A2A, and OpenAPI schemas |
| [**docs**](https://github.com/inference-gateway/docs) | Documentation site |

### SDKs

| Repository | Language |
|---|---|
| [**sdk**](https://github.com/inference-gateway/sdk) | Go |
| [**typescript-sdk**](https://github.com/inference-gateway/typescript-sdk) | TypeScript |
| [**rust-sdk**](https://github.com/inference-gateway/rust-sdk) | Rust |
| [**python-sdk**](https://github.com/inference-gateway/python-sdk) | Python |

### Agent Development

| Repository | Description |
|---|---|
| [**adl**](https://github.com/inference-gateway/adl) | Agent Definition Language for declarative agent definitions |
| [**adl-cli**](https://github.com/inference-gateway/adl-cli) | Scaffold and manage A2A-powered enterprise agents |
| [**adk**](https://github.com/inference-gateway/adk) | Agent Development Kit (Go) |
| [**typescript-adk**](https://github.com/inference-gateway/typescript-adk) | Agent Development Kit (TypeScript) |
| [**rust-adk**](https://github.com/inference-gateway/rust-adk) | Agent Development Kit (Rust) |

### A2A Agents

| Repository | Description |
|---|---|
| [**google-calendar-agent**](https://github.com/inference-gateway/google-calendar-agent) | Google Calendar scheduling & automation |
| [**browser-agent**](https://github.com/inference-gateway/browser-agent) | Browser automation via Playwright |
| [**documentation-agent**](https://github.com/inference-gateway/documentation-agent) | Context7-style documentation access |
| [**grafana-agent**](https://github.com/inference-gateway/grafana-agent) | Grafana dashboards automation |
| [**n8n-agent**](https://github.com/inference-gateway/n8n-agent) | n8n workflow generation & automation |
| [**mock-agent**](https://github.com/inference-gateway/mock-agent) | Mocking and testing |

### Tools & Community

| Repository | Description |
|---|---|
| [**a2a-debugger**](https://github.com/inference-gateway/a2a-debugger) | A2A agents troubleshooter |
| [**registry**](https://github.com/inference-gateway/registry) | Registry for A2A agents |
| [**awesome-a2a**](https://github.com/inference-gateway/awesome-a2a) | Curated list of A2A-compatible agents |
| [**infer-action**](https://github.com/inference-gateway/infer-action) | GitHub Action for the Infer CLI |

---

## 🚀 Quick Start

```bash
# Run with Docker
docker run -p 8080:8080 \
  -e OPENAI_API_KEY=your-key \
  ghcr.io/inference-gateway/inference-gateway:latest

# Or install the CLI
curl -fsSL https://raw.githubusercontent.com/inference-gateway/cli/main/install.sh | bash
infer init && infer chat
```

👉 Full setup guide: [docs.inference-gateway.com/getting-started](https://docs.inference-gateway.com/getting-started)

---

## 🤝 Contributing

We welcome contributions of all kinds — bug reports, feature requests, documentation improvements, and code!

- ⭐ **Star** the [main repo](https://github.com/inference-gateway/inference-gateway) to show your support
- 🐛 **Report bugs** via [GitHub Issues](https://github.com/inference-gateway/inference-gateway/issues)
- 💬 **Join discussions** in [GitHub Discussions](https://github.com/orgs/inference-gateway/discussions)
- 🔧 **Submit PRs** — see `CONTRIBUTING.md` in each repository

---

<div align="center">

Released under the [MIT License](https://opensource.org/licenses/MIT) · Built with ❤️ in Go

</div>

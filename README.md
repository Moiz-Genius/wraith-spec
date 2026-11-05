# 👻 Wraith — Autonomous CLI Coding Agent Specification

[![Specification](https://img.shields.io/badge/Spec-Architecture%20%26%20Prompt-8A2BE2.svg)](WRAITH_BUILD_PROMPT.md)
[![TypeScript](https://img.shields.io/badge/Target-TypeScript%205.x-3178C6.svg?logo=typescript)](https://www.typescriptlang.org/)
[![Runtime](https://img.shields.io/badge/Runtime-Node.js%2020%2B-brightgreen.svg?logo=node.js)](https://nodejs.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/Moiz-Genius/wraith-spec/pulls)

> *"The ghost in your codebase."*

**Wraith** is a full architectural blueprint and implementation specification for a local-first, blazing-fast CLI autonomous coding agent. Designed with zero bloat, full filesystem and shell authority, provider-agnostic multi-model inference, and sub-second cold starts.

---

## ⚡ Core Philosophy

- **Zero Bloat, Terminal-First**: No background daemons, no HTTP servers, no WebSockets, no heavy web GUIs or mobile wrappers. Wraith lives entirely inside your terminal where developers work.
- **Full Operational Autonomy**: Grants the agent full authority to inspect, create, edit, delete, and execute code within your project directory without tedious confirmation prompts for every atomic step.
- **Sub-Second Cold Start**: Instant startup with lazy-loaded provider SDKs and lightweight dependencies.
- **Model Agnostic**: First-class support for OpenAI, Anthropic Claude, Groq, Mistral, Google Gemini, Ollama, and any custom OpenAI-compatible endpoint without code modifications.
- **Signature Identity**: Cohesive purple ghost aesthetic, consistent terminal glyphs (`👻`, `🔮`, `⚡`, `💀`, `✨`), and high-clarity streaming output.

---

## 🏗️ Architecture & Component Design

The specification in [WRAITH_BUILD_PROMPT.md](WRAITH_BUILD_PROMPT.md) defines 17 core architectural sections:

1. **Identity & Theme**: Visual language, terminal palette, and consistent line prefix glyphs.
2. **CLI Shell & REPL**: Command parsing, flag handling, interactive read-eval-print loop with multi-line input support.
3. **Provider Layer**: Unified streaming client interface abstracting major AI APIs and local models.
4. **Tool System**: Core primitives for autonomous engineering:
   - `read_file`: Byte/line-sliced safe file inspection.
   - `write_file`: Atomic file creation and full overwrites.
   - `edit_file`: Robust search-and-replace chunk editing with fuzzy tolerance.
   - `delete_file`: File removal with pre-deletion checkpoints.
   - `bash`: Interactive or bounded command execution with streaming stdout/stderr.
   - `glob` & `grep`: Ultra-fast pattern and regex workspace search.
5. **Context & Compaction Engine**: Token budgeting, sliding-window summarization, and conversation compaction.
6. **Safety & Checkpointing**: Automatic ephemeral git/stash checkpoints prior to destructive operations to allow instant operator rollbacks.
7. **Persona Injection**: Project-level persona rules via `AGENTS.md` and `SOUL.md`.

---

## 🚀 How to Use This Specification

### Bootstrapping with an AI IDE Agent
You can feed [WRAITH_BUILD_PROMPT.md](WRAITH_BUILD_PROMPT.md) directly to an AI coding assistant (such as Cursor, Windsurf, or Claude Code) as a complete instruction set:

1. Create a new project folder:
   ```bash
   mkdir wraith && cd wraith
   npm init -y
   ```
2. Paste the contents of `WRAITH_BUILD_PROMPT.md` as the prompt to build the entire TypeScript codebase end-to-end.

---

## 📄 Repository Contents

- [`WRAITH_BUILD_PROMPT.md`](WRAITH_BUILD_PROMPT.md) — The comprehensive 17-section build prompt and technical blueprint.
- [`article.md`](article.md) — Geopolitical research paper and reference notes.

---

## 📄 License

This specification is open-sourced under the [MIT License](LICENSE).

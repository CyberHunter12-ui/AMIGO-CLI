# AMIGO CLI 🤖⚡

> **A Hybrid, Tool-Augmented AI Workspace Assistant & Orchestration Engine**

AMIGO CLI is a modular, terminal-based AI assistant designed to bridge conversational intelligence with system automation, local document grounding, multi-agent orchestration, and browser workflows. Built with a security-first architecture, it provides an extensible command-line workspace without granting risky, unconstrained system permissions.

---

## ✨ Key Features

- **Hybrid Inference Architecture:** Seamlessly supports both locally hosted fine-tuned models (SFT) and cloud-based OpenAI-compatible endpoints (such as NVIDIA API).
- **Model Context Protocol (MCP) Integration:** Native support for tool discovery and execution using the Model Context Protocol:
  - **Headless Browser Automation:** Driven by Playwright MCP for inspection, scraping, and screenshot generation.
  - **Bounded Filesystem Operations:** Controlled workspace reads and strictly isolated artifact output.
  - **System Utility Tools:** Time zone conversion, date processing, and weather prefetching fallback.
- **Conservative Multi-Agent Orchestration:** Task-difficulty routing layer that dispatches parallel worker agents (_Researcher, Coder, Executor, FileOps_) only when complex multi-step execution is required.
- **Local Context & Document Reference:** Ground prompts using local files (`.pdf`, `.docx`, `.md`, `.txt`, `.csv`, `.json`, code files) stored in `my_files/` or referenced directly.
- **Strict Security Boundaries:** AST-level safety parsing, regex guardrails, bounded directory operations (`model_out_dir/`), and user-approval workflows before code execution.
- **Multimodal & Vision Capabilities:** Screen capture assistance (`/screen`), explicit image analysis, and OCR fallback context processing.
- **Persistent Memory & Session Syncing:** Bounded, privacy-preserving long-term fact extraction (`echo_memory.json`) and atomic chat session management.
- **Voice & Accessibility:** Optional speech recognition input, queue-managed text-to-speech (TTS), clipboard helpers, and Windows tray/hotkey integration.

---

## 🏗️ System Architecture

```text
                                +---------------------------+
                                |        User Input         |
                                |  (Terminal UI / Speech)   |
                                +-------------+-------------+
                                              |
                                              v
                                +-------------+-------------+
                                |     Application Layer     |
                                |       (Router / UI)       |
                                +-------------+-------------+
                                              |
                +-----------------------------+-----------------------------+
                |                             |                             |
                v                             v                             v
   +------------+------------+   +------------+------------+   +------------+------------+
   |   Task Difficulty Router|   |   Local Document Context|   | Dynamic Session & Memory|
   | (Single Agent vs Multi) |   |    (`my_files` Loader)   |   |     Persistence Engine    |
   +------------+------------+   +------------+------------+   +------------+------------+
                |                             |                             |
                +-----------------------------+-----------------------------+
                                              |
                                              v
                                +-------------+-------------+
                                | Hybrid Model Interface    |
                                | (Local SFT / Cloud API)   |
                                +-------------+-------------+
                                              |
                                              v
                                +-------------+-------------+
                                |    MCP Tool Registry      |
                                | (Playwright, FS, Time)    |
                                +-------------+-------------+
                                              |
                                              v
                                +-------------+-------------+
                                |  Guarded Output Sandbox   |
                                |    (`model_out_dir/`)     |
                                +---------------------------+
```

---

## 🔐 Security & Directory Isolation Model

AMIGO CLI enforces an explicit access boundary:

| Directory        | Scope & Permissions       | Purpose                                                                                   |
| :--------------- | :------------------------ | :---------------------------------------------------------------------------------------- |
| `my_files/`      | **Read-Only** (App Logic) | User-provided reference documents and data sources.                                       |
| `model_out_dir/` | **Read / Write** (FS MCP) | The single permitted output directory for generated project files, code, and screenshots. |
| `config/`        | **System Internal**       | Atomic JSON persistence for models, sessions, and long-term user facts.                   |

- **Code Execution:** Safe-candidate analysis via AST checks and explicit execution approval prompts.
- **Credential Protection:** API keys and environment parameters remain isolated in local `.env` files and are never included in persistence structures or output logs.

---

## 🚀 Getting Started

### Prerequisites

- **Python:** 3.10+ (Python 3.13 supported)
- **Node.js & npx:** Required for starting stdio MCP servers (Playwright & Filesystem)
- **Tesseract OCR (Optional):** Required if local OCR extraction for images is desired

### Installation

1. **Clone the repository:**

   ```bash
   git clone https://github.com/your-username/amigo-cli.git
   cd amigo-cli
   ```

2. **Set up a virtual environment:**

   ```bash
   python -m venv venv
   # On Windows:
   venv\Scripts\activate
   # On macOS/Linux:
   source venv/bin/activate
   ```

3. **Install dependencies:**

   ```bash
   pip install -r requirements.txt
   ```

4. **Configure environment variables:**
   Create a `.env` file in the project root:

   ```env
   NVIDIA_API_KEY=your_api_key_here
   # AMIGO_ENABLE_GLOBAL_HOTKEY=true  # Optional Windows integration
   ```

5. **Launch the CLI:**
   ```bash
   python main.py
   ```

---

## 🛠️ Usage & Essential Slash Commands

AMIGO CLI features a built-in command router for local settings, sessions, and capabilities.

| Command              | Category   | Function                                                    |
| :------------------- | :--------- | :---------------------------------------------------------- |
| `/help`              | General    | View command categories and active options                  |
| `/status`            | Runtime    | View active session, provider model, MCP status, and memory |
| `/sessions`          | Sessions   | List stored chat sessions                                   |
| `/new <name>`        | Sessions   | Create and activate a new conversation session              |
| `/switch <id\|name>` | Sessions   | Switch active chat session                                  |
| `/model`             | Config     | Interactively select or configure provider models           |
| `/files`             | Workspace  | Inspect supported reference files inside `my_files/`        |
| `/screen`            | Multimodal | Capture current display and analyze visual context          |
| `/multiline`         | Input      | Enter multiline input mode for pasting long text or emails  |
| `/voice` / `/text`   | Audio      | Toggle between microphone voice recognition and typed input |
| `/memory`            | Profile    | Inspect extracted long-term user memory facts               |

### Example Prompts

```text
> Read the requirements document in my_files and give me an executive summary.
> Create a Python web scraper project inside model_out_dir/scraper with src and tests.
> Take a screenshot of https://news.ycombinator.com using Playwright and summarize the top headline.
> Read and analyze "C:\Users\username\Documents\report.pdf"
```

---

## 📁 Repository Structure

```text
amigo-cli/
│
├── main.py                     # Primary Application Coordinator & Event Loop
├── .mcp.json                   # Model Context Protocol Server Configurations
├── requirements.txt            # Project Python Dependencies
│
├── core/                       # Core Architecture Modules
│   ├── paths.py                # Stable Workspace Directory Resolution
│   ├── config_manager.py       # Configuration & State Management
│   ├── tools.py                # Web Search, Extraction, & Guarded Execution
│   ├── security.py             # Security Guardrails & AST Verification
│   ├── orchestrator.py         # Multi-Agent Routing Logic
│   ├── sub_agent.py            # Parallel Worker Implementations
│   ├── mcp_client.py           # Async JSON-RPC MCP Bridge
│   ├── features.py             # Focused Intelligent Feature Suite
│   └── speech.py               # Speech Recognition & TTS Queue Engine
│
├── my_files/                   # User Source Documents (Read-Only Context)
└── model_out_dir/              # Bounded Output Root for Generated Artifacts
    └── screenshots/            # Playwright & System Screenshots
```

---

## 🛡️ License & Disclaimer

This repository is distributed under the **MIT License**.

_Disclaimer: AMIGO CLI provides guarded AST checks and strict output directory policies. However, code execution and system automation tools should be monitored, and production deployments should run inside isolated containerized environments._

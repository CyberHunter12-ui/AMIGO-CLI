# AMIGO CLI

A all time go buddy to help you around and do things just from your terminal  ...



\# AMIGO CLI: Complete Architecture and Workflow



\## 1. System overview



AMIGO CLI is a Python terminal-based AI assistant. The active application is

`main.py`; the reusable implementation is organized under `core/`. The system

combines:



\- OpenAI-compatible model inference through NVIDIA's endpoint

\- Configurable model providers

\- Text and voice interaction

\- Persistent sessions and long-term user memory

\- Document reference through `my\_files`

\- Conservative difficulty-based multi-agent orchestration

\- Headless browser automation through Playwright MCP

\- Controlled filesystem access through Filesystem MCP

\- Project-bound output generation in `model\_out\_dir`

\- User-approved code execution with static safety checks

\- Smart spam detection

\- Intelligent document classification

\- Sentiment analysis

\- Personalized recommendations

\- OCR text extraction

\- Document question answering

\- Web research summarization

\- Screenshot and image prompt attachments

\- Current-screen assistance

\- Optional Windows integrations: tray actions, explicit clipboard commands,

&#x20; notifications, global hotkey, and launch-time file handoff



The high-level design is:



```text

User

&#x20; |

&#x20; v

Terminal UI / Voice Input

&#x20; |

&#x20; v

main.py (application coordinator)

&#x20; |

&#x20; +--> command router (/model, /session, /files, /persona, ...)

&#x20; |

&#x20; +--> document context loader (my\_files)

&#x20; |

&#x20; +--> difficulty classifier

&#x20; |       |

&#x20; |       +--> direct single-agent request

&#x20; |       |

&#x20; |       +--> complex independent tasks

&#x20; |               |

&#x20; |               v

&#x20; |         Orchestrator --> parallel SubAgents

&#x20; |

&#x20; +--> MCPToolRegistry --> Playwright / Filesystem MCP servers

&#x20; |

&#x20; v

OpenAI-compatible model API

&#x20; |

&#x20; v

Response, tool calls, files, screenshots, session state

```



\## 2. Main runtime components



\### 2.1 `main.py`



`main.py` is the active entry point and coordinator. It:



1\. Loads environment variables and application configuration.

2\. Loads the active model and persona.

3\. Creates the OpenAI-compatible asynchronous client.

4\. Starts the MCP server registry.

5\. Loads the active session and long-term memory.

6\. Starts the voice listener thread.

7\. Displays the terminal interface.

8\. Receives commands or user prompts.

9\. Adds relevant document, memory, and research context.

10\. Decides whether orchestration is necessary.

11\. Sends the final prompt to the active model.

12\. Executes model-requested MCP tools when needed.

13\. Stores the assistant response in the active session.

14\. Closes browser contexts after each request.



\### 2.2 `core/config\_manager.py`



This module owns persistent JSON configuration:



| File | Purpose |

|---|---|

| `config/models\_config.json` | Model names, provider IDs, and active model index |

| `config/personalities\_config.json` | Persona profiles and active persona |

| `config/chat\_sessions.json` | Named conversations and token totals |

| `config/echo\_memory.json` | Extracted long-term user facts |



The model configuration supports:



\- Adding a model

\- Renaming a model

\- Removing a model

\- Selecting the active model

\- Duplicate model-ID rejection

\- Validation of empty or invalid entries

\- Automatic active-index correction after removal



Session helpers also resolve a numeric list position, exact ID, or

case-insensitive session name without changing the persisted session schema.

New session IDs receive a collision-safe suffix when needed.



Example model configuration:



```json

{

&#x20; "active\_index": 0,

&#x20; "models": \[

&#x20;   {

&#x20;     "name": "Fast Model",

&#x20;     "id": "provider/fast-model"

&#x20;   },

&#x20;   {

&#x20;     "name": "Reasoning Model",

&#x20;     "id": "provider/reasoning-model"

&#x20;   }

&#x20; ]

}

```



\### 2.3 `core/paths.py`



This module defines stable application paths:



```text

PROJECT\_ROOT = project directory containing main.py

MY\_FILES\_DIR = PROJECT\_ROOT/my\_files

MODEL\_OUT\_DIR = PROJECT\_ROOT/model\_out\_dir

```



Paths are derived from the application location rather than the shell's current

directory. Therefore, launching the program from another directory does not

redirect configuration or generated output.



\### 2.4 `core/tools.py`



This module provides local capabilities:



\- Internet search through DuckDuckGo

\- Wikipedia search

\- Document discovery in `my\_files`

\- PDF text extraction through `pypdf`

\- DOCX text extraction

\- Plain-text, Markdown, JSON, CSV, YAML, HTML, XML, Python, and log reading

\- Safe model-output file creation

\- User-approved code execution



Document reads are restricted to the `my\_files` folder when using the local

document-reference workflow. General read requests may also use the read-only

Filesystem MCP server.



Generated files use `\_model\_output\_path()`, which prevents path traversal outside

`model\_out\_dir`.



\### 2.5 `core/features.py`



This isolated capability layer provides the requested focused features without

changing the base model or MCP protocol:



| Feature | Implementation |

|---|---|

| Smart spam detection | Local indicator scoring for urgency, credential requests, financial requests, links, and excessive punctuation |

| Intelligent document classifier | Keyword-based category scoring for academic, business, technical, legal, and personal documents |

| Sentiment analysis | Lightweight positive/negative term scoring with neutral fallback |

| Personalized recommendations | Goal/topic matching against recommendation templates |

| OCR | Pillow plus pytesseract for readable image paths |

| Document question answering | Reads bounded document context, then asks the active model |

| Web research summarizer | Reuses existing web search, then asks the active model to summarize |

| Screenshot attachments | Finds explicit image paths or recent screenshots and sends bounded image content with the prompt |

| Current-screen assistance | Captures the display on `/screen` or an explicit screen-assistance request, temporarily minimizing the Windows CLI console so the underlying app is captured, then reuses image attachment handling |

| Windows tray integration | Runs the existing `core/tray.py` notification-area icon on a background thread with focus, screen, session, status, and quit actions |

| Optional system adapters | `core/integrations.py` provides thread-safe events, explicit clipboard access, redacted notifications, opt-in hotkey registration, and launch-time file arguments |

| Explicit file context | Finds existing paths in the prompt and reads supported external documents as bounded read-only context |



Features can be called explicitly with slash commands or activated by

conservative natural-language prefixes such as `check spam`, `classify

document`, `analyze sentiment`, `perform ocr`, `ask the document`, and `web

research`. Ordinary prompts continue through the existing workflow.



The request path also skips the background memory-extraction call unless the

prompt contains likely personal-fact markers. Simple requests use a smaller

response budget and fewer tool rounds, while complex requests retain the

existing larger processing path.



Explicit file paths are resolved only for reading. Their contents are appended

to the current prompt as bounded context; they do not alter the write or code

execution boundaries.



If an API endpoint rejects the image with a multimodal-processing error, the

request is retried once as text-only. OCR output is included as fallback

context when the local Tesseract engine is available; otherwise the response

states that visual details could not be verified.



For current-screen assistance, `core/features.py` captures the display into

`model\_out\_dir/screenshots` and attaches the resulting PNG only to the current

model request. The capture is user-triggered and failures are returned as a

visible notice.



Persistence uses atomic JSON replacement for model, personality, session, and

long-term-memory files. MCP configuration failures are reported as startup

warnings instead of preventing ordinary text mode, and MCP/tray cleanup is

safe to invoke more than once.



Optional system integrations are isolated in `core/integrations.py`. A

thread-safe event bridge carries tray and hotkey events to the main loop.

Clipboard access is command-triggered and bounded, notifications are redacted

and best-effort, and startup file arguments reuse the existing read-only file

context path. Optional dependencies never become requirements for ordinary

text prompts.



Speech services are isolated in `core/speech.py`. `SpeechSettings` bounds

language, timeout, rate, and volume values; `SpeechOutput` owns one reusable

TTS worker and queue; and recognition errors are categorized before being shown

to the user. The existing microphone loop remains active only in voice mode, so

text-only use does not add audio work. TTS engine and voice metadata reuse avoid

reinitialization per response, and the bounded queue prevents stale playback

from creating unbounded latency.



The model boundary also uses `core/context\_budget.py` for deterministic context

limits. Recent history and supplementary document/research context are compacted

only when constructing an outbound request, while persisted session data and

the source files remain unchanged. Simple prompts use smaller completion and

tool-round budgets; complex requests retain larger budgets.



Session synchronization compares active state before persisting, retaining

atomic writes while avoiding duplicate disk work. Voice and integration event

queues are bounded with newest-event retention, and user-facing runtime errors

are bounded and credential-redacted before display. These safeguards reduce

latency and stale work without changing command routing or provider requests.



Long-term learning reuses the existing structured extraction call. Deterministic

candidate detection skips unrelated prompts; `core/memory.py` normalizes,

deduplicates, bounds, and filters sensitive values before persistence. Common

first-person preference, workflow, and project statements are extracted locally

before the optional structured parser, reducing latency and token use. Existing

memory is merged rather than replaced, and only compact non-empty fields are

injected into the system prompt as untrusted preference/context data. `/memory`

shows the stored context, `/memory refresh` reprocesses recent user turns, and

`/memory clear` preserves the existing clear-memory behavior.



The terminal UI keeps `draw\_screen()` compact by showing only essential status

and quick commands. Full command discovery is rendered on demand by

`core/command\_help.py`: `/help` shows categories and `/help <category>` shows

the commands in one category. This is presentation-only; command recognition

and all existing aliases remain unchanged.



`/status` reports current runtime state without contacting the model. The

request loop retains only the latest failed prompt in memory, and `/retry`

re-enters the normal request path for that prompt. No successful request is

duplicated automatically and no additional persistence format is introduced.



Launch-time file arguments are converted into the same bounded read-only

context path used by explicit prompt file references. Tray and hotkey callbacks

only enqueue events; the main asyncio loop performs screen capture, session

changes, and model requests. This prevents background UI threads from

concurrently mutating the model pipeline.



On Windows, screen capture excludes known Console Host and Windows Terminal

window classes instead of hiding whichever window happens to be foreground.

This keeps other applications visible for assistance and avoids unexpected

focus changes after tray-triggered captures.



The per-request optimization path computes the simple/complex routing decision

once and reuses it for orchestration, tool-round, temperature, and token

settings. It does not change the existing thresholds or complex-task behavior.



\### 2.6 `core/orchestrator.py`



The orchestrator prevents unnecessary agent creation.



It first classifies a request as:



\- `simple`: conversation, explanation, one question, or one focused change

\- `moderate`: multiple sequential steps that one agent can complete

\- `complex`: multiple independent workstreams that benefit from parallel agents



Only a complex request with at least two valid independent tasks is allowed to

create parallel sub-agents. Long prompts alone do not cause multi-agent

execution.



\### 2.7 `core/sub\_agent.py`



Supported worker roles are:



| Role | Responsibility |

|---|---|

| `Researcher` | Internet and Wikipedia research |

| `Coder` | Generate code or technical implementation content |

| `Executor` | Run approved code through the guarded execution path |

| `FileOps` | Read or write through the controlled local tools |



Each worker returns a structured `SubAgentResult` containing task ID, role,

status, and output. The orchestrator combines those results into context for the

main model.



\### 2.8 `core/mcp\_client.py`



The MCP bridge starts MCP servers as subprocesses and communicates through

newline-delimited JSON-RPC over standard input/output.



Its responsibilities are:



1\. Read `.mcp.json`.

2\. Replace `${workspaceFolder}` with the stable project path.

3\. Start each configured MCP server.

4\. Send the MCP `initialize` request.

5\. Send the `notifications/initialized` notification.

6\. Discover tools through `tools/list`.

7\. Convert MCP tool schemas to OpenAI-compatible function tools.

8\. Receive model tool calls.

9\. Invoke the correct MCP server.

10\. Return tool results to the model.

11\. Close browser contexts and MCP processes cleanly.



\## 3. MCP architecture



The current `.mcp.json` configures four MCP servers:



\### Playwright MCP



```text

Purpose: Browser navigation, inspection, and screenshots

Mode: Headless

Output: Screenshots redirected to model\_out\_dir/screenshots

Cleanup: Browser contexts closed after each model request

```



\### Writable Filesystem MCP



```text

Server: filesystem-model-output

Root: PROJECT\_ROOT/model\_out\_dir

Permission: Read and write within model\_out\_dir

```



This is the primary filesystem channel for generated project artifacts.



\### Read-only Filesystem MCP



```text

Server: filesystem-read-only

Root: PROJECT\_ROOT/..

Permission: Read/list/search/info only

```



The bridge exposes only an allowlist of read-oriented tools:



\- `read\_file`

\- `read\_text\_file`

\- `read\_media\_file`

\- `read\_multiple\_files`

\- `list\_directory`

\- `list\_directory\_with\_sizes`

\- `directory\_tree`

\- `search\_files`



\### Time MCP



```text

Server: time

Package: mcp-server-time (official MCP server)

Launcher: uvx mcp-server-time

Permission: Read-only time and timezone calculations

Fallback: Startup warning if uv/uvx or the optional package is unavailable

```



The model is instructed to use this server for current time, date, timezone

conversion, and other time-sensitive calculations instead of guessing. The

server receives no filesystem path and has no write or execution boundary.

Windows users can enable it by installing `uv`; the CLI otherwise continues

without this optional server and reports the startup failure as a warning.



If the optional server cannot start, `MCPToolRegistry` exposes local

standard-library time tools and an on-demand Open-Meteo weather tool instead.

These fallbacks are invoked only through the model tool loop, have bounded

network timeouts, and do not create background polling or add filesystem access.

Explicit time/weather requests also use a deterministic prefetch before model

inference so the response has a current result even when a model declines to

call a tool; unrelated prompts do not trigger this path.

\- `get\_file\_info`

\- `list\_allowed\_directories`



Write, edit, move, create, and delete operations are filtered out.



\## 4. Complete request workflow



\### Phase A: Startup



```text

1\. Process starts with python main.py

2\. Stable PROJECT\_ROOT is calculated

3\. .env is loaded

4\. NVIDIA\_API\_KEY is checked

5\. models\_config.json is loaded

6\. personalities\_config.json is loaded

7\. MCP servers are started

8\. MCP tools are discovered

9\. chat\_sessions.json is loaded

10\. echo\_memory.json is loaded

11\. Voice listener thread starts

12\. Terminal screen is rendered

```



\### Phase B: Input collection



The user can interact through:



\- Text mode

\- Voice mode

\- Slash commands

\- Multiline paste mode for emails and long text



Voice input is collected by a background listener and placed into a queue. The

main asynchronous loop consumes that queue without blocking the rest of the

application.



For multiline text, `/multiline`, `/paste`, or `/email` starts a bounded input

capture loop. Every line is collected until `/end` appears on its own line.

The resulting block is joined with newline characters and submitted as one

user message, so email paragraphs are not processed one line at a time.



\### Phase C: Command routing



Commands beginning with `/` are handled locally and do not go to the model.



| Command | Function |

|---|---|

| `/text` | Switch to text input |

| `/voice` | Switch to voice input |

| `/session` | List, create, load, or delete sessions |

| `/model` | List and select configured models |

| `/model add` | Add a provider model |

| `/model edit` | Rename a model |

| `/model remove` | Remove a model |

| `/persona` | Select a personality |

| `/files` | List supported files in `my\_files` |

| `/toggle speech on/off` | Control text-to-speech |

| `/clear mem` | Clear stored memory and current conversation state |

| `/quit` | Save state and shut down |



\### Phase D: Context preparation



For a normal user prompt, the application can add:



1\. Current date and time.

2\. Long-term user memory.

3\. Relevant document content from `my\_files`.

4\. Multi-agent research or implementation results.

5\. MCP capability instructions.

6\. Recent user and assistant history.



Only the most recent six user/assistant messages are sent as conversational

history, which limits prompt growth.



\### Phase E: Difficulty decision



```text

Prompt

&#x20; |

&#x20; +--> Obviously short/simple?

&#x20; |       |

&#x20; |       +--> Direct model request

&#x20; |

&#x20; +--> Longer or multi-step?

&#x20;         |

&#x20;         v

&#x20;     Orchestrator classification

&#x20;         |

&#x20;         +--> Simple/moderate: direct model request

&#x20;         |

&#x20;         +--> Complex + independent tasks:

&#x20;                 parallel sub-agents

&#x20;                 combined results

&#x20;                 main model synthesis

```



\### Phase F: Model and MCP tool loop



The active model receives the prepared messages and optional MCP tools.



```text

Send completion request

&#x20; |

&#x20; v

Model returns text or tool calls

&#x20; |

&#x20; +--> Text response

&#x20; |       |

&#x20; |       v

&#x20; |    Finish request

&#x20; |

&#x20; +--> Tool calls

&#x20;         |

&#x20;         v

&#x20;     Parse JSON arguments

&#x20;         |

&#x20;         v

&#x20;     MCPToolRegistry validates and dispatches

&#x20;         |

&#x20;         v

&#x20;     Tool result is appended as a tool message

&#x20;         |

&#x20;         v

&#x20;     Send the updated conversation back to the model

```



The loop is limited to eight completion/tool rounds to prevent an unbounded

tool cycle.



\### Phase G: Response and persistence



After the model returns its final text:



1\. Browser contexts are closed.

2\. Token usage is added to the active session.

3\. The assistant response is appended to chat memory.

4\. The session is saved to `config/chat\_sessions.json`.

5\. The response is rendered in the terminal.

6\. Optional speech output is started.



\## 5. File and permission workflow



\### Read operations



```text

User/model requests a file

&#x20; |

&#x20; +--> my\_files document request

&#x20; |       |

&#x20; |       v

&#x20; |   local document extractor

&#x20; |

&#x20; +--> external/project file request

&#x20;         |

&#x20;         v

&#x20;     read-only Filesystem MCP

```



When a user includes a quoted or recognizable absolute file path in a prompt,

the local feature layer can read supported document formats directly and attach

the content as read-only context. This is separate from all write and

execution guards.



\### Write operations



```text

Model requests generated file

&#x20; |

&#x20; v

filesystem-model-output or write\_file\_tool

&#x20; |

&#x20; v

model\_out\_dir path validation

&#x20; |

&#x20; +--> valid: create parent folders and write

&#x20; |

&#x20; +--> invalid: return security error

```



\### Code execution



```text

Model proposes code

&#x20; |

&#x20; v

Static AST and regular-expression guardrails

&#x20; |

&#x20; +--> unsafe import/call/mutation: reject

&#x20; |

&#x20; +--> safe candidate: ask user for approval

&#x20;                 |

&#x20;                 +--> reject: no execution

&#x20;                 |

&#x20;                 +--> approve: run with project cwd and timeout

```



The execution mechanism is a guarded subprocess, not a complete operating-system

sandbox. Strong isolation would require a container, restricted account, or

dedicated sandbox service.



\## 6. Persistence workflow



\### Session state



`config/chat\_sessions.json` stores:



\- Active session ID

\- Session names

\- Creation timestamps

\- Recent conversation memory

\- Token totals



When sessions load, oversized document content is replaced by a marker so entire

source documents are not repeatedly copied into persistent chat history.



\### Long-term memory



The model extracts selected user facts such as:



\- Name

\- Location

\- College year

\- Technology stack

\- Interests



Those facts are stored in `config/echo\_memory.json` and injected into future

system prompts when available.



\## 7. Folder architecture



```text

AMIGO CLI v1/

|

|-- main.py                         Active entry point

|-- app.py                          Legacy duplicate implementation

|-- .mcp.json                       MCP server definitions

|-- .env                            Local secrets; do not submit

|

|-- core/

|   |-- paths.py                    Stable project directories

|   |-- config\_manager.py           JSON configuration and persistence

|   |-- tools.py                    Search, documents, files, execution

|   |-- security.py                 AST, regex, and path guardrails

|   |-- orchestrator.py             Difficulty classification and dispatch

|   |-- sub\_agent.py                Worker-agent implementations

|   |-- mcp\_client.py               MCP JSON-RPC bridge

|   `-- tray.py                     Optional tray integration

|

|-- config/

|   |-- models\_config.json

|   |-- personalities\_config.json

|   |-- chat\_sessions.json

|   `-- echo\_memory.json

|

|-- my\_files/                       User reference documents

|-- model\_out\_dir/                  Generated model output

|   `-- screenshots/                Playwright screenshots

|

|-- tests/

|   |-- test\_security\_boundaries.py

|   `-- test\_model\_config.py

|

`-- DOCUMENTATION/

&#x20;   |-- AMIGO\_CLI\_Project\_Documentation.md

&#x20;   |-- AMIGO\_CLI\_Project\_Documentation.docx

&#x20;   `-- AMIGO\_CLI\_Architecture\_and\_Workflow.md

```



\## 8. Error handling



The application reports errors through:



\- Terminal status messages

\- MCP startup warnings

\- MCP tool error messages returned to the model

\- File-operation error results

\- Security-block messages

\- Model configuration validation errors

\- Session and persistence fallbacks



MCP failures are isolated per server during startup. If one server fails, the

others can still be discovered and used, while the startup warning is displayed.



\## 9. Design strengths



\- Modular separation between UI, tools, orchestration, persistence, and security

\- Conservative multi-agent dispatch

\- Configurable model providers without Python edits

\- Headless browser automation

\- Explicit output-folder policy

\- Read-only external filesystem access

\- User confirmation before code execution

\- Session persistence and long-term memory

\- Automated regression tests for key boundary rules



\## 10. Current limitations



\- Static code checks cannot guarantee complete OS-level isolation.

\- The application depends on provider availability and valid model IDs.

\- PDF extraction quality depends on whether the PDF contains selectable text.

\- Voice recognition requires a working microphone and speech-recognition service.

\- MCP servers require Node.js and `npx`.

\- The legacy `app.py` remains in the repository but is not the active modular

&#x20; execution path.



\## 11. Short operational summary



```text

Start

&#x20; -> Load configuration and memory

&#x20; -> Start MCP services

&#x20; -> Accept text/voice input

&#x20; -> Handle local command or prepare model context

&#x20; -> Classify task difficulty

&#x20; -> Optionally run parallel specialist agents

&#x20; -> Ask active model for response

&#x20; -> Execute approved MCP tool calls when requested

&#x20; -> Generate final answer

&#x20; -> Save session and memory state

&#x20; -> Close browser contexts

&#x20; -> Wait for next request

```




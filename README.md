# Local AI Agent with Ollama

A command-line AI assistant written in Python that uses an Ollama model through an OpenAI-compatible API. It supports streamed responses, multi-step tool calls, persistent chat history, and retrieval-augmented generation (RAG) over a local company knowledge base.

The agent does not use an agent framework. The chat loop, tool schema, tool execution, memory, and RAG flow are implemented directly in Python.

## Features

- Streaming chat responses
- Multi-step function/tool calling
- Current time tool
- Weather lookup through `wttr.in`
- Hard-coded demo city lookup
- Local knowledge-base search
- Cosine-similarity RAG over `.txt` documents
- Saved JSON chat history with resume and new-chat commands
- Debug output for tool calls and retrieved chunks
- Standalone examples for streaming, embeddings, search, chunking, and ChromaDB

## How It Works

`simple_ai_assistant.py` sends the chat history and tool definitions to the configured model. If the model requests a tool, `tools.py` executes the matching Python function and adds the result to the conversation. The loop repeats for up to five model rounds for a single user message.

```text
User
  |
  v
simple_ai_assistant.py
  |-- memory.py ---------------- saves and loads chat JSON
  |-- OpenAI-compatible API -> Ollama chat model
  |-- tools.py ---------------- runs requested tools
        |-- get_current_time
        |-- get_city_weather
        |-- get_user_city
        |-- search_knowledge_base
              |
              v
           rag.py -> vector_store.json
```

The model is not given direct access to the filesystem or Python functions. It only produces function-call requests, and the local Python code executes them.

## Project Structure

```text
python-ai-agent-ollama/
├── simple_ai_assistant.py  Main chat loop and CLI commands
├── tools.py                Tool definitions and tool dispatcher
├── rag.py                  Document loading, chunking, embeddings, and search
├── memory.py               Saved-chat list, load, and save operations
├── Utilities/
│   └── utilities.py        Optional debug logging
├── knowledge_base/         Local .txt documents
├── Code Snippets/          Standalone experiments
├── requirements.txt        Python dependencies
├── .env.example            Example configuration
└── .gitignore
```

## Requirements

- Python 3.10 or newer
- Ollama running locally
- A chat model that supports function/tool calling
- An Ollama embedding model
- Internet access for the weather tool

The default setup uses:

- `qwen3:4b` as the chat model
- `nomic-embed-text` as the embedding model

## Quick Start

### 1. Install and start Ollama

Install Ollama, start its service, and download the required models:

```bash
ollama pull qwen3:4b
ollama pull nomic-embed-text
```

### 2. Create a virtual environment

**Windows:**

```powershell
python -m venv .venv
.venv\Scripts\activate
```

**macOS/Linux:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install Python dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure environment variables

Create `.env` from the example:

**Windows:**

```powershell
copy .env.example .env
```

**macOS/Linux:**

```bash
cp .env.example .env
```

The default values are:

```env
BASE_URL=http://127.0.0.1:11434/v1
API_KEY=dummy
MODEL=qwen3:4b
DEBUG=False
EMBED_MODEL=nomic-embed-text
KB_FOLDER=knowledge_base
VECTOR_STORE_FILE=knowledge_base/vector_store.json
CHAT_FOLDER=chat_history
```

Ollama ignores `API_KEY`, but the OpenAI Python client requires one.

### 5. Run the agent

Run the program from the project root:

```bash
python simple_ai_assistant.py
```

At startup, the program offers to continue the most recently saved chat. Inside the chat:

- type `exit`, `quit`, `bye`, or `goodbye` to stop
- type `/new` to start a new chat
- type `/chats` to list saved chats
- type `/resume` to select a saved chat

Chat commands are handled by local Python code and are not sent to the model.

## Configuration

| Variable | Purpose |
|---|---|
| `BASE_URL` | OpenAI-compatible API endpoint |
| `API_KEY` | Client API key; use a dummy value for Ollama |
| `MODEL` | Ollama chat model name |
| `DEBUG` | Set exactly to `True` to print grey debug messages |
| `EMBED_MODEL` | Ollama embedding model name |
| `KB_FOLDER` | Folder containing knowledge-base `.txt` files |
| `VECTOR_STORE_FILE` | Path for the generated JSON vector store |
| `CHAT_FOLDER` | Folder containing saved chat JSON files |

Use `NAME=value` syntax without quotes around values.

## Available Tools

| Tool | Parameters | Result |
|---|---|---|
| `get_current_time` | None | Local date and time |
| `get_city_weather` | `city` | One-line weather text from `wttr.in` |
| `get_user_city` | None | Hard-coded `Bangalore` demo value |
| `search_knowledge_base` | `query` | Up to two relevant chunks with source and chunk ID |

Tool failures are converted into text and sent back to the model so that the CLI can continue.

## RAG Knowledge Base

The included knowledge base contains sample Zorvex Technologies documents for:

- hiring
- HR
- leave
- medical benefits
- incident reports
- technology processes
- travel expenses

On the first import of `rag.py`, the program:

1. Reads every `.txt` file in `KB_FOLDER`.
2. Splits each file on blank lines.
3. Embeds every chunk with `EMBED_MODEL`.
4. Saves the text, embeddings, source filenames, and chunk IDs to `VECTOR_STORE_FILE`.

Later starts load the JSON file directly. Search embeds the user's query and compares it against every stored chunk using NumPy cosine similarity. The most relevant two chunks are returned to the model.

Example questions:

```text
How many weeks of maternity leave do I get?
What was the root cause of INC-4021?
How do I claim travel expenses?
What is the company's policy on pet insurance?
```

When a company question is outside the retrieved documents, the system prompt instructs the model to say that it could not find the answer in the knowledge base.

### Add a Document

1. Add a `.txt` file to `knowledge_base/`.
2. Separate document sections with blank lines.
3. Delete `knowledge_base/vector_store.json`.
4. Start the agent again to rebuild and embed the index.

The vector store is stale if a document or `EMBED_MODEL` changes.

## Persistent Chats

Each completed turn is saved as a JSON file in `CHAT_FOLDER`. The filename is based on the chat start time.

```json
{
  "title": "First message",
  "updated": "2026-11-23 01:16:11",
  "messages": [
    {
      "role": "user",
      "content": "Example question"
    }
  ]
}
```

The system prompt is not stored. It is rebuilt from `simple_ai_assistant.py` each time, so changes to the prompt apply to resumed chats.

Chat files are written through a temporary `.tmp` file and replaced atomically. The chat title is the first 40 characters of the first user message.

If a saved JSON file is missing or malformed, loading or listing it may fail; remove the damaged file if this occurs.

## Debugging

Set `DEBUG=True` in `.env`, then restart the agent. Debug output shows:

- model tool-call steps
- tool names and arguments
- tool results
- knowledge-base search scores and sources

## Standalone Code Snippets

The `Code Snippets` folder contains independent scripts:

| Script | Demonstrates |
|---|---|
| `stream_test.py` | Streaming text from Ollama |
| `stream_tools_test.py` | Reassembling streamed tool calls |
| `embedding_test.py` | Creating text embeddings |
| `semantic_search_test.py` | Cosine similarity search |
| `doc_search.py` | Embedding and searching complete documents |
| `chunk_search.py` | Chunking, embedding, and saving a JSON vector store |
| `chroma_simple.py` | Persistent ChromaDB storage and search |

Run snippets from the project root, for example:

```bash
python "Code Snippets/stream_test.py"
python "Code Snippets/embedding_test.py"
python "Code Snippets/chunk_search.py"
```

ChromaDB is not in `requirements.txt` because it is only needed by `chroma_simple.py`. To try that example:

```bash
pip install chromadb
python "Code Snippets/chroma_simple.py"
```

The main agent uses `rag.py` and `vector_store.json`; it does not currently use ChromaDB.

## Adding a Tool

To add a tool, update `tools.py`:

1. Implement a Python function.
2. Add an OpenAI-compatible function schema to `TOOLS`.
3. Register the function name in `FUNCTION_NAME_TO_FUNCTION_MAP`.

When the tool should only be used in a specific situation, also update `SYSTEM_MESSAGE` in `simple_ai_assistant.py` so the model knows about it.

## Limitations

- Chat history is sent in full and is not trimmed or summarized.
- Chat JSON files are stored as plain text.
- The saved chat folder is created by the application but is not ignored by the current `.gitignore`.
- `get_user_city` is a hard-coded demonstration value.
- Weather requests require internet access and depend on `wttr.in`.
- Search is a brute-force scan and is intended for a small knowledge base.
- Search always compares against every chunk and has no relevance threshold.
- Chunking is based on blank lines, so different document formats may need another strategy.
- Pressing `Ctrl+C` during a stream stops text generation; any partial tool-call request is discarded.
- Multiple instances of the agent can overwrite the same saved chat file.
- Small local models may make incorrect or incomplete tool calls.

## Troubleshooting

**The agent cannot connect**

- Confirm Ollama is running.
- Confirm `BASE_URL` ends in `/v1`.
- Run `ollama list` and compare `MODEL` with the installed model names.

**The embedding index is not being created**

- Confirm `nomic-embed-text` is installed:

  ```bash
  ollama pull nomic-embed-text
  ```

- Confirm `KB_FOLDER` and `VECTOR_STORE_FILE` exist in `.env`.
- Delete `VECTOR_STORE_FILE` to rebuild it.

**The model does not call tools**

- Confirm the selected model supports tool/function calling.
- Set `DEBUG=True` to inspect model and tool activity.
- Update the system prompt when changing or adding tools.

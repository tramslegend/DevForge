# 🔨 DevForge

> A local-first AI assistant for your codebase. No cloud, no API keys, just you and your repo.

**DevForge** keeps your repository context synced and lets you ask questions about your code instantly - all running locally with Ollama.

Born from frustration with cloud tools and constant copy-pasting. Built during weekend vibe-coding sessions (name inspired by Skyrim armor crafting 😅).

## ✨ Features

- **📚 Incremental Context Updates** - Only re-embeds changed files (hash-based delta detection)
- **🧠 Smart Retrieval** - Cosine similarity search with deduplication and freshness scoring
- **🔒 100% Local** - Everything runs on your machine. No API keys, no data leaves your laptop
- **⚡ Fast Queries** - Vector embeddings for semantic code search
- **🎯 Cited Answers** - Every response includes source references `[file:start-end]`
- **🛡️ Privacy First** - Your code stays yours

## 🚀 Quick Start

### Prerequisites

1. **Install Ollama** - [ollama.ai](https://ollama.ai)
   ```bash
   # Start Ollama server
   ollama serve
   ```

2. **Pull Required Models**
   ```bash
   ollama pull gemma3:latest
   ollama pull nomic-embed-text:latest
   ```

### Installation

```bash
# Clone the repo
git clone <your-repo-url>
cd devforge

# Install dependencies
npm install

# Make it globally available (optional)
npm link
```

## 📖 Usage

### 1. Initialize Your Repository

Navigate to any Git repository and run:

```bash
devforge read
```

This will:
- ✅ Check Ollama is running and models are available
- ✅ Create `.devforge/` directory (config, ignore patterns, manifest, index)
- ✅ Scan your repo respecting ignore patterns
- ✅ Chunk files into ~2000 character segments
- ✅ Generate embeddings for all code
- ✅ Store vectors in local index

**Output:**
```
🔎 Preflight: Ollama & models
✅ Ollama running, required models found.
🔧 Initializing DevForge at: /path/to/repo/.devforge
✅ Created .devforge/config.json
✅ Created .devforge/ignore
📚 Scanning repository (respecting config & ignore)…
✅ Refreshed manifest: 42 files
   Languages: js(15), json(8), md(3)
🧠 Embedding repo content (added/changed files only)…
   · (1/42) src/chat.js … ok (3 chunks)
   · (2/42) src/read.js … ok (5 chunks)
   ...
✅ Embedded 42 file(s), 156 chunk(s)
```

### 2. Ask Questions

```bash
# Simple query
devforge ask "where is JWT verified?"

# Show sources
devforge ask "how does the embedding work?" --show-sources

# Use more context chunks
devforge ask "explain the authentication flow" -k 20

# Skip refresh if repo hasn't changed
devforge ask "what database is used?" --no-refresh

# Longer responses
devforge ask "summarize the API architecture" --max-tokens 1024
```

**Example Output:**
```
🔎 Retrieving relevant code… ok (12 chunk(s))

JWT verification happens in the authentication middleware. The token is 
extracted from the Authorization header and verified using the JWT_SECRET 
from the environment [src/middleware/auth.js:23-45]. If verification fails, 
it returns a 401 Unauthorized response.

Sources:
  - src/middleware/auth.js:23-45
  - src/config/auth.js:10-15
```

### 3. Refresh After Changes

Made changes to your code? Just run:

```bash
devforge read
```

DevForge will:
- Detect added, changed, and removed files (using SHA-256 hashes)
- Only re-embed the delta
- Update the index incrementally

**Incremental Update Example:**
```
📚 Scanning repository…
✅ Refreshed manifest: 43 files
   Δ changes → added: 1, changed: 2, removed: 0
   ➕ Added:
     src/new-feature.js
   ✏️ Changed:
     src/auth.js
     src/config.js
🧠 Embedding repo content (added/changed files only)…
   · (1/3) src/new-feature.js … ok (4 chunks)
   · (2/3) src/auth.js … ok (3 chunks)
   · (3/3) src/config.js … ok (2 chunks)
✅ Embedded 3 file(s), 9 chunk(s)
```

### 4. Force Rebuild (if needed)

```bash
devforge read --force
```

Clears cache and rebuilds the entire index from scratch.

## ⚙️ Configuration

DevForge creates `.devforge/config.json` on first run:

```json
{
  "airgap": false,
  "max_file_kb": 256,
  "models": {
    "chat": "gemma3:latest",
    "embed": "nomic-embed-text:latest"
  },
  "paths": {
    "include": ["**/*"],
    "exclude": [
      "node_modules/**",
      "dist/**",
      ".git/**",
      "*.lock",
      "*.bin",
      "*.jpg",
      "*.png",
      "*.pdf",
      ".env",
      "**/*.key",
      "**/*.pem"
    ]
  },
  "retrieval": {
    "bm25_k": 80,
    "vec_k": 80,
    "final_k": 20
  }
}
```

### Customize Models

Edit `config.json` to use different models:

```json
{
  "models": {
    "chat": "llama3.2:latest",      // Bigger model if you have RAM
    "embed": "nomic-embed-text:latest"
  }
}
```

### Add Custom Ignore Patterns

Edit `.devforge/ignore` to exclude specific files:

```
# Custom ignores
secrets.txt
internal/**
*.tmp
```

## 🔧 Commands

### `devforge hello`
Test that DevForge is working.

### `devforge read`
Initialize or refresh repository context.

**Options:**
- `--force` - Clear cache and rebuild from scratch

### `devforge ask <question>`
Query your codebase.

**Options:**
- `-k, --topk <n>` - Number of chunks to retrieve (default: 12)
- `--max-tokens <n>` - Max tokens for answer (default: 512)
- `--model <name>` - Override chat model
- `--show-sources` - Print source references
- `--no-refresh` - Skip context refresh (faster, use existing index)

## 🏗️ How It Works

### 1. **Scanning**
DevForge uses `fast-glob` to find files matching your include/exclude patterns, computing SHA-256 hashes for change detection.

### 2. **Chunking**
Files are split into ~2000 character chunks with small overlap to preserve context across boundaries.

### 3. **Embedding**
Each chunk is embedded using `nomic-embed-text` (produces 768-dimensional vectors). These are stored in `.devforge/index.json`.

### 4. **Retrieval**
When you ask a question:
- Your question is embedded into the same vector space
- Cosine similarity scores all chunks
- Top-k most relevant chunks are retrieved
- Deduplication ensures diverse file coverage

### 5. **Generation**
The context (retrieved chunks) + your question are sent to the chat model, which generates an answer with source citations.

### 6. **Delta Updates**
On subsequent `devforge read`:
- Compares current file hashes to previous manifest
- Only re-embeds changed files
- Prunes removed files from index
- Keeps everything in sync

## 💡 Tips

**Performance:**
- Start with `gemma3:latest` (efficient, good quality)
- Upgrade to `llama3.2:latest` or `mixtral` if you have 16GB+ RAM
- Use `--no-refresh` for quick queries when repo hasn't changed

**Better Results:**
- Be specific in questions: "How does X validate Y?" vs "Tell me about X"
- Increase `-k` for complex queries that span multiple files
- Use `--show-sources` to verify context relevance

**Keep Context Fresh:**
- Run `devforge read` after pulling changes
- Consider adding a git post-merge hook:
  ```bash
  #!/bin/bash
  # .git/hooks/post-merge
  devforge read --silent
  ```

## 🐛 Troubleshooting

**"Ollama is not running"**
```bash
# Start Ollama server
ollama serve
```

**"Missing model(s)"**
```bash
ollama pull gemma3:latest
ollama pull nomic-embed-text:latest
```

**"No vectors found"**
```bash
# Initialize the repo first
devforge read
```

**Slow embeddings on large repos:**
- Adjust `max_file_kb` in config to skip huge files
- Add more patterns to `.devforge/ignore`
- Embeddings are one-time cost; subsequent updates are incremental

**Poor answer quality:**
- Try increasing `-k` to retrieve more context
- Use `--show-sources` to check if relevant code was retrieved
- Consider using a larger chat model

## 📂 Project Structure

```
.devforge/
├── config.json      # Configuration
├── ignore           # Additional ignore patterns
├── manifest.json    # File metadata + hashes
└── index.json       # Vector embeddings

src/
├── cli.js          # Command-line interface
├── read.js         # Init, scan, embed
├── chat.js         # Query & retrieval
├── scan.js         # File scanning + hashing
├── embeddings.js   # Chunking + embedding
└── ollama.js       # Ollama API checks
```

## 🤝 Contributing

This started as a weekend project, but contributions are welcome!

1. Fork the repo
2. Create a feature branch
3. Make your changes
4. Submit a PR

## 📜 License

ISC

## 🎮 Origin Story

Built out of frustration with cloud tools that never stayed in sync with my local repo. Wanted something dead simple that just works - no APIs, no copy-pasting, no nonsense.

The name "DevForge" came to me while crafting armor in Skyrim. Sometimes the best tools are the ones you build while vibe-coding on weekends.

---

**Made with ☕ and weekend energy**
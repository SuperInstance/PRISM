# PRISM

[![TypeScript](https://img.shields.io/badge/TypeScript-5.7-blue.svg)](https://www.typescriptlang.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![Cloudflare Workers](https://img.shields.io/badge/Cloudflare-Workers-orange.svg)](https://workers.cloudflare.com/)

**Lightning-fast semantic code search powered by vector embeddings.**

> Search by meaning, not keywords. PRISM indexes your entire codebase and lets AI assistants find the exact code they need — in milliseconds.

---

## Overview

PRISM is a semantic code search engine and Claude Code plugin that gives AI assistants instant, accurate memory of your codebase. It automatically indexes your code into vector embeddings, enabling natural-language search across millions of tokens of source code.

The core problem: Claude Code can only see ~128K tokens at once, but real codebases span millions. PRISM bridges this gap by converting code into searchable vector representations using embedding models (BGE-Small 384d via Cloudflare Workers AI or Nomic via Ollama), then performing fast similarity search to retrieve the most relevant code chunks for any query.

PRISM runs in two modes:
- **Claude Code Plugin** — install once, auto-starts with every project, zero config
- **CLI** — `prism index ./src` and `prism search "user authentication"` for manual use

**Result:** 10x faster debugging, instant onboarding, and 90% lower API costs by sending only relevant context.

---

## Architecture

PRISM is built on a modular, layered architecture designed for extensibility and performance:

```
┌─────────────────────────────────────────────────────┐
│                  Claude Code Plugin                  │
│            (claude-code-plugin/daemon/)              │
│  File Watcher → Indexer → Search Server → MCP Tools │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP / MCP
┌──────────────────────▼──────────────────────────────┐
│              Cloudflare Worker (API Layer)            │
│           src/worker-vectorize.ts                    │
│  ┌─────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │ Router   │→ │ Handlers │→ │ Response Helpers  │  │
│  └─────────┘  └──────────┘  └───────────────────┘  │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  Core Engine                         │
│                                                      │
│  ┌──────────────┐   ┌────────────────────────┐     │
│  │ Indexer       │   │ Embedding Service       │     │
│  │ Orchestrator  │── │ Cloudflare AI (primary) │     │
│  │ (6-stage      │   │ Ollama (fallback)       │     │
│  │  pipeline)    │   └────────────────────────┘     │
│  └──────┬───────┘                                   │
│         │                                            │
│  ┌──────▼───────┐   ┌────────────────────────┐     │
│  │ Vector DB    │   │ Relevance Scorer       │     │
│  │ MemoryVector │   │ 5-signal weighted rank  │     │
│  │ HNSWIndex    │   │ Semantic + Proximity +  │     │
│  │ SQLite       │   │ Symbol + Recency + Freq │     │
│  │ D1VectorDB   │   └────────────────────────┘     │
│  └──────────────┘                                   │
│                                                      │
│  ┌──────────────┐   ┌────────────────────────┐     │
│  │ Model Router │   │ Token Optimizer        │     │
│  │ Cost-aware   │   │ Adaptive Compressor    │     │
│  │ AI selection │   │ Intent Detection       │     │
│  └──────────────┘   │ Chunk Selection        │     │
│                      └────────────────────────┘     │
└─────────────────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                Storage Layer                         │
│  Cloudflare Vectorize │ D1 │ KV │ R2 │ SQLite      │
└─────────────────────────────────────────────────────┘
```

### Core Modules

| Module | Path | Purpose |
|---|---|---|
| **Indexer Orchestrator** | `src/indexer/IndexerOrchestrator.ts` | 6-stage pipeline: collect → filter → chunk → embed → store → metadata |
| **Embedding Service** | `src/embeddings/EmbeddingService.ts` | Multi-provider embedding gen (Cloudflare + Ollama) with rate limiting |
| **Vector DB** | `src/vector-db/` | In-memory, HNSW, SQLite, and D1-backed vector storage |
| **Relevance Scorer** | `src/scoring/` | 5-signal weighted scoring: semantic (40%), proximity (25%), symbol (20%), recency (10%), frequency (5%) |
| **Model Router** | `src/model-router/ModelRouter.ts` | Cost-aware AI model selection (Ollama → Cloudflare → Haiku → Sonnet → Opus) |
| **Adaptive Compressor** | `src/compression/AdaptiveCompressor.ts` | 4-level progressive compression (light → medium → aggressive → signature-only) |
| **Token Optimizer** | `src/token-optimizer/` | Intent detection, chunk selection, budget-aware context assembly |
| **Cloudflare Worker** | `src/worker-vectorize.ts` | Serverless API with Vectorize, D1, Workers AI bindings |

### Indexing Pipeline

1. **File Collection** — Recursive scan with glob include/exclude patterns
2. **Incremental Filtering** — SHA-256 checksum + mtime comparison to skip unchanged files (~90x speedup)
3. **Chunking** — Language-aware parsing (WASM Rust indexer for Tree-sitter support)
4. **Embedding Generation** — Batch processing via Cloudflare AI (384d BGE-Small) or Ollama (768d Nomic)
5. **Vector Storage** — Batch insert into vector database with metadata
6. **Metadata Update** — File records, checksums, statistics for next run

### Infrastructure (Cloudflare)

- **Vectorize** — Managed ANN vector index (cosine similarity, 384 dimensions)
- **D1** — SQLite database for file metadata, chunk content, and checksums
- **Workers AI** — Serverless embedding generation (@cf/baai/bge-small-en-v1.5)
- **KV** — Key-value storage for index metadata
- **R2** — Object storage for large artifacts

---

## Features

### Semantic Code Search
- Search by meaning using vector embeddings, not keyword matching
- Multi-factor relevance scoring with 5 weighted signals
- Configurable filters (language, path prefix, date range, minimum score)

### Multi-Provider Embeddings
- **Cloudflare Workers AI** (primary) — BGE-Small 384d, free tier, ~100ms/batch
- **Ollama** (fallback) — Nomic 768d, unlimited local inference
- Automatic fallback between providers with rate limit management

### Cost-Optimized Model Routing
- Automatic model selection based on query complexity and token count
- Decision tree: Ollama (free) → Cloudflare (free) → Haiku ($0.25/M) → Sonnet ($3/M) → Opus ($15/M)
- Budget tracking and cost estimation before each request

### Adaptive Token Compression
- 4-level progressive compression: light (1.2x) → medium (3x) → aggressive (15x) → signature-only (30x)
- Preserves imports, types, and structural elements
- Language-aware signature extraction for TypeScript, JavaScript, Python

### Incremental Indexing
- SHA-256 checksum-based change detection (handles git operations correctly)
- Only re-indexes modified or new files
- Deleted file detection and cleanup
- ~90x speedup for typical development workflows (10 files vs 10K)

### Claude Code Plugin
- **Zero configuration** — install once, works for all projects
- **Auto-start** — launches when you open any project
- **Auto-update** — re-indexes on file changes via file watcher
- **MCP integration** — Model Context Protocol tools for Claude Code
- **Project isolation** — separate index per project

### Cross-Platform
- macOS, Linux, Windows
- Supports 15+ programming languages
- Local-first architecture (works offline after indexing)

---

## Quick Start

### Option 1: Claude Code Plugin (Recommended)

```bash
/plugin install https://github.com/SuperInstance/prism/tree/main/claude-code-plugin
```

That's it. PRISM auto-starts, auto-indexes, and auto-updates.

### Option 2: CLI

```bash
# Install globally
npm install -g @claudes-friend/prism

# Index your project
prism index ./src

# Search by meaning
prism search "user authentication"

# Check index stats
prism stats

# Health check
prism health
```

### Option 3: Cloudflare Worker

```bash
# Clone the repo
git clone https://github.com/SuperInstance/prism.git
cd prism

# Install dependencies
npm install

# Create Cloudflare resources and deploy
npm run deploy:full

# Or deploy individual components
npm run deploy:prod
```

---

## API

### REST Endpoints (Cloudflare Worker)

#### `GET /health`
Health check with Vectorize and D1 status.

```json
{
  "status": "healthy",
  "version": "0.3.1",
  "vectorize": { "dimensions": 384, "metric": "cosine", "count": 12450 },
  "d1_initialized": true
}
```

#### `POST /api/index`
Index code files with automatic embedding generation.

```bash
curl -X POST https://your-worker.your-subdomain.workers.dev/api/index \
  -H "Content-Type: application/json" \
  -d '{
    "files": [
      { "path": "src/auth.ts", "content": "export function login() { ... }" }
    ],
    "options": { "incremental": true }
  }'
```

```json
{ "files": 1, "chunks": 12, "errors": 0, "duration": 342 }
```

#### `POST /api/search`
Semantic code search with filters.

```bash
curl -X POST https://your-worker.your-subdomain.workers.dev/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "how does user authentication work",
    "limit": 10,
    "minScore": 0.5,
    "filters": { "language": "typescript", "pathPrefix": "src/" }
  }'
```

```json
{
  "results": [
    {
      "id": "src/auth.ts:0",
      "filePath": "src/auth.ts",
      "content": "export function login(user: User): Promise<Session> { ... }",
      "startLine": 1,
      "endLine": 15,
      "language": "typescript",
      "score": 0.92
    }
  ],
  "query": "how does user authentication work",
  "total": 1
}
```

#### `GET /api/stats`
Index statistics and metadata.

```json
{
  "chunks": 12450,
  "files": 342,
  "vectorize": { "dimensions": 384, "metric": "cosine", "count": 12450 }
}
```

### CLI Commands

| Command | Description |
|---|---|
| `prism index <path>` | Index a directory recursively |
| `prism search <query>` | Semantic code search |
| `prism stats` | Show index statistics |
| `prism health` | Check system health |

### Search Filters

| Filter | Type | Description |
|---|---|---|
| `language` | `string` | Filter by programming language |
| `pathPrefix` | `string` | Filter by directory prefix |
| `filePath` | `string` | Pattern-match file paths |
| `createdAfter` | `number` | Unix timestamp (lower bound) |
| `createdBefore` | `number` | Unix timestamp (upper bound) |
| `minScore` | `number` | Minimum relevance score (0–1) |
| `limit` | `number` | Max results (default: 10, max: 100) |

---

## Development

### Prerequisites

- **Node.js** >= 18.0.0
- **TypeScript** 5.7+
- **Wrangler CLI** (for Cloudflare Workers dev/deploy)

### Setup

```bash
# Clone and install
git clone https://github.com/SuperInstance/prism.git
cd prism
npm install

# Build WASM indexer (optional, requires Rust)
npm run build:wasm

# Build TypeScript
npm run build:ts

# Or build everything
npm run build
```

### Available Scripts

| Script | Description |
|---|---|
| `npm run dev` | Start local Cloudflare Workers dev server |
| `npm run build` | Full build (WASM + TypeScript) |
| `npm run test` | Run test suite (Vitest) |
| `npm run test:coverage` | Run tests with coverage report |
| `npm run lint` | Lint source code |
| `npm run lint:fix` | Lint and auto-fix |
| `npm run format` | Format with Prettier |
| `npm run typecheck` | Type-check without emitting |
| `npm run deploy` | Deploy to Cloudflare Workers |
| `npm run deploy:full` | Create resources + deploy |

### Project Structure

```
prism/
├── src/
│   ├── worker-vectorize.ts      # Cloudflare Worker entry point
│   ├── shared/utils.ts          # Shared utilities (chunking, hashing, CORS)
│   ├── core/                    # Core interfaces, types, utilities
│   │   ├── interfaces/          # IFileSystem, IIndexer, IEmbeddingService, etc.
│   │   ├── types/               # CodeChunk, SearchResults, PrismError
│   │   ├── services/            # FileSystem implementation
│   │   └── utils/               # Embedding math, token counting, file utils
│   ├── indexer/                 # Indexing pipeline
│   │   ├── IndexerOrchestrator.ts  # 6-stage orchestration
│   │   ├── WasmIndexer.ts       # WASM/Rust tree-sitter chunking
│   │   ├── IndexStorage.ts      # Local JSON metadata storage
│   │   ├── D1IndexStorage.ts    # Cloudflare D1 metadata storage
│   │   ├── SQLiteIndexStorage.ts # SQLite metadata storage
│   │   ├── HNSWIndex.ts         # HNSW approximate nearest neighbor
│   │   ├── ProgressReporter.ts  # Indexing progress tracking
│   │   └── checksum.ts          # SHA-256 checksum utilities
│   ├── embeddings/              # Embedding generation
│   │   └── EmbeddingService.ts  # Multi-provider (Cloudflare + Ollama)
│   ├── vector-db/               # Vector storage backends
│   │   ├── MemoryVectorDB.ts    # In-memory brute-force
│   │   ├── HNSWIndex.ts         # HNSW-accelerated search
│   │   ├── SQLiteVectorDB.ts    # SQLite-backed persistent storage
│   │   └── D1VectorDB.ts        # Cloudflare D1-backed storage
│   ├── scoring/                 # Relevance scoring
│   │   ├── scores/              # RelevanceScorer (5-signal weighted)
│   │   └── features/            # semantic, proximity, symbol, recency, frequency
│   ├── model-router/            # Cost-aware AI model selection
│   │   ├── ModelRouter.ts       # Decision tree: free → cheap → balanced → premium
│   │   ├── ComplexityAnalyzer.ts # Query complexity scoring
│   │   ├── BudgetTracker.ts     # Usage and cost tracking
│   │   ├── OllamaClient.ts      # Local LLM client
│   │   └── CloudflareClient.ts  # Cloudflare Workers AI client
│   ├── compression/             # Token optimization
│   │   └── AdaptiveCompressor.ts # 4-level progressive compression
│   ├── token-optimizer/         # Context optimization
│   │   ├── TokenOptimizer.ts    # Budget-aware context assembly
│   │   ├── IntentDetector.ts    # Query intent classification
│   │   ├── ChunkSelector.ts     # Optimal chunk selection
│   │   └── SimpleTokenCounter.ts # Token estimation
│   ├── config/                  # Configuration management
│   │   └── ConfigurationService.ts
│   └── cli/                     # CLI types and commands
├── claude-code-plugin/          # Claude Code plugin
│   ├── daemon/                  # Background daemon
│   │   ├── server.js            # HTTP search server
│   │   ├── file-watcher.js      # File change detection
│   │   ├── file-indexer.js      # Local JSON indexer
│   │   ├── project-detector.js  # Auto project detection
│   │   └── search-cache.js      # Response caching
│   ├── commands/                # Claude Code slash commands
│   ├── scripts/                 # Install/uninstall scripts
│   └── test/                    # Plugin tests
├── prism/                       # Standalone CLI submodule
│   ├── src/                     # CLI-specific source
│   │   ├── core/                # PrismEngine
│   │   ├── mcp/                 # MCP server integration
│   │   ├── ollama/              # Ollama integration
│   │   └── prism-indexer/       # Rust/WASM indexer
│   └── tests/                   # Standalone CLI tests
├── tests/                       # Main test suite
│   ├── unit/                    # Unit tests
│   ├── integration/             # Integration tests
│   └── scoring/                 # Scoring benchmarks
├── migrations/                  # D1 database migrations
├── docs/                        # Extended documentation
└── wrangler.toml                # Cloudflare Workers config
```

### Testing

```bash
# Run all tests
npm test

# Run with coverage
npm run test:coverage

# Run specific test suites
npm run test:worker              # Worker integration tests
npm run test:worker              # Worker integration tests

# Run tests for the plugin
cd claude-code-plugin && npm test
```

### Code Quality

```bash
# Type-check
npm run typecheck

# Lint
npm run lint

# Auto-fix lint issues
npm run lint:fix

# Format code
npm run format
```

### Database Migrations

```bash
# Create D1 database
npm run db:create

# Run all migrations
npm run db:migrate

# Backup database
npm run db:backup
```

---

## Technical Details

| Aspect | Specification |
|---|---|
| **Embedding Model** | @cf/baai/bge-small-en-v1.5 (384d) / nomic-embed-text (768d) |
| **Search Latency** | <10ms (Vectorize), ~5ms/1K chunks (MemoryVectorDB) |
| **Memory Usage** | <50MB RAM for typical codebases |
| **CPU Usage** | <1% idle, <5% during indexing |
| **Security** | 100% local option, path traversal protection, CORS restrictions |
| **Languages** | TypeScript, JavaScript, Python, Go, Rust, Java, C#, PHP, Ruby, and more |
| **Platforms** | macOS, Linux, Windows |
| **Node.js** | >= 18.0.0 |

---

## Documentation

| Doc | Description |
|---|---|
| [Installation Guide](./USER_GUIDE.md) | Detailed setup instructions |
| [API Reference](./API_REFERENCE.md) | All commands and options |
| [Troubleshooting](./TROUBLESHOOTING.md) | Common issues and solutions |
| [Configuration](./CONFIGURATION.md) | Advanced settings |
| [Contributing](./CONTRIBUTING.md) | Contribution guidelines |

---

## Contributing

We welcome contributions! See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

---

## License

MIT License — See [LICENSE](./LICENSE) for details.

---

## Links

- **Repository:** https://github.com/SuperInstance/prism
- **Issues:** https://github.com/SuperInstance/prism/issues
- **Version:** 0.3.1 (Production-Ready)

---

<img src="callsign1.jpg" width="128" alt="callsign">

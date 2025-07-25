# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Graphiti is a Python framework for building temporally-aware knowledge graphs designed for AI agents. It enables real-time incremental updates to knowledge graphs without batch recomputation, making it suitable for dynamic environments.

Key features:

- Bi-temporal data model with explicit tracking of event occurrence times
- Hybrid retrieval combining semantic embeddings, keyword search (BM25), and graph traversal
- Support for custom entity definitions via Pydantic models (allows flexible ontology creation)
- Integration with Neo4j and FalkorDB as graph storage backends
- Real-time incremental updates without batch recomputation

## Development Commands

### Main Development Commands (run from project root)

```bash
# Install dependencies
uv sync --extra dev

# Format code (ruff import sorting + formatting)
make format

# Lint code (ruff + pyright type checking)
make lint

# Run tests
make test

# Run all checks (format, lint, test)
make check
```

### Server Development (run from server/ directory)

```bash
cd server/
# Install server dependencies
uv sync --extra dev

# Run server in development mode
uvicorn graph_service.main:app --reload

# Format, lint, test server code
make format
make lint
make test
```

### MCP Server Development (run from mcp_server/ directory)

```bash
cd mcp_server/
# Install MCP server dependencies
uv sync

# Run with Docker Compose
docker-compose up
```

## Code Architecture

### Core Library (`graphiti_core/`)

- **Main Entry Point**: `graphiti.py` - Contains the main `Graphiti` class that orchestrates all functionality
- **Graph Storage**: `driver/` - Database drivers for Neo4j and FalkorDB
- **LLM Integration**: `llm_client/` - Clients for OpenAI, Anthropic, Gemini, Groq
- **Embeddings**: `embedder/` - Embedding clients for various providers
- **Graph Elements**: `nodes.py`, `edges.py` - Core graph data structures
- **Search**: `search/` - Hybrid search implementation with configurable strategies
- **Prompts**: `prompts/` - LLM prompts for entity extraction, deduplication, summarization
- **Utilities**: `utils/` - Maintenance operations, bulk processing, datetime handling

### Server (`server/`)

- **FastAPI Service**: `graph_service/main.py` - REST API server
- **Routers**: `routers/` - API endpoints for ingestion and retrieval
- **DTOs**: `dto/` - Data transfer objects for API contracts

### MCP Server (`mcp_server/`)

- **MCP Implementation**: `graphiti_mcp_server.py` - Model Context Protocol server for AI assistants
- **Docker Support**: Containerized deployment with Neo4j

## Testing

- **Unit Tests**: `tests/` - Comprehensive test suite using pytest
- **Integration Tests**: Tests marked with `_int` suffix require database connections
- **Evaluation**: `tests/evals/` - End-to-end evaluation scripts

## Configuration

### Environment Variables

- `OPENAI_API_KEY` - Required for LLM inference and embeddings
- `USE_PARALLEL_RUNTIME` - Optional boolean for Neo4j parallel runtime (enterprise only)
- `SEMAPHORE_LIMIT` - Controls concurrency (default: 10). Increase for higher throughput if your LLM provider allows it
- `GRAPHITI_TELEMETRY_ENABLED` - Set to 'false' to disable anonymous usage telemetry
- Provider-specific keys: `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`, `GROQ_API_KEY`, `VOYAGE_API_KEY`

### Performance and Concurrency

- **Concurrency**: Default concurrency is set low (SEMAPHORE_LIMIT=10) to prevent 429 rate limit errors
- **Rate Limiting**: If you encounter 429 errors, lower SEMAPHORE_LIMIT. For higher throughput providers, increase it
- **Telemetry**: Anonymous usage statistics are collected by default. Disable with `GRAPHITI_TELEMETRY_ENABLED=false`

### Database Setup

- **Neo4j**: Version 5.26+ required, available via Neo4j Desktop
  - Database name defaults to `neo4j` (hardcoded in Neo4jDriver)
  - Override by passing `database` parameter to driver constructor
- **FalkorDB**: Version 1.1.2+ as alternative backend
  - Database name defaults to `default_db` (hardcoded in FalkorDriver)
  - Override by passing `database` parameter to driver constructor

## Development Guidelines

### Code Style

- Use Ruff for formatting and linting (configured in pyproject.toml)
- Line length: 100 characters
- Quote style: single quotes
- Type checking with Pyright is enforced
- Main project uses `typeCheckingMode = "basic"`, server uses `typeCheckingMode = "standard"`

### Testing Requirements

- Run tests with `make test` or `pytest`
- Integration tests require database connections and are marked with `_int` suffix
- Use `pytest-xdist` for parallel test execution
- Run specific test files: `pytest tests/test_specific_file.py`
- Run specific test methods: `pytest tests/test_file.py::test_method_name`
- Run only integration tests: `pytest tests/ -k "_int"`
- Run only unit tests: `pytest tests/ -k "not _int"`

### LLM Provider Support

The codebase supports multiple LLM providers but works best with services supporting structured output (OpenAI, Gemini). Other providers may cause schema validation issues, especially with smaller models.

**Installation with LLM providers:**
```bash
# Install with specific providers
pip install graphiti-core[anthropic,groq,google-genai,falkordb]
```

**Azure OpenAI Configuration:**
For Azure deployments, use separate AsyncAzureOpenAI clients for LLM and embedding endpoints. Pass these to OpenAIClient, OpenAIEmbedder, and OpenAIRerankerClient with your Azure deployment names in LLMConfig.

### MCP Server Usage Guidelines

When working with the MCP server, follow the patterns established in `mcp_server/cursor_rules.md`:

- Always search for existing knowledge before adding new information
- Use specific entity type filters (`Preference`, `Procedure`, `Requirement`)
- Store new information immediately using `add_memory`
- Follow discovered procedures and respect established preferences

## Fork-Specific Changes (Multi-Tenant Support)

This fork includes significant improvements to the MCP server:

### Docker Permission Fix
- **Issue**: Original Dockerfile installed `uv` in root user directory, inaccessible after switching to `USER app`
- **Solution**: Install `uv` to system path `/usr/local/bin/` for universal access
- **Status**: Tested with Podman build - working correctly

### Dynamic Project Support via X-Project Header
- **Implementation**: Added `ProjectHeaderMiddleware` to extract `X-Project` HTTP header
- **Validation**: Basic format validation (alphanumeric, hyphens, underscores only)  
- **Case Handling**: Supports both `x-project` and `X-Project` headers
- **Error Handling**: Graceful fallback if header extraction fails

### Multi-Tenant Data Isolation
- **Core Function**: `get_dynamic_group_id()` with priority system:
  1. Explicit function parameter `group_id`
  2. `X-Project` header value
  3. Config default value
- **Modified Tools**: All MCP tools now support dynamic group_id:
  - `add_memory`
  - `search_memory_nodes` 
  - `search_memory_facts`
  - `get_episodes`

### Usage Patterns
```bash
# Via HTTP header (recommended for multi-tenant scenarios)
curl -H "X-Project: project-name" ...

# Via function parameter (backward compatible)
add_memory(..., group_id="specific-project")

# Default fallback (no header, no parameter)
# Uses config.group_id or 'default'
```

### Data Isolation Guarantees
- Graphiti uses strict Cypher WHERE clauses: `WHERE n.group_id IN $group_ids`
- Different projects are completely isolated - no cross-project data access
- Applies to all levels: nodes, edges, communities, fulltext search
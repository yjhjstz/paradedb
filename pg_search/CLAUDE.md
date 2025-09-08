# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

pg_search is a PostgreSQL extension that enables full-text search using the BM25 algorithm. It's built on top of Tantivy (Rust-based Lucene alternative) using pgrx (Postgres extension framework for Rust). The extension integrates deeply with PostgreSQL's query planner and executor to provide efficient search capabilities.

## Build Commands

### Development Setup
```bash
# Initialize pgrx (replace --pg14 with your PostgreSQL version)
cargo pgrx init --pg14=/usr/lib/postgresql/14/bin/pg_config

# Run development server with extension loaded
cargo pgrx run

# Build with specific PostgreSQL version (avoid version conflicts)
cargo build --features "pg14,cbdb,pg_test"
```

### Testing
```bash
# Run all tests (requires DATABASE_URL environment variable)
cargo test

# Run pgrx tests
cargo pgrx test

# Run specific test
cargo test --test <test_name>
```

The tests require a `.env` file with:
```
DATABASE_URL=postgres://USER_NAME@localhost:PORT/pg_search
```
Where PORT = 28800 + postgres_version (e.g., 28817 for Postgres 14).

### ICU Tokenizer Development
To enable ICU tokenizer support for additional languages, add `--features icu` to build/run commands:
```bash
cargo pgrx run --features icu
cargo pgrx test --features icu
```

## Architecture Overview

### Core Components

**PostgreSQL Integration Layer** (`src/postgres/`):
- `rel.rs`: Reference-counted wrapper around PostgreSQL relations with proper lifecycle management
- `customscan/`: Custom scan nodes for integrating with PostgreSQL's query executor
- `build.rs`, `build_parallel.rs`: Index building logic with parallel worker support
- `storage/`: Block-based storage system for persisting Tantivy indices in PostgreSQL pages

**Tantivy Index Management** (`src/index/`):
- `directory/mvcc.rs`: MVCC-aware directory implementation that stores Tantivy data in PostgreSQL blocks
- `writer/`: Index writers with segment management
- `reader/`: Index readers for query execution
- `search.rs`: Search execution engine

**Query Processing** (`src/query/`):
- BM25 scoring integration with PostgreSQL's cost-based optimizer
- Proximity search support
- Range queries and filtering

**API Layer** (`src/api/`):
- SQL functions and operators exposed to users
- Query builders and search configuration

### Key Architectural Patterns

1. **MVCC Integration**: The extension implements its own MVCC directory (`MVCCDirectory`) that stores Tantivy indices directly in PostgreSQL's block storage, ensuring ACID compliance.

2. **Custom Scan Nodes**: Uses PostgreSQL's custom scan API to integrate BM25 search into query plans, allowing the optimizer to choose between index scans and sequential scans.

3. **Parallel Processing**: Supports parallel index building and query execution using PostgreSQL's parallel query infrastructure.

4. **Reference Counting**: Critical resources like `PgSearchRelation` use reference counting with proper cleanup in PostgreSQL's transaction context.

## Important Development Notes

### PgSearchRelation Safety
The `PgSearchRelation` struct is reference-counted and can be in a "closed" state where its internal Option is None. Always check for null states when accessing the underlying relation pointer to avoid segmentation faults.

### PostgreSQL Version Features
The extension supports multiple PostgreSQL versions (14-17) through feature flags. Use `--no-default-features --features pg14` (or appropriate version) to avoid conflicts.

### Memory Context Management
The extension heavily uses PostgreSQL's memory context system. Be aware of context switches and ensure proper cleanup in error cases.

### Testing Environment
Tests run against a dedicated PostgreSQL instance created by pgrx. The extension must be loaded in `shared_preload_libraries` for versions < 17 due to background worker requirements.

## Common File Patterns

- `src/postgres/customscan/pdbscan/`: Main scan execution logic
- `src/index/directory/mvcc.rs`: Core storage integration 
- `src/postgres/storage/`: Low-level block management
- `tests/pg_regress/`: PostgreSQL regression tests
- Test files use setup/cleanup SQL pairs in `tests/pg_regress/common/`

## Critical Debugging Areas

When encountering crashes, check:
1. `PgSearchRelation` null pointer access (common in `src/postgres/rel.rs`)
2. Memory context violations during index operations
3. MVCC directory state consistency
4. Parallel worker communication issues

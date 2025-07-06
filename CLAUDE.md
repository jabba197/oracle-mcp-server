# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an MCP (Model Context Protocol) server that provides Oracle database schema context to AI assistants. It implements a three-layer architecture: DatabaseConnector → SchemaManager → DatabaseContext.

## Common Development Commands

```bash
# Install dependencies with UV
uv sync --frozen

# Run the server locally
uv run main.py

# Code formatting
black .

# Linting
ruff check .

# Type checking
mypy .

# Run test database
cd test/db && docker-compose up

# Build Docker image
docker build -t oracle-mcp-server .
```

## Architecture

The codebase follows a layered architecture:

1. **main.py**: Entry point defining 13 MCP tools for database introspection
2. **db_context/database.py**: Oracle connection management with pooling and retry logic
3. **db_context/schema/manager.py**: Intelligent schema caching system that handles large databases (10,000+ tables)
4. **db_context/schema/formatter.py**: Schema formatting for AI consumption
5. **db_context/models.py**: Data models (TableInfo, ColumnInfo, etc.)

## Key Implementation Details

- **Schema Caching**: The SchemaManager implements a sophisticated caching strategy that persists schema metadata to disk, crucial for handling large Oracle databases efficiently
- **Connection Modes**: Supports both thin (pure Python) and thick (Oracle Client) modes
- **Error Handling**: All database operations include proper error handling and retry logic
- **Type Safety**: Strict typing throughout with mypy in strict mode

## Environment Configuration

Required environment variables:
- `ORACLE_CONNECTION_STRING`: Database connection string (format: `user/password@host:port/service`)
- `TARGET_SCHEMA`: Optional schema override
- `CACHE_DIR`: Cache directory (default: `.cache`)
- `THICK_MODE`: Enable thick mode (default: false)

## Testing

Use the Docker-based test database in `test/db/` for development. It includes sample schemas and data for testing all MCP tools.

## Important Notes

- The project uses UV as the package manager (not pip)
- Python 3.12+ is required
- All code should maintain type hints and pass mypy strict checks
- The schema cache is critical for performance - never bypass it for large databases
- When modifying database queries, test with both small and large schemas
- **View Support**: The system now supports both tables and views with graceful permission handling
- **Permission Handling**: Uses fallback queries from ALL_* to USER_* views for limited permissions
- **Client-side Similarity**: Uses Python's difflib instead of Oracle's UTL_MATCH for permission compatibility
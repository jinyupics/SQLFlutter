# CLAUDE.md - AI Assistant Guidelines for SQLFlutter

## Project Overview

SQLFlutter is an **architectural specification** proposing a radical simplification of Flutter application development. The core thesis:

```
UI = sql.f(database)
Database = sql.f(Database, UserAction)
```

**This is a documentation-driven project**, not an implementation codebase. The repository contains architectural specifications and design documents describing a novel approach to Flutter state management.

## Repository Structure

```
SQLFlutter/
├── README.md          # English specification document
├── README_CN.md       # Chinese specification document (中文版本)
├── LICENSE            # Apache License 2.0
└── CLAUDE.md          # This file
```

## Core Architectural Concepts

### 1. Routing as Data
- Navigation stored in a SQL table, not a state object
- Single source of truth: `SELECT current_screen FROM routing`
- Route changes via `UPDATE routing SET current_screen = '/path'`

### 2. Widget Content = SQL Query
- Every widget binds to a SQL query result
- Data changes trigger automatic UI updates
- No `setState()`, no `notifyListeners()`, no manual rebuilds

### 3. User Actions = Database Operations
- All user interactions become INSERT/UPDATE/UPSERT operations
- No new state creation, only database mutations
- Actions and UI are completely decoupled through the database

### 4. Minimal Architecture
```
Traditional: Action → Controller → Service → Repository → Model → Database → ... → UI
SQLFlutter:  Action → Database → UI
```

No services, models, repositories, DTOs, or mappers - just tables and queries.

## Proposed API Design

| Method | Purpose | Reactive |
|--------|---------|----------|
| `sql.watch()` | Query data with auto-refresh | Yes |
| `sql.run()` | Execute INSERT/UPDATE/DELETE | No |

## Development Workflow

### When Contributing to Specifications

1. **Maintain bilingual parity** - Changes to README.md should be reflected in README_CN.md and vice versa
2. **Preserve architectural consistency** - All examples should follow the `Action → Database → UI` pattern
3. **Use concrete examples** - Include SQL schema and Dart code snippets
4. **Include visual diagrams** - ASCII art diagrams help explain concepts

### Writing Style Guidelines

- Keep explanations concise and practical
- Compare with traditional approaches to highlight benefits
- Use real-world scenarios (task lists, user profiles, notifications)
- Show complete working examples, not just fragments

## Key Files Reference

| File | Purpose | Language |
|------|---------|----------|
| `README.md` | Main specification | English |
| `README_CN.md` | Main specification | Chinese |

## Important Conventions

### SQL Examples
- Use SQLite-compatible syntax
- Show complete table definitions with `CREATE TABLE`
- Include common operations: SELECT, INSERT, UPDATE, UPSERT

### Dart Examples
- Target Flutter/Dart syntax
- Show reactive patterns with `sql.watch()`
- Show mutations with `sql.run()`
- Keep widget code minimal and readable

### Diagram Style
- Use ASCII box-drawing characters
- Show data flow direction with arrows (→, ↓, ←, ↑)
- Label each section clearly

## Current Project Status

- **Phase**: Specification/Design (no implementation yet)
- **Focus**: Documenting the architectural paradigm
- **Future**: May evolve into a Dart/Flutter library

## What This Project Is NOT

- Not an implementation of the Drift database library
- Not a state management package (yet)
- Not production-ready code
- Not a replacement for existing solutions (until implemented)

## Common Tasks for AI Assistants

### Adding New Concepts
1. Add to README.md with English explanation
2. Add corresponding section to README_CN.md
3. Include SQL and Dart examples
4. Add visual diagram if helpful

### Fixing Documentation
1. Ensure fixes apply to both language versions
2. Maintain consistent terminology across documents
3. Keep examples syntactically correct

### Expanding Examples
1. Follow existing patterns for consistency
2. Use the same sample domain (tasks, users, routing)
3. Show the simplicity advantage vs. traditional approaches

## License

Apache License 2.0 - permissive open-source license allowing commercial use, modification, and distribution.

## Related Technologies

Understanding these helps when working on this project:
- **Flutter**: Google's UI toolkit for cross-platform apps
- **Drift**: Reactive persistence library for Flutter (inspiration for SQL patterns)
- **SQLite**: The underlying database engine
- **Reactive programming**: The auto-refresh paradigm this project promotes

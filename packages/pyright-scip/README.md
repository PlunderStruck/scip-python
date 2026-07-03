# scip-python-plus

Enhanced fork of [scip-python](https://github.com/sourcegraph/scip-python) with full support for `self.method()` calls, instance attributes, and decorated functions.

Built on [Pyright](https://github.com/microsoft/pyright) for generating [SCIP](https://github.com/sourcegraph/scip) indexes from Python projects.

## What's different from upstream scip-python?

The original scip-python silently drops all `MemberAccess` nodes, which means:

- `self.method()` calls produce no cross-references
- `self.attribute` access is invisible to the index
- Decorated functions (e.g., `@app.get`, `@pytest.fixture`) show as dead code

**scip-python-plus fixes all of this:**

| Feature | scip-python | scip-python-plus |
|---|---|---|
| `self.method()` calls | Not tracked | Fully resolved via Pyright type evaluator |
| `self.attribute` access | Not tracked | Tracked as references |
| Instance attribute definitions (`self.x = ...` in `__init__`) | Not emitted | Emitted as definition symbols |
| Decorated functions (`@app.get`, `@route`) | Reported as dead code | Implicit reference prevents false positives |
| Static/class method access (`MyClass.method()`) | Not tracked | Fully resolved |
| Module member access (`os.path`) | Not tracked | Resolved via module type |

## Install

```bash
npm install -g scip-python-plus
```

Requires Node v16+ and Python 3.10+.

Used automatically by [scip-query](https://github.com/PlunderStruck/scip-query) when indexing Python projects.

This package now installs both `scip-python-plus` and `scip-python` on your `PATH`. `scip-python-plus` is the preferred command name, and `scip-python` remains as a compatibility alias.

## Usage

```bash
# Activate your virtual environment first
scip-python-plus index . --project-name=my-project --project-version=0.1.0

# Convert to SQLite for scip-query
scip expt-convert --output index.db index.scip

# Query the index
scip-query dead
scip-query call-graph my_function
scip-query refs my_function
```

If you hit an out-of-memory error, increase the heap size:
```bash
NODE_OPTIONS="--max-old-space-size=8192" scip-python-plus index . --project-name=my-project
```

### Options

```bash
# Index only a specific subdirectory
scip-python-plus index . --project-name=my-project --target-only=src/subdir

# Add a namespace prefix to all symbols
scip-python-plus index . --project-name=my-project --project-namespace=implicit.namespace

# Supply a custom environment file (skips pip detection)
scip-python-plus index . --project-name=my-project --environment=path/to/env.json
```

## Environment file format

If you don't use pip, supply a JSON file listing your packages:

```json
[
  {
    "name": "PyYAML",
    "version": "6.0",
    "files": [
      "yaml/__init__.py",
      "yaml/composer.py",
      "yaml/tokens.py"
    ]
  }
]
```

## Technical details

All changes are in `packages/pyright-scip/src/treeVisitor.ts`. The key fix: the original code returned `ScipSymbol.empty()` for all `MemberAccess` parse nodes. This fork resolves them using Pyright's `TypeEvaluator.getTypeOfExpression()` and `lookUpClassMember()`, which already have full type information — it just wasn't being used for SCIP emission.

## Upstream

Based on [sourcegraph/scip-python](https://github.com/sourcegraph/scip-python). Upstream changes to Pyright internals are minimal (3 patches for indexing stability).

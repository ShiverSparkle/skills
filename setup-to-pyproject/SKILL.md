---
name: setup-to-pyproject
description: Migrate Python projects from setup.py/setup.cfg to pyproject.toml for use with uv. Use when upgrading legacy Python packaging, converting setup.py to modern pyproject.toml format, setting up dependency groups for development/testing, or ensuring `uv run pytest` works correctly. Handles dependency-groups with dev dependencies.
---

# Setup.py to pyproject.toml Migration for uv

Migrate legacy Python packaging to modern pyproject.toml with uv compatibility.

## Core Structure

```toml
[project]
name = "package-name"
version = "0.1.0"
description = "Package description"
readme = "README.md"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.28",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=4.0",
    "ruff>=0.4",
    "mypy>=1.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

## Key Migration Steps

### 1. Extract metadata from setup.py

Map setup() arguments to [project] table:
- `name` → `name`
- `version` → `version`
- `description` → `description`
- `long_description` → `readme` (use filename)
- `author` + `author_email` → `authors = [{name = "...", email = "..."}]`
- `url` → `urls.Homepage`
- `python_requires` → `requires-python`
- `install_requires` → `dependencies`
- `entry_points.console_scripts` → `[project.scripts]`

### 2. Set up dependency-groups for dev dependencies

**Critical for `uv run pytest` to work:**

```toml
[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=4.0",
]
```

Then run tests with:
```bash
uv run --group dev pytest
```

Or install dev dependencies:
```bash
uv sync --group dev
uv run pytest
```

### 3. Map extras_require to dependency-groups or optional-dependencies

For dev/test extras → use `[dependency-groups]`:
```toml
[dependency-groups]
dev = ["pytest>=8.0", "ruff>=0.4"]
test = ["pytest>=8.0", "coverage>=7.0"]
docs = ["sphinx>=7.0", "sphinx-rtd-theme>=2.0"]
```

For user-facing optional features → use `[project.optional-dependencies]`:
```toml
[project.optional-dependencies]
postgres = ["psycopg2>=2.9"]
redis = ["redis>=5.0"]
```

### 4. Choose build backend

**hatchling** (recommended for most projects):
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

**setuptools** (if complex build needs):
```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"]
```

### 5. Configure tool settings

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"

[tool.ruff]
line-length = 88
target-version = "py310"

[tool.mypy]
python_version = "3.10"
strict = true
```

## Complete Example

**Before (setup.py):**
```python
from setuptools import setup, find_packages

setup(
    name="mypackage",
    version="1.0.0",
    description="My package",
    author="Dev",
    author_email="dev@example.com",
    python_requires=">=3.10",
    packages=find_packages(where="src"),
    package_dir={"": "src"},
    install_requires=[
        "requests>=2.28",
        "click>=8.0",
    ],
    extras_require={
        "dev": ["pytest>=8.0", "ruff>=0.4"],
    },
    entry_points={
        "console_scripts": [
            "mycli=mypackage.cli:main",
        ],
    },
)
```

**After (pyproject.toml):**
```toml
[project]
name = "mypackage"
version = "1.0.0"
description = "My package"
readme = "README.md"
requires-python = ">=3.10"
authors = [{name = "Dev", email = "dev@example.com"}]
dependencies = [
    "requests>=2.28",
    "click>=8.0",
]

[project.scripts]
mycli = "mypackage.cli:main"

[dependency-groups]
dev = [
    "pytest>=8.0",
    "ruff>=0.4",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/mypackage"]
```

## uv Workflow

```bash
# Initialize project (creates pyproject.toml if missing)
uv init

# Add runtime dependency
uv add requests

# Add dev dependency to group
uv add --group dev pytest ruff mypy

# Sync all dependencies including dev
uv sync --group dev

# Run tests
uv run --group dev pytest

# Or after sync:
uv run pytest

# Build package
uv build

# Publish to PyPI
uv publish
```

## Files to Delete After Migration

- `setup.py`
- `setup.cfg`
- `MANIFEST.in` (usually, unless complex includes)
- `requirements.txt` / `requirements-dev.txt` (optional, uv manages these)

## Troubleshooting

**`uv run pytest` fails with "pytest not found":**
```bash
uv sync --group dev  # Install dev dependencies first
uv run pytest
```

**Package not found during import:**
Ensure build backend finds your packages:
```toml
# For src layout with hatchling:
[tool.hatch.build.targets.wheel]
packages = ["src/mypackage"]

# For src layout with setuptools:
[tool.setuptools.packages.find]
where = ["src"]
```

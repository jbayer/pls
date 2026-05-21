# pls - Two-Environment Flox Workflow

This repo demonstrates the recommended two-environment pattern for building and distributing packages with Flox. A **published package** and a **pushed environment** are two distinct types of FloxHub artifacts, so they use separate environments.

## Overview

| | Env 1 - Producer | Env 2 - Consumer |
|---|---|---|
| **Location** | Project root (`.flox/`) | `consumer/` subdirectory |
| **Purpose** | Build toolchain + build definition | Installs the published package |
| **Contains** | Go compiler, source code, `[build]` section | Only the published `jbayer/hwinf.pls` package |
| **FloxHub artifact** | `flox publish` produces a **package** | `flox push` produces a **remote environment** |

## Env 1 - The Producer

The producer environment lives alongside the source code in the git repo. The `[install]` section carries the build toolchain and the `[build]` section compiles the binary into `$out/bin`.

### Manifest (`./flox/env/manifest.toml`)

```toml
[install]
go.pkg-path = "go"

[build."hwinf.pls"]
description = "Hello world pls CLI"
version = "0.1.0"
command = '''
  go build -o pls .
  mkdir -p $out/bin
  cp pls $out/bin/
'''
```

**Important:** The dotted package name `hwinf.pls` must be quoted in TOML (`[build."hwinf.pls"]`). Without quotes, TOML interprets the dot as table nesting — `[build.hwinf.pls]` would parse as a `pls` key inside a `hwinf` table inside `build`, which is not valid and will produce an error like:

```
unknown field `pls`, expected one of `command`, `runtime-packages`, `sandbox`, `version`, `description`, `license`
in `build.hwinf`
```

### Build and Publish Commands

```bash
# Build the package locally
flox build hwinf.pls

# Test the built binary
./result-hwinf.pls/bin/pls

# Publish the package to FloxHub (requires clean git state + pushed remote)
flox publish -o jbayer hwinf.pls
```

After publishing, the package is available as `jbayer/hwinf.pls` in the Flox catalog.

## Env 2 - The Consumer

The consumer environment is a thin, source-free environment that installs the published package and gets pushed to FloxHub. It does not need the source repo or build tools.

### Manifest (`consumer/.flox/env/manifest.toml`)

```toml
[install]
pls.pkg-path = "jbayer/hwinf.pls"
```

### Setup and Push Commands

```bash
# Create the consumer environment
flox init -d consumer -n hwinf.pls

# Install the published package
flox install -d consumer jbayer/hwinf.pls

# Test locally
flox activate -d consumer -- pls

# Push the environment to FloxHub
flox push -d consumer -o jbayer
```

## Consuming the Package

Once both artifacts are on FloxHub, there are two ways to use the package:

### Option A: Install the package into any environment

```bash
flox install jbayer/hwinf.pls
```

### Option B: Activate the remote consumer environment

```bash
flox activate -r jbayer/hwinf.pls -- pls
```

## Prerequisites for Publishing

The `flox publish` command has hard prerequisites:

- Environment must be in a **git repository**
- All tracked files must be **clean** (no uncommitted changes)
- A **remote** must be configured
- The current revision must be **pushed** to the remote
- The environment must contain at least one package in `[install]`

Flox clones the repo to a temp location and performs a clean build to ensure reproducibility.

## Naming Convention

Following the recommended convention for organizations with multiple teams:

- **Package names** use dots for hierarchy: `hwinf.pls` (team.package)
- **Full install path**: `jbayer/hwinf.pls` (org/team.package)
- This maps to the pattern an organization like NVIDIA would use: `nvidia/hwinf.pls`

### Quoting Dotted Names in TOML

Because TOML uses dots as table separators, any dotted name in a table header must be quoted. This applies to both the `[build]` section and the `[install]` section:

```toml
# Correct - quotes preserve the dot as part of the name
[build."hwinf.pls"]

# Wrong - TOML parses this as nested tables and will error
[build.hwinf.pls]
```

On the command line, quotes are not needed — `flox build hwinf.pls` and `flox install jbayer/hwinf.pls` work as-is.

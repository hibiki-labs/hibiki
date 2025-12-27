# Hibiki Repository Structure

This document describes the Git repository hierarchy for the Hibiki project.

## Overview

The Hibiki project follows a **monorepo with submodules** architecture. The main `hibiki` repository contains four component repositories as Git submodules, while each component also exists as a standalone repository for independent development.

## Directory Layout

```
/Hibiki/
│
├── hibiki/                    # Main repository (parent)
│   ├── hibiki-console/        # Submodule
│   ├── hibiki-engine/         # Submodule
│   ├── hibiki-infra/          # Submodule
│   └── hibiki-ingest/         # Submodule
│
├── hibiki-console/            # Standalone repository
├── hibiki-engine/             # Standalone repository
├── hibiki-infra/              # Standalone repository
└── hibiki-ingest/             # Standalone repository
```

## Repositories

| Repository | GitHub Remote | Description |
|------------|---------------|-------------|
| `hibiki` | https://github.com/abgnpr/hibiki.git | Main/parent repository |
| `hibiki-console` | https://github.com/abgnpr/hibiki-console.git | Console component |
| `hibiki-engine` | https://github.com/abgnpr/hibiki-engine.git | Engine component |
| `hibiki-infra` | https://github.com/abgnpr/hibiki-infra.git | Infrastructure component |
| `hibiki-ingest` | https://github.com/abgnpr/hibiki-ingest.git | Ingest component |

## Architecture Details

### Main Repository (`hibiki`)

The `hibiki` repository serves as the parent project and contains references to all component repositories as Git submodules. This allows:

- Unified versioning of all components
- Integration testing across components
- Single clone for the complete project

### Component Repositories

Each component exists in two forms:

1. **As a submodule** inside `hibiki/` - linked to a specific commit in the parent repo
2. **As a standalone repo** at the root level - for independent development

### Branch Strategy

All repositories use the `dev` branch as the primary development branch.

## Working with This Structure

### Cloning the Full Project

```bash
git clone --recurse-submodules https://github.com/abgnpr/hibiki.git
```

### Updating Submodules

```bash
cd hibiki
git submodule update --remote
```

### Working on a Component Independently

```bash
cd hibiki-console  # Use the standalone repo
git pull origin dev
# Make changes...
git push origin dev
```

### Syncing Submodule Changes to Parent

After pushing changes to a component repository:

```bash
cd hibiki
git submodule update --remote hibiki-console
git add hibiki-console
git commit -m "Update hibiki-console submodule"
git push origin dev
```

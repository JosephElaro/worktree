# WORKTREE — The Next-Generation Version Control System

> **Automatic Control Architecture for Source Code Management**
> Replace Git. Rethink versioning. Stay compatible.

---

## Table of Contents

1. [Vision](#1-vision)
2. [Why Not Git?](#2-why-not-git)
3. [Core Concepts](#3-core-concepts)
4. [Architecture Overview](#4-architecture-overview)
5. [The Protocol](#5-the-protocol)
6. [The Server](#6-the-server)
7. [The Client & SDK](#7-the-client--sdk)
8. [Nested Tree Model](#8-nested-tree-model)
9. [Automatic Control Architecture](#9-automatic-control-architecture)
10. [Access & Control Architecture](#10-access--control-architecture)
11. [Native Git Compatibility](#11-native-git-compatibility)
12. [Platform Support](#12-platform-support)
13. [Project Structure](#13-project-structure)
14. [Milestones & Roadmap](#14-milestones--roadmap)
15. [Technical Decisions](#15-technical-decisions)
16. [Open Questions](#16-open-questions)

---

## 1. Vision

Worktree is a **modern version control system** built from the ground up in Rust to replace Git. It is designed around two pillars:

- **A Protocol** — defines the data structures, tree model, object format, diffing semantics, and wire format for communication.
- **A Server** — a persistent background process that watches your filesystem, detects changes in real-time, and manages branches, commits, and synchronization automatically.

Git was revolutionary in 2005. It is now 2024. The workflows we force onto Git — monorepos, fine-grained permissions, multi-team collaboration, CI-driven branching — were never part of its original design. Worktree does not patch over Git. It replaces the model entirely.

Critically, Worktree ships with **native Git compatibility**. It can read Git repositories, convert them to Worktree format, and export Worktree trees back to Git. This means teams can adopt Worktree incrementally — import existing Git history, work in Worktree's superior model, and push back to Git remotes when collaborating with the rest of the world. No big-bang migration required.

---

## 2. Why Not Git?

| Problem with Git | Worktree's Answer |
|---|---|
| Flat repository model — one `.git` root, everything is folders | **Nested Trees** — trees inside trees, each with their own history, branches, and permissions |
| Manual everything — you stage, commit, branch, merge by hand | **Automatic Control Architecture** — the server detects changes, structures branches, and creates commits intelligently |
| No native permissions model — relies on hosting platforms (GitHub, GitLab) | **Built-in Access & Control Architecture** — tenant-level and user-level permissions are first-class protocol concepts |
| Monorepo tooling is bolted on (git-subtree, git-submodule, sparse-checkout) | **Native nested tree model** — a monorepo is just a root tree containing child trees. No hacks. |
| Performance degrades at scale — large repos, large files, deep histories | **Rust-native performance** — zero-cost abstractions, memory safety, and a protocol designed for efficiency from day one |
| No daemon / background intelligence | **Persistent server process** — runs as a background service, watches the filesystem, handles events in real-time |
| Poor Windows support historically | **First-class Windows, Linux, and macOS support** — the server installs and runs as a native background process on all three |
| Switching requires abandoning all existing repos and workflows | **Native Git compatibility** — import Git repos with full history, export back to Git, push/pull to Git remotes. Adopt incrementally. |

---

## 3. Core Concepts

### 3.1 Trees

A **Tree** is the fundamental unit of Worktree. It represents a versioned directory structure with its own:

- History (commits / snapshots)
- Branches
- Permissions
- Configuration

Unlike Git where you have ONE repository root, Worktree allows **nested trees**. A root tree can contain child trees, and those children can contain their own children. Each tree is independently versioned but aware of its parent.

### 3.2 Snapshots (Commits)

A **Snapshot** is a point-in-time capture of a tree's state. Unlike Git commits which are manually created, Worktree snapshots can be:

- **Auto-generated** by the server when meaningful change boundaries are detected
- **Manually created** by the user when explicit checkpoints are desired
- **Structured** by the Automatic Control Architecture based on rules and patterns

### 3.3 Branches

A **Branch** is a named divergence within a tree. Worktree branches are:

- Automatically suggested and created based on detected change patterns
- Scoped to a specific tree (not the entire root)
- Subject to the Access & Control Architecture

### 3.4 Tenants & Users

A **Tenant** is an organizational unit (team, company, project group). A **User** is an individual identity. Permissions are resolved through a hierarchy:

```
Tenant → Team → User → Tree → Branch
```

### 3.5 Events

An **Event** is any state change detected by the server — file creation, modification, deletion, rename, permission change, etc. Events drive the Automatic Control Architecture.

---

## 4. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                            WORKTREE                                  │
│                                                                      │
│  ┌─────────────────────┐       ┌──────────────────────────┐          │
│  │      PROTOCOL       │       │         SERVER           │          │
│  │                     │       │                          │          │
│  │  • Object Model     │◄─────►│  • Background Process    │          │
│  │  • Tree Structure   │       │  • Filesystem Watcher    │          │
│  │  • Wire Format      │       │  • Event Engine          │          │
│  │  • Diff Semantics   │       │  • Auto-Commit Engine    │          │
│  │  • Merge Semantics  │       │  • Auto-Branch Engine    │          │
│  │  • Permission Model │       │  • Sync Engine           │          │
│  │  • Snapshot Format  │       │  • Permission Enforcer   │          │
│  │                     │       │  • Storage Backend       │          │
│  └─────────┬───────────┘       └────────────┬─────────────┘          │
│            │                                │                        │
│            │         ┌──────────┐           │                        │
│            └────────►│   API    │◄──────────┘                        │
│                      │  Layer   │                                    │
│                      └─────┬────┘                                    │
│                            │                                         │
│               ┌────────────┼────────────┐                            │
│               │            │            │                            │
│          ┌────▼───┐  ┌─────▼────┐  ┌────▼───┐                       │
│          │  CLI   │  │  Client  │  │  SDK   │                       │
│          │ Client │  │   GUI    │  │ (lib)  │                       │
│          └────────┘  └──────────┘  └────────┘                       │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                    GIT COMPATIBILITY LAYER                     │   │
│  │                                                               │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐ │   │
│  │  │  Git Reader   │  │  Git Writer  │  │  Git Remote Bridge  │ │   │
│  │  │              │  │              │  │                     │ │   │
│  │  │ .git/ → WT   │  │ WT → .git/  │  │ push/pull to Git   │ │   │
│  │  │ objects      │  │ objects      │  │ remotes (GitHub,    │ │   │
│  │  │ history      │  │ history      │  │ GitLab, Bitbucket)  │ │   │
│  │  └──────────────┘  └──────────────┘  └─────────────────────┘ │   │
│  └───────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. The Protocol

The Protocol is the **specification layer**. It defines *what* Worktree is without prescribing *how* the server implements it. Any conforming implementation must satisfy the protocol.

### 5.1 Responsibilities

- **Object Model** — defines the types: `Tree`, `Blob`, `Snapshot`, `Branch`, `Delta`, `Manifest`, `Permission`, `Tenant`, `User`
- **Tree Structure** — defines how trees nest, reference each other, and maintain independent histories
- **Wire Format** — binary serialization format for all protocol objects (efficient, compact, versionable)
- **Diff Semantics** — how changes between two snapshots are computed and represented
- **Merge Semantics** — how two diverged branches are reconciled (conflict model, resolution strategies)
- **Permission Model** — the schema for access control: who can read, write, branch, merge, admin at each tree level
- **Snapshot Format** — the content-addressable structure of a snapshot (Merkle tree based, like Git, but supporting nested tree references)
- **Git Object Mapping** — defines the bidirectional mapping between Git objects (commit, tree, blob, tag) and Worktree objects (Snapshot, Tree, Blob, Branch)

### 5.2 Protocol Crate

```
worktree-protocol/
├── src/
│   ├── lib.rs
│   ├── object/
│   │   ├── mod.rs
│   │   ├── tree.rs          // Tree definition and nesting rules
│   │   ├── blob.rs          // File content objects
│   │   ├── snapshot.rs      // Point-in-time tree captures
│   │   ├── branch.rs        // Branch references and metadata
│   │   ├── delta.rs         // Change representation
│   │   └── manifest.rs      // Tree manifest (what's in the tree)
│   ├── identity/
│   │   ├── mod.rs
│   │   ├── tenant.rs        // Tenant (org-level) identity
│   │   └── user.rs          // User identity
│   ├── permission/
│   │   ├── mod.rs
│   │   ├── model.rs         // Permission schema
│   │   ├── scope.rs         // Permission scopes (tree, branch, global)
│   │   └── policy.rs        // Policy evaluation logic
│   ├── wire/
│   │   ├── mod.rs
│   │   ├── encode.rs        // Serialization
│   │   ├── decode.rs        // Deserialization
│   │   └── format.rs        // Format version and negotiation
│   ├── diff/
│   │   ├── mod.rs
│   │   ├── compute.rs       // Diff algorithm
│   │   └── patch.rs         // Patch application
│   ├── merge/
│   │   ├── mod.rs
│   │   ├── strategy.rs      // Merge strategies
│   │   └── conflict.rs      // Conflict representation and resolution
│   ├── compat/
│   │   ├── mod.rs
│   │   ├── git_object_map.rs   // Mapping rules: Git objects ↔ Worktree objects
│   │   ├── git_ref_map.rs      // Mapping rules: Git refs/branches/tags ↔ Worktree branches
│   │   └── git_hash_map.rs     // SHA-1 (Git) ↔ BLAKE3 (Worktree) hash index
│   └── hash.rs              // Content-addressable hashing (BLAKE3)
├── Cargo.toml
└── README.md
```

---

## 6. The Server

The Server is the **runtime layer**. It is a persistent background process that manages trees, reacts to filesystem events, and serves the API.

### 6.1 Responsibilities

- **Background Process Management** — install, start, stop, restart as a system service on Windows, Linux, and macOS
- **Filesystem Watching** — monitor registered tree roots for changes in real-time
- **Event Engine** — process filesystem events into semantic change events
- **Auto-Commit Engine** — analyze change patterns and automatically create snapshots at intelligent boundaries
- **Auto-Branch Engine** — detect when work diverges and suggest or create branches automatically
- **Sync Engine** — push/pull trees between local and remote servers
- **Git Bridge** — import from Git, export to Git, and act as a transparent proxy to Git remotes
- **Permission Enforcer** — enforce the Access & Control Architecture at runtime
- **Storage Backend** — persist objects, trees, snapshots to disk efficiently

### 6.2 Server Crate

```
worktree-server/
├── src/
│   ├── lib.rs
│   ├── main.rs               // Entry point, service setup
│   ├── config/
│   │   ├── mod.rs
│   │   └── settings.rs       // Server configuration
│   ├── service/
│   │   ├── mod.rs
│   │   ├── daemon.rs         // Background process lifecycle
│   │   ├── install.rs        // Service installation per OS
│   │   └── health.rs         // Health checks
│   ├── watcher/
│   │   ├── mod.rs
│   │   ├── fs.rs             // Filesystem watcher (notify crate)
│   │   └── debounce.rs       // Event debouncing and batching
│   ├── engine/
│   │   ├── mod.rs
│   │   ├── event.rs          // Event processing pipeline
│   │   ├── auto_commit.rs    // Automatic snapshot creation
│   │   ├── auto_branch.rs    // Automatic branch management
│   │   └── rules.rs          // Configurable rules for automation
│   ├── sync/
│   │   ├── mod.rs
│   │   ├── push.rs           // Push trees to remote
│   │   ├── pull.rs           // Pull trees from remote
│   │   └── transport.rs      // Network transport (QUIC / TCP)
│   ├── git/
│   │   ├── mod.rs
│   │   ├── import.rs         // Git repo → Worktree tree conversion
│   │   ├── export.rs         // Worktree tree → Git repo conversion
│   │   ├── remote.rs         // Push/pull to Git remotes (GitHub, GitLab, etc.)
│   │   ├── mirror.rs         // Live bidirectional sync with a Git remote
│   │   └── submodule.rs      // Git submodule → nested tree conversion
│   ├── storage/
│   │   ├── mod.rs
│   │   ├── backend.rs        // Storage trait
│   │   ├── disk.rs           // On-disk object store
│   │   └── index.rs          // Object index for fast lookups
│   ├── auth/
│   │   ├── mod.rs
│   │   ├── session.rs        // Session management
│   │   └── enforcer.rs       // Permission enforcement at runtime
│   └── api/
│       ├── mod.rs
│       ├── grpc.rs           // gRPC API surface
│       └── handlers.rs       // Request handlers
├── Cargo.toml
└── README.md
```

---

## 7. The Client & SDK

### 7.1 CLI Client

The primary interface for developers. Designed to feel familiar to Git users but with Worktree's paradigm.

```
worktree-cli/
├── src/
│   ├── main.rs
│   ├── commands/
│   │   ├── mod.rs
│   │   ├── init.rs           // Initialize a new tree
│   │   ├── status.rs         // Show tree status
│   │   ├── snapshot.rs       // Manual snapshot creation
│   │   ├── branch.rs         // Branch management
│   │   ├── merge.rs          // Merge branches
│   │   ├── log.rs            // View snapshot history
│   │   ├── sync.rs           // Push / pull
│   │   ├── tree.rs           // Manage nested trees
│   │   ├── permission.rs     // Manage permissions
│   │   ├── git.rs            // Git interop commands (import, export, mirror)
│   │   └── server.rs         // Server management (start/stop/status)
│   └── output/
│       ├── mod.rs
│       ├── format.rs         // Output formatting
│       └── color.rs          // Terminal colors
├── Cargo.toml
└── README.md
```

**Example CLI Usage:**

```sh
# Initialize a root tree
wt init my-project

# Add a nested tree inside the root
wt tree add my-project/services/auth-service
wt tree add my-project/services/api-gateway
wt tree add my-project/libs/shared-models

# Check status (shows all trees)
wt status

# View auto-generated snapshots
wt log

# Create a manual snapshot in a specific tree
wt snapshot services/auth-service -m "Finalize OAuth2 flow"

# Set permissions
wt permission set services/auth-service --tenant backend-team --allow write
wt permission set libs/shared-models --user jane@company.com --allow read

# Sync
wt sync push
wt sync pull

# ── Git Compatibility ──

# Import an existing Git repo (full history preserved)
wt git import /path/to/my-git-repo
wt git import https://github.com/org/repo.git

# Export a Worktree tree back to a Git repo
wt git export services/auth-service --output /path/to/git-repo

# Add a Git remote and push/pull as if it were a Worktree remote
wt git remote add origin https://github.com/org/repo.git
wt git push origin main
wt git pull origin main

# Live mirror — keep a Worktree tree in bidirectional sync with a Git remote
wt git mirror services/auth-service --remote origin --branch main

# Clone a Git repo directly into a Worktree tree
wt git clone https://github.com/org/repo.git my-project
```

### 7.2 SDK

A Rust library crate that third-party tools, editors, CI systems, and GUIs can use to interact with Worktree programmatically.

```
worktree-sdk/
├── src/
│   ├── lib.rs
│   ├── client.rs             // High-level client API
│   ├── connection.rs         // Connect to local/remote server
│   ├── tree.rs               // Tree operations
│   ├── snapshot.rs           // Snapshot operations
│   ├── branch.rs             // Branch operations
│   ├── sync.rs               // Sync operations
│   ├── permission.rs         // Permission operations
│   └── error.rs              // Error types
├── Cargo.toml
└── README.md
```

**Example SDK Usage:**

```rust
use worktree_sdk::Client;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::connect_local().await?;

    // List all trees
    let trees = client.trees().list().await?;
    for tree in &trees {
        println!("{} ({})", tree.path(), tree.id());
    }

    // Get status of a specific tree
    let status = client.trees().status("services/auth-service").await?;
    println!("Changed files: {}", status.changed_files().len());

    // Create a snapshot
    client.snapshots()
        .create("services/auth-service", "Manual checkpoint")
        .await?;

    Ok(())
}
```

---

## 8. Nested Tree Model

This is Worktree's **defining feature**. Here's how it compares to Git:

### Git's Model (Flat)

```
my-project/                    ← .git lives here. ONE repo.
├── .git/
├── services/
│   ├── auth-service/          ← just a folder. no independent history.
│   └── api-gateway/           ← just a folder. no independent history.
├── libs/
│   └── shared-models/         ← just a folder. no independent history.
└── README.md
```

Everything shares one history. One branch applies to the entire repo. Permissions? Handled externally by GitHub/GitLab with CODEOWNERS hacks.

### Worktree's Model (Nested Trees)

```
my-project/                    ← Root Tree (has its own history, branches, permissions)
├── .worktree/                 ← Root tree metadata
├── services/
│   ├── auth-service/          ← Child Tree (independent history, branches, permissions)
│   │   └── .worktree/
│   └── api-gateway/           ← Child Tree (independent history, branches, permissions)
│       └── .worktree/
├── libs/
│   └── shared-models/         ← Child Tree (independent history, branches, permissions)
│       └── .worktree/
└── README.md
```

**Key properties of nested trees:**

1. **Independent History** — each tree tracks its own snapshots. Changes to `auth-service` don't pollute `api-gateway`'s history.
2. **Independent Branches** — `auth-service` can have a `feature/oauth` branch while `api-gateway` stays on `main`. No branch affects siblings unless explicitly linked.
3. **Independent Permissions** — the backend team owns `auth-service`, the platform team owns `api-gateway`, and `shared-models` is read-only for everyone except the architecture team.
4. **Parent Awareness** — the root tree knows about its children. A root-level snapshot can reference the state of all child trees (a "super-snapshot").
5. **Selective Sync** — pull only the trees you need. Don't clone the entire monorepo to work on one service.

### Tree Reference Model

```
RootTree
├── Manifest
│   ├── blob: README.md → hash_abc
│   ├── tree-ref: services/auth-service → TreeId(001) @ SnapshotId(latest)
│   ├── tree-ref: services/api-gateway → TreeId(002) @ SnapshotId(latest)
│   └── tree-ref: libs/shared-models → TreeId(003) @ SnapshotId(latest)
└── Snapshots
    ├── snap_001: { manifest_hash, parent: None, timestamp, author }
    └── snap_002: { manifest_hash, parent: snap_001, timestamp, author }
```

Each child tree has its **own** snapshot chain. The root tree's manifest contains **references** to child trees at specific snapshot IDs, not copies of their content.

---

## 9. Automatic Control Architecture

The ACA is the intelligence layer that makes Worktree fundamentally different from Git.

### 9.1 Change Detection

The server's filesystem watcher detects changes in real-time. But raw filesystem events (file modified, file created) are too noisy. The Event Engine processes raw events into **semantic changes**:

| Raw Event | Semantic Change |
|---|---|
| `auth-service/src/oauth.rs` modified | Code change in `auth-service` |
| `auth-service/Cargo.toml` modified | Dependency change in `auth-service` |
| `auth-service/src/oauth.rs` + `auth-service/src/tokens.rs` modified within 30s | Related code change batch in `auth-service` |
| `shared-models/src/user.rs` modified, then `auth-service/src/handlers.rs` modified | Cross-tree related change |

### 9.2 Auto-Snapshot Rules

The Auto-Commit Engine uses configurable rules to decide when to create snapshots:

- **Time-based**: Snapshot after N minutes of inactivity following changes
- **Size-based**: Snapshot after N files changed or N bytes changed
- **Boundary-based**: Snapshot when the user switches to a different tree
- **Semantic-based**: Snapshot when a logical unit of work is detected (e.g., a complete function added)
- **Manual override**: User can always force a snapshot or disable auto-snapshotting

### 9.3 Auto-Branch Rules

The Auto-Branch Engine manages branching:

- **Divergence Detection**: If changes in a tree start conflicting with a remote's main line, suggest a branch
- **Feature Detection**: If a new file pattern is detected (e.g., new module, new test suite), suggest a feature branch
- **Isolation**: If a user's permissions restrict them to a subtree, auto-branch their work within that scope

### 9.4 Configuration

```toml
# .worktree/config.toml

[auto_snapshot]
enabled = true
inactivity_timeout_secs = 300      # Snapshot after 5 min of inactivity
max_changed_files = 50             # Snapshot after 50 files changed
on_tree_switch = true              # Snapshot when user moves to another tree

[auto_branch]
enabled = true
divergence_threshold = 5           # Suggest branch after 5 conflicting changes
isolate_by_user = false            # Don't auto-branch per user by default

[watcher]
debounce_ms = 200
ignore_patterns = [
    "**/target/**",
    "**/node_modules/**",
    "**/.worktree/**",
]
```

---

## 10. Access & Control Architecture

Permissions are a **protocol-level concept**, not an afterthought bolted onto a hosting platform.

### 10.1 Permission Hierarchy

```
Global (Server-Level)
  └── Tenant (Organization)
       └── Team (Group of Users)
            └── User (Individual)
                 └── Tree (Specific Tree)
                      └── Branch (Specific Branch)
```

Permissions cascade downward. A tenant-level `write` permission grants write to all trees and branches within that tenant's scope, unless overridden at a lower level.

### 10.2 Permission Types

| Permission | Description |
|---|---|
| `read` | View tree contents and history |
| `write` | Modify files within a tree |
| `snapshot` | Create snapshots (manual or configure auto) |
| `branch` | Create, rename, delete branches |
| `merge` | Merge branches |
| `admin` | Manage permissions, configure tree settings |
| `sync` | Push/pull to/from remotes |

### 10.3 Policy Examples

```toml
# Tenant: "backend-team" has full access to backend services
[[policy]]
tenant = "backend-team"
tree = "services/auth-service"
permissions = ["read", "write", "snapshot", "branch", "merge", "sync"]

# Tenant: "frontend-team" can read shared models but not write
[[policy]]
tenant = "frontend-team"
tree = "libs/shared-models"
permissions = ["read"]

# User: "jane@company.com" is admin of the entire root tree
[[policy]]
user = "jane@company.com"
tree = "/"
permissions = ["admin"]

# User: "intern@company.com" can only read and write in their feature branch
[[policy]]
user = "intern@company.com"
tree = "services/api-gateway"
branch = "feature/intern-task"
permissions = ["read", "write", "snapshot"]
```

### 10.4 Enforcement

Permissions are enforced at **two layers**:

1. **Server-side** — the server rejects any operation that violates policy before it touches storage
2. **Client-side** — the CLI/SDK checks permissions before sending requests (fast-fail, but not a security boundary)

---

## 11. Native Git Compatibility

Worktree does not ask the world to abandon Git overnight. Instead, it provides **native, bidirectional compatibility with Git** at the protocol level. You can convert Git to Worktree, convert Worktree to Git, and even work against Git remotes transparently.

### 11.1 Design Philosophy

Git compatibility is **not an afterthought or a plugin**. It is a first-class subsystem with its own crate (`worktree-git`) and dedicated modules in both the protocol and server. The principle is simple:

> **Worktree can work anywhere Git works today, but Git cannot do what Worktree does.**

This means:
- Every Git repository in the world is a valid Worktree import source
- Every Worktree tree can be exported to a valid Git repository
- Worktree can push to and pull from GitHub, GitLab, Bitbucket, and any Git remote
- Teams can adopt Worktree one developer at a time — no coordinated migration

### 11.2 Git → Worktree Conversion

Import a Git repository into Worktree with full history preservation.

**What gets converted:**

| Git Concept | Worktree Equivalent |
|---|---|
| Commit | Snapshot |
| Tree object | Tree manifest entry |
| Blob object | Blob (content-addressable, re-hashed with BLAKE3) |
| Branch (`refs/heads/*`) | Branch |
| Tag (`refs/tags/*`) | Tagged Snapshot (annotated) or Snapshot alias (lightweight) |
| HEAD | Active branch pointer |
| `.gitignore` | `.worktreeignore` (auto-converted, original preserved) |
| `.gitmodules` + submodules | Nested Trees (each submodule becomes a child tree) |
| `.gitattributes` | Tree configuration (LFS patterns, merge drivers, etc.) |

**How it works:**

1. **Parse** — read the `.git/` directory using `libgit2` (via the `git2` Rust crate)
2. **Walk** — traverse the full commit graph from all branch tips
3. **Map** — convert each Git object to its Worktree equivalent, building a SHA-1 → BLAKE3 hash index
4. **Store** — write Worktree objects to the `.worktree/` object store
5. **Index** — build the hash mapping table so future Git interop can translate between the two worlds
6. **Submodules** — if the repo has `.gitmodules`, recursively import each submodule as a nested child tree with its own independent history

**History fidelity:**

- Commit messages → Snapshot messages (preserved verbatim)
- Commit authors / timestamps → Snapshot author / timestamp (preserved)
- Commit parents → Snapshot parents (preserved, including merge commits)
- The full DAG structure is maintained — you lose nothing

```
$ wt git import /path/to/my-git-repo

Scanning Git repository...
  Found 3,482 commits across 12 branches
  Found 4 submodules
  
Converting objects...
  ████████████████████████████████ 100%  (47,291 objects)

Converting submodules to nested trees...
  libs/shared-utils     → tree(001) [284 snapshots]
  libs/proto-defs       → tree(002) [91 snapshots]
  vendor/third-party    → tree(003) [12 snapshots]
  tools/scripts         → tree(004) [45 snapshots]

Building hash index (SHA-1 → BLAKE3)...
  ████████████████████████████████ 100%

Import complete.
  Root tree:    my-git-repo
  Child trees:  4
  Snapshots:    3,914 (3,482 root + 432 from submodules)
  Branches:     12
  Hash index:   47,291 entries
```

### 11.3 Worktree → Git Export

Export any Worktree tree (or the entire root tree) back to a valid Git repository.

**What gets exported:**

| Worktree Concept | Git Equivalent |
|---|---|
| Snapshot | Commit |
| Tree manifest | Tree object |
| Blob | Blob (re-hashed with SHA-1) |
| Branch | Branch (`refs/heads/*`) |
| Tagged Snapshot | Tag (`refs/tags/*`) |
| Active branch | HEAD |
| Nested child trees | Git submodules (`.gitmodules` auto-generated) |
| `.worktreeignore` | `.gitignore` (auto-converted) |
| Permissions / Tenants | Not exported (Git has no equivalent — logged as warning) |
| Auto-snapshots | Commits (optionally squashed to reduce noise) |

**Export modes:**

- **Full export** — every snapshot becomes a Git commit. Lossless round-trip.
- **Squashed export** — auto-snapshots are collapsed into logical commits. Cleaner history for Git consumers.
- **Shallow export** — only the latest N snapshots. Fast, for CI/CD pipelines that just need current state.
- **Single-tree export** — export one child tree as a standalone Git repo (detached from root).

```
$ wt git export services/auth-service --output /tmp/auth-git --mode squashed

Exporting tree: services/auth-service
  Mode: squashed (auto-snapshots collapsed)

Converting snapshots → commits...
  ████████████████████████████████ 100%  (127 snapshots → 34 commits)

Writing Git objects...
  ████████████████████████████████ 100%

Generating .gitignore from .worktreeignore...
  Done.

Export complete.
  Output: /tmp/auth-git
  Commits: 34
  Branches: 3 (main, feature/oauth, fix/token-refresh)

$ cd /tmp/auth-git && git log --oneline
a3f7c2d Finalize OAuth2 flow
b1e9a04 Add token refresh endpoint
...
```

### 11.4 Git Remote Bridge

Worktree can push to and pull from **any standard Git remote** — GitHub, GitLab, Bitbucket, self-hosted Gitea, bare Git servers, anything that speaks the Git transport protocol.

**How it works:**

1. **Register** a Git remote on a Worktree tree: `wt git remote add origin https://github.com/org/repo.git`
2. **On push** — the server converts the relevant Worktree snapshots to Git commits on the fly, then pushes them using the Git smart HTTP or SSH transport protocol
3. **On pull** — the server fetches Git commits from the remote, converts them to Worktree snapshots, and merges them into the local tree
4. **Hash index** — the SHA-1 ↔ BLAKE3 mapping table ensures objects are never re-downloaded or re-converted

**Authentication:**

Git remote auth uses the same credentials the user already has configured:
- SSH keys (`~/.ssh/`)
- Git credential helpers (`git credential-manager`, `git credential-osxkeychain`, etc.)
- Environment variables (`GIT_ASKPASS`, `GIT_SSH_COMMAND`)
- `.netrc` / credential store

No new auth setup required. If `git push` works, `wt git push` works.

**Selective push/pull:**

Because Worktree has nested trees, you can push/pull individual child trees to different Git remotes:

```
# auth-service syncs with one GitHub repo
wt git remote add auth-origin https://github.com/org/auth-service.git --tree services/auth-service
wt git push auth-origin main

# api-gateway syncs with a different repo
wt git remote add gw-origin https://github.com/org/api-gateway.git --tree services/api-gateway
wt git push gw-origin main

# Or push the entire root tree (including children) to a monorepo remote
wt git remote add mono-origin https://github.com/org/monorepo.git
wt git push mono-origin main
```

### 11.5 Live Mirror Mode

For teams where some developers use Git and others use Worktree, **mirror mode** keeps a Worktree tree in continuous bidirectional sync with a Git remote.

```
wt git mirror services/auth-service --remote origin --branch main
```

When mirror mode is active:
- Local Worktree snapshots are automatically converted and pushed to the Git remote
- New Git commits on the remote are automatically pulled, converted, and merged into the Worktree tree
- Conflicts are flagged and queued for resolution (same merge system as native Worktree conflicts)

This allows a **gradual team adoption**: developer A uses Worktree with all its features, developer B uses plain Git, and they both push/pull to the same GitHub repo without ever knowing the other exists.

### 11.6 Git Compatibility Crate

A dedicated crate handles all Git interop logic:

```
worktree-git/
├── src/
│   ├── lib.rs
│   ├── import/
│   │   ├── mod.rs
│   │   ├── repo.rs              // Read and parse a .git/ directory
│   │   ├── walker.rs            // Walk the Git commit graph
│   │   ├── converter.rs         // Git object → Worktree object conversion
│   │   └── submodule.rs         // .gitmodules → nested tree conversion
│   ├── export/
│   │   ├── mod.rs
│   │   ├── builder.rs           // Build a .git/ directory from Worktree objects
│   │   ├── converter.rs         // Worktree object → Git object conversion
│   │   ├── squash.rs            // Squash auto-snapshots for cleaner Git history
│   │   └── submodule.rs         // Nested trees → .gitmodules generation
│   ├── remote/
│   │   ├── mod.rs
│   │   ├── transport.rs         // Git smart HTTP + SSH transport
│   │   ├── push.rs              // Convert-and-push to Git remote
│   │   ├── pull.rs              // Fetch-and-convert from Git remote
│   │   ├── auth.rs              // Git credential helper integration
│   │   └── mirror.rs            // Bidirectional live sync
│   ├── hash_index/
│   │   ├── mod.rs
│   │   └── store.rs             // SHA-1 ↔ BLAKE3 persistent mapping table
│   └── config/
│       ├── mod.rs
│       ├── gitignore.rs         // .gitignore ↔ .worktreeignore conversion
│       └── gitattributes.rs     // .gitattributes parsing and mapping
├── Cargo.toml
└── README.md
```

### 11.7 Object Mapping In Detail

The core of Git compatibility is the bidirectional mapping between Git's object model and Worktree's object model.

**Git → Worktree:**

```
Git Commit
  ├── tree_hash (SHA-1)        →  Manifest (BLAKE3)
  ├── parent_hashes[]          →  Snapshot.parents[]
  ├── author + timestamp       →  Snapshot.author + Snapshot.timestamp
  ├── committer + timestamp    →  Snapshot.committer + Snapshot.timestamp
  └── message                  →  Snapshot.message

Git Tree
  ├── entries[]                →  Manifest.entries[]
  │   ├── name                 →  entry.name
  │   ├── mode (file/dir/etc)  →  entry.kind (Blob/Tree/Symlink)
  │   └── hash (SHA-1)         →  entry.hash (BLAKE3, looked up via hash index)

Git Blob
  └── content (bytes)          →  Blob.content (same bytes, different hash)

Git Tag (annotated)
  ├── target_hash              →  TaggedSnapshot.target
  ├── tagger + timestamp       →  TaggedSnapshot.author + TaggedSnapshot.timestamp
  └── message                  →  TaggedSnapshot.message
```

**Worktree → Git:**

The reverse mapping works identically. The only lossy conversions are:
- **Permissions** — Git has no equivalent. Dropped on export (with a warning).
- **Nested tree metadata** — Converted to `.gitmodules` entries, but the rich tree-ref model is simplified.
- **Auto-snapshot metadata** — The `auto: true` flag on snapshots has no Git equivalent. Optionally squashed.

### 11.8 Round-Trip Guarantee

Worktree guarantees that for any Git repository:

```
Git Repo → wt git import → wt git export --mode full → Git Repo'
```

`Git Repo'` is **semantically identical** to `Git Repo`:
- Same commit graph (same parent relationships)
- Same file contents at every commit
- Same branch names pointing to equivalent commits
- Same tag names pointing to equivalent commits
- Same commit messages, authors, and timestamps

The only difference is that SHA-1 hashes will differ (Git recomputes them). The content and structure are identical.

---

## 12. Platform Support

Worktree runs as a **native background process** on all major operating systems.

| Platform | Service Type | Install Location | Data Location |
|---|---|---|---|
| **Windows** | Windows Service (via `sc.exe` or NSSM) | `%ProgramFiles%\Worktree\` | `%APPDATA%\Worktree\` |
| **macOS** | launchd agent (`~/Library/LaunchAgents/`) | `/usr/local/bin/worktree-server` | `~/Library/Application Support/Worktree/` |
| **Linux** | systemd user service | `/usr/local/bin/worktree-server` | `~/.local/share/worktree/` |

### Installation Flow

```sh
# Install the CLI + Server
curl -fsSL https://install.worktree.dev | sh

# Or on Windows
winget install worktree

# The installer:
# 1. Places the `wt` CLI binary on PATH
# 2. Places the `worktree-server` binary in the appropriate location
# 3. Registers the server as a background service
# 4. Starts the server

# Verify
wt server status
# Output: Worktree Server v0.1.0 — running (PID 12345)
```

---

## 13. Project Structure

### Workspace Layout

```
worktree/
├── Cargo.toml                     # Workspace root
├── WORKTREE_PLAN.md               # This document
├── README.md
├── LICENSE
│
├── crates/
│   ├── worktree-protocol/         # The Protocol — data structures, formats, rules
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   ├── worktree-server/           # The Server — background daemon, engines, storage
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   ├── worktree-git/              # Git Compatibility — import, export, remote bridge
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   ├── worktree-cli/              # The CLI Client — user-facing commands
│   │   ├── Cargo.toml
│   │   └── src/
│   │
│   └── worktree-sdk/              # The SDK — library for third-party integrations
│       ├── Cargo.toml
│       └── src/
│
├── tests/                         # Integration tests
│   ├── protocol_tests/
│   ├── server_tests/
│   ├── git_compat_tests/          # Git round-trip and interop tests
│   └── e2e_tests/
│
├── docs/                          # Documentation
│   ├── protocol-spec.md
│   ├── server-architecture.md
│   ├── git-compatibility.md       # Git interop guide
│   ├── cli-reference.md
│   └── sdk-guide.md
│
└── scripts/                       # Build, install, CI scripts
    ├── install.sh
    ├── install.ps1
    └── ci.sh
```

### Dependency Graph

```
worktree-protocol   ← depends on nothing (pure data + logic)
       ▲
       ├──────────────────────┐
       │                      │
worktree-git        ← depends on worktree-protocol + git2 (libgit2)
       ▲
       │
worktree-server     ← depends on worktree-protocol + worktree-git
       ▲
       │
worktree-sdk        ← depends on worktree-protocol (communicates with server via API)
       ▲
       │
worktree-cli        ← depends on worktree-sdk
```

---

## 14. Milestones & Roadmap

### Phase 0 — Foundation (Weeks 1–4)

- [ ] Set up Cargo workspace with all five crates (protocol, server, git, cli, sdk)
- [ ] Define core protocol types: `Tree`, `Blob`, `Snapshot`, `Branch`, `Manifest`
- [ ] Implement BLAKE3 content-addressable hashing
- [ ] Implement basic wire format (encode/decode) with serde + bincode or custom format
- [ ] Implement on-disk storage backend (object store)
- [ ] Basic `wt init` — create a root tree with `.worktree/` directory
- [ ] Basic `wt status` — scan tree and show changed files
- [ ] Unit tests for all protocol types

### Phase 1 — Single Tree Operations (Weeks 5–8)

- [ ] Implement snapshot creation (manual)
- [ ] Implement snapshot history (`wt log`)
- [ ] Implement branching (`wt branch create/list/switch`)
- [ ] Implement basic diff (`wt diff`)
- [ ] Implement basic merge (fast-forward + three-way)
- [ ] Implement `.worktree/config.toml` parsing
- [ ] Integration tests for single-tree workflows

### Phase 1.5 — Git Import (Weeks 9–10)

- [ ] Implement Git repo reader using `git2` crate (parse `.git/` directory)
- [ ] Implement commit graph walker
- [ ] Implement Git object → Worktree object converter (commit→snapshot, tree→manifest, blob→blob)
- [ ] Implement SHA-1 → BLAKE3 hash index (persistent mapping table)
- [ ] Implement `wt git import` CLI command (local path)
- [ ] Implement `.gitignore` → `.worktreeignore` conversion
- [ ] Round-trip test: import a Git repo, verify all history is preserved
- [ ] Test with real-world repos (small, medium, large)

### Phase 2 — The Server (Weeks 11–16)

- [ ] Implement server binary with start/stop lifecycle
- [ ] Implement filesystem watcher (using `notify` crate)
- [ ] Implement event debouncing and batching
- [ ] Implement event engine (raw events → semantic changes)
- [ ] Implement auto-snapshot engine with configurable rules
- [ ] Implement server installation as background service (Windows, macOS, Linux)
- [ ] Implement gRPC API surface for client ↔ server communication
- [ ] `wt server start/stop/status` commands
- [ ] Integration tests for server lifecycle and event processing

### Phase 3 — Nested Trees (Weeks 17–22)

- [ ] Implement nested tree creation (`wt tree add`)
- [ ] Implement tree-ref in manifests (parent tree references child trees)
- [ ] Implement independent history per tree
- [ ] Implement independent branching per tree
- [ ] Implement super-snapshots (root snapshot referencing child tree states)
- [ ] Implement selective operations scoped to a specific tree
- [ ] Implement `wt tree list/status/remove`
- [ ] Implement Git submodule → nested tree conversion during import
- [ ] Integration tests for nested tree workflows

### Phase 4 — Git Export & Remote Bridge (Weeks 23–28)

- [ ] Implement Worktree object → Git object converter (snapshot→commit, manifest→tree, blob→blob)
- [ ] Implement `wt git export` with full, squashed, and shallow modes
- [ ] Implement nested tree → `.gitmodules` generation on export
- [ ] Implement Git remote registration (`wt git remote add`)
- [ ] Implement Git smart HTTP transport (push)
- [ ] Implement Git smart HTTP transport (pull / fetch)
- [ ] Implement Git SSH transport
- [ ] Implement Git credential helper integration (SSH keys, credential managers)
- [ ] Implement `wt git push` and `wt git pull` commands
- [ ] Implement selective push/pull (individual trees to different Git remotes)
- [ ] Round-trip test: import → export → verify identical to original
- [ ] Integration tests against GitHub, GitLab (test repos)

### Phase 5 — Access & Control (Weeks 29–34)

- [ ] Implement identity model (Tenant, Team, User)
- [ ] Implement permission schema and policy evaluation
- [ ] Implement server-side permission enforcement
- [ ] Implement `wt permission set/get/list` commands
- [ ] Implement session/authentication for multi-user scenarios
- [ ] Integration tests for permission enforcement

### Phase 6 — Worktree-Native Sync & Networking (Weeks 35–42)

- [ ] Implement Worktree remote registration (`wt remote add`)
- [ ] Implement push (send snapshots + objects to remote Worktree server)
- [ ] Implement pull (fetch snapshots + objects from remote Worktree server)
- [ ] Implement transport layer (QUIC preferred, TCP fallback)
- [ ] Implement selective sync (pull only specific trees)
- [ ] Implement conflict detection on pull
- [ ] End-to-end tests for multi-server sync

### Phase 7 — Git Live Mirror & Auto-Branch Intelligence (Weeks 43–50)

- [ ] Implement `wt git mirror` — bidirectional live sync with a Git remote
- [ ] Implement automatic conflict detection during mirror sync
- [ ] Implement `wt git clone` — clone a Git repo directly into a Worktree tree
- [ ] Implement divergence detection
- [ ] Implement auto-branch engine with configurable rules
- [ ] Implement cross-tree change detection
- [ ] Implement smart merge suggestions
- [ ] Configuration UI in CLI for automation rules

### Phase 8 — SDK & Ecosystem (Weeks 51–56)

- [ ] Stabilize SDK public API
- [ ] Write SDK documentation and examples
- [ ] Implement SDK connection management (local + remote)
- [ ] Implement SDK git interop methods (import/export/remote operations)
- [ ] Create example integrations (VS Code extension scaffold, CI plugin scaffold)
- [ ] Publish SDK crate

### Phase 9 — Polish & Release (Weeks 57–62)

- [ ] Performance benchmarking and optimization (including Git conversion perf)
- [ ] Git compatibility test suite against top 100 GitHub repos by size/complexity
- [ ] Security audit of permission enforcement and transport
- [ ] Cross-platform testing matrix (Windows, macOS, Linux)
- [ ] Installer scripts and distribution packages
- [ ] Documentation site
- [ ] Public beta release

---

## 15. Technical Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Language** | Rust | Performance, memory safety, cross-platform, strong type system |
| **Hashing** | BLAKE3 | Faster than SHA-256, tree-hashing mode, widely adopted in new systems |
| **Serialization** | Bincode (internal), JSON (API debug mode) | Bincode is fast and compact for wire format; JSON for human debugging |
| **Filesystem Watching** | `notify` crate | Cross-platform, well-maintained, supports all target OSes |
| **Git Interop** | `git2` crate (libgit2 bindings) | Battle-tested C library with safe Rust bindings, handles all Git formats and transports |
| **API** | gRPC (tonic) | Strongly typed, efficient, streaming support, great Rust ecosystem |
| **Transport** | QUIC (quinn crate) with TCP fallback | Modern, multiplexed, encrypted by default, handles NAT better |
| **Storage** | Content-addressable object store on disk | Proven model (Git uses it), deduplication built-in |
| **Hash Index** | LMDB (via `heed` crate) or SQLite | Fast persistent key-value store for SHA-1 ↔ BLAKE3 mapping table |
| **Async Runtime** | Tokio | Industry standard for async Rust, required by tonic and quinn |
| **CLI Framework** | clap | De facto standard for Rust CLIs, derive macros for ergonomics |
| **Service Management** | Platform-native (systemd, launchd, Windows Service) | Reliable, users expect native service behavior |

---

## 16. Open Questions

These are decisions and designs that need further exploration:

1. **Snapshot Granularity** — How fine-grained should auto-snapshots be? Every save? Every logical change? Need user research.

2. **Garbage Collection** — With auto-snapshots, storage will grow fast. What's the GC strategy? Keep last N snapshots? Compact old snapshots into summary snapshots?

3. **Large File Handling** — Should Worktree have a native LFS-like system? Or handle large files differently from the start (chunked storage, lazy loading)? How does this interact with Git LFS during import/export?

4. **Offline Mode** — How does the system behave when the server is down? Should the CLI have a degraded local-only mode?

5. **Git Mirror Conflict Policy** — When live mirror mode encounters a conflict between a local Worktree change and a remote Git change, what is the default resolution? Auto-branch? Pause sync? Queue for manual resolution?

6. **Conflict Resolution UX** — Git's conflict markers are famously terrible. What's Worktree's conflict resolution experience?

7. **Hook System** — Should Worktree have pre/post-snapshot hooks like Git's pre/post-commit hooks? Or a more modern event-subscription model?

8. **Remote Protocol** — Should remote Worktree servers speak the same gRPC API as local, or a dedicated sync protocol?

9. **Web Interface** — Should the server expose a web UI for tree browsing, history viewing, and permission management?

10. **Encryption at Rest** — Should tree contents be encrypted on disk? Per-tree encryption keys?

11. **Git Shallow Clone Support** — When importing from a Git remote, should Worktree support shallow clones (partial history)? Or always require full history?

12. **Hash Index Storage Limits** — The SHA-1 ↔ BLAKE3 mapping table will grow with every imported Git repo. Should it be per-tree, global, or garbage-collected when a Git remote is removed?

13. **Git Hook Compatibility** — Should `wt git push` trigger the same Git hooks (pre-push, post-receive) that a native `git push` would? Or should Worktree's own event system replace them?

14. **Partial Git Export** — When exporting a nested tree that references blobs in sibling trees, how are those cross-tree references handled in the Git output?

15. **GitHub/GitLab API Integration** — Beyond Git transport, should Worktree integrate with platform APIs (PRs, issues, CI triggers) or stay strictly at the Git protocol level?

---

## Summary

Worktree is not a Git wrapper. It is not a Git extension. It is a **replacement** — one that speaks Git's language when it needs to.

| | Git | Worktree |
|---|---|---|
| Model | Flat repo, manual everything | Nested trees, automatic intelligence |
| Branching | Global, manual | Per-tree, automatic or manual |
| Permissions | External (GitHub/GitLab) | Built-in, protocol-level |
| Server | None (dumb transport) | Persistent, intelligent background process |
| Monorepo | Painful (subtree, submodule, sparse-checkout) | Native (nested trees are the model) |
| Performance | Degrades at scale | Designed for scale from day one (Rust, BLAKE3, selective sync) |
| Platform | Linux-first, Windows second-class | First-class Windows, Linux, macOS |
| Git Interop | N/A — it *is* Git | Native bidirectional conversion, push/pull to any Git remote, live mirror mode |
| Adoption Path | N/A | Incremental — import existing repos, work alongside Git users, no big-bang migration |

**Worktree: Trees all the way down. And yes, it still talks to GitHub.**
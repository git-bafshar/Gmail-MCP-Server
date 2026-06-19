# Gmail MCP Server Documentation

## Update Draft Feature Implementation

This directory contains comprehensive documentation for adding the `update_draft` tool to the Gmail MCP Server.

### Documentation Files

1. **[implementation-checklist.md](./implementation-checklist.md)** - START HERE
   - Complete step-by-step checklist from setup to upstream PR
   - Includes Strategy Decision: Test in fork first (Option A - PREFERRED)
   - Checkboxes for tracking progress through all phases
   - Phase 0: Pre-implementation setup
   - Phase 1: Code implementation
   - Phase 2: Build and test
   - Phase 3: Commit and push
   - Phase 4: Merge to your fork's main (PREFERRED)
   - Production testing period (1-2 weeks)
   - Phase 5: Pull request to upstream (after validation)
   - Phase 6: Post-merge cleanup

2. **[mcp-server-analysis.md](./mcp-server-analysis.md)** - Technical Reference
   - Current MCP server code structure analysis
   - Detailed implementation code with line numbers
   - Gmail API details and limitations
   - Attachment handling discussion
   - Testing plan

3. **[pr-workflow-setup.md](./pr-workflow-setup.md)** - Git Workflow Guide
   - Git remote setup (origin vs upstream)
   - Fork synchronization procedures
   - Pull request creation process
   - PR description template
   - Pre-PR checklist
   - Post-merge cleanup procedures
   - Ongoing maintenance strategy

## Quick Start

### Working Directory

All commands in the documentation assume you're in:
```bash
cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
```

### Strategy: Test in Fork First (Option A)

**Chosen Approach:** Implement → Merge to your fork → Test in production → PR to upstream

**Why:** Allows real-world validation before proposing changes to upstream maintainer.

### First Steps

1. **Read** `implementation-checklist.md` from top to bottom
2. **Execute** Phase 0 (git setup):
   ```bash
   git remote add upstream https://github.com/GongRzhe/Gmail-MCP-Server.git
   git fetch upstream
   ```
3. **Follow** the checklist phase by phase, checking off items as you complete them

## What is update_draft?

A new MCP tool that enables programmatic updates to existing Gmail drafts while preserving thread context and metadata.

### Current Limitation

The Gmail MCP server can create drafts (`draft_email`) but cannot update them. This creates friction for workflows where drafts start with placeholder content (e.g., "Agent instructions: draft email about X") that needs to be replaced with final content.

### Proposed Solution

New tool `update_draft` that:
- Takes draft ID and optional fields for partial updates
- Fetches existing draft to preserve unchanged fields
- Merges new values with existing
- Updates via Gmail API's `users.drafts.update` endpoint

## Repository Structure

```
/Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server/
├── src/
│   ├── index.ts              # Main server file (where you'll add update_draft)
│   ├── label-manager.ts      # Label utilities
│   ├── filter-manager.ts     # Filter utilities
│   └── utl.ts               # Email creation utilities
├── dist/                     # Compiled JavaScript (auto-generated)
├── docs/                     # THIS DIRECTORY - Implementation documentation
│   ├── README.md            # This file
│   ├── implementation-checklist.md
│   ├── mcp-server-analysis.md
│   └── pr-workflow-setup.md
├── package.json             # Dependencies and build scripts
└── tsconfig.json            # TypeScript configuration
```

## Build Commands

```bash
# Build TypeScript to JavaScript
npm run build

# After building, restart Claude Code to reload the MCP server
```

## Git Remotes

- **origin:** `https://github.com/git-bafshar/Gmail-MCP-Server.git` (your fork)
- **upstream:** `https://github.com/GongRzhe/Gmail-MCP-Server.git` (original, to be added)

## Questions?

- **Conceptual questions about MCP:** See notes in this README or ask Claude
- **Git workflow questions:** See `pr-workflow-setup.md`
- **Implementation questions:** See `mcp-server-analysis.md`
- **Step-by-step guidance:** Follow `implementation-checklist.md`

## What is an MCP Server?

**MCP (Model Context Protocol) Server** is a local process that runs on your machine and acts as a bridge between Claude and external systems.

### Key Characteristics

- **Local process:** Runs only on your machine when Claude Code is running
- **Stdio communication:** Communicates with Claude via standard input/output (not HTTP)
- **Started automatically:** Claude Code launches it as a child process
- **No network:** Not a web server - never accessible from internet
- **Your credentials:** Uses your OAuth tokens to access Gmail as you

### How It Works

```
┌─────────────┐         stdio          ┌──────────────┐       Gmail API      ┌─────────┐
│ Claude Code │ ◄──────────────────────► │  Gmail MCP   │ ◄────────────────────► │  Gmail  │
│             │  (function calls/JSON)  │    Server    │   (OAuth/HTTPS)      │         │
└─────────────┘                         └──────────────┘                      └─────────┘
```

1. Claude Code starts Gmail MCP server as child process
2. You ask Claude to perform Gmail operation
3. Claude sends JSON message to MCP server via stdio
4. MCP server calls Gmail API with your OAuth credentials
5. MCP server returns response to Claude via stdio
6. Claude presents result to you

### Updates

When you modify code and rebuild:
1. Run `npm run build` to compile TypeScript → JavaScript
2. Restart Claude Code to reload the updated server
3. New functionality immediately available

## Next Steps

Start with `implementation-checklist.md` and work through it phase by phase!

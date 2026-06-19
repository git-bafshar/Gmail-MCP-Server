# PR Workflow Setup: Adding update_draft to Gmail MCP Server

## Current Repository State

**Local Repository:** `/Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server`

**Git Remotes:**
- **origin:** `https://github.com/git-bafshar/Gmail-MCP-Server.git` (your fork)
- **upstream:** Not configured yet (needs to be added)

**Original Upstream:** `https://github.com/GongRzhe/Gmail-MCP-Server`

**Current Branch:** `main`

**Status:**
- Has uncommitted changes in `src/index.ts`, `package.json`, `package-lock.json`
- Changes appear to be minor (removed empty capabilities object)

## Required Setup Steps

### 1. Add Upstream Remote

Configure the original repository as upstream to stay in sync:

```bash
cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
git remote add upstream https://github.com/GongRzhe/Gmail-MCP-Server.git
git fetch upstream
```

**Verification:**
```bash
git remote -v
# Should show:
# origin    https://github.com/git-bafshar/Gmail-MCP-Server.git (fetch)
# origin    https://github.com/git-bafshar/Gmail-MCP-Server.git (push)
# upstream  https://github.com/GongRzhe/Gmail-MCP-Server.git (fetch)
# upstream  https://github.com/GongRzhe/Gmail-MCP-Server.git (push)
```

### 2. Sync Your Fork with Upstream

Before starting new work, ensure your fork is up-to-date:

```bash
# Fetch latest changes from upstream
git fetch upstream

# Make sure you're on main branch
git checkout main

# Merge upstream changes into your main
git merge upstream/main

# Push updated main to your fork
git push origin main
```

### 3. Handle Existing Uncommitted Changes

**Current uncommitted changes:**
- `src/index.ts` - Removed empty capabilities object
- `package.json` - Unknown changes
- `package-lock.json` - Likely dependency updates

**Decision needed:** Should these changes be:
1. **Committed separately first** (if they're intentional improvements)
2. **Stashed for now** (if they're experimental)
3. **Discarded** (if they're accidental)

**Recommended approach:**
```bash
# Option 1: Commit existing changes first
git add -A
git commit -m "chore: remove empty capabilities object from server config"
git push origin main

# Option 2: Stash for later
git stash save "WIP: server config cleanup"

# Option 3: Discard changes
git restore src/index.ts package.json package-lock.json
```

### 4. Create Feature Branch for update_draft

```bash
# Create and switch to new branch
git checkout -b feature/update-draft

# Verify you're on the new branch
git branch
```

**Branch naming convention:**
- Use `feature/` prefix for new features
- Use `fix/` prefix for bug fixes
- Use descriptive names: `feature/update-draft`, `feature/draft-update-support`, etc.

## Implementation Workflow

### Phase 1: Implement update_draft Feature

1. **Make code changes** following `mcp-server-analysis.md`
   - Add `UpdateDraftSchema` to `src/index.ts`
   - Add `handleUpdateDraft()` function
   - Register tool in `ListToolsRequestSchema`
   - Add case statement in `CallToolRequestSchema`

2. **Test locally**
   ```bash
   npm run build
   # Restart Claude Code to reload MCP server
   # Test update_draft functionality
   ```

3. **Commit changes**
   ```bash
   git add src/index.ts
   git commit -m "feat: add update_draft tool for modifying existing Gmail drafts

   - Add UpdateDraftSchema with optional fields for partial updates
   - Implement handleUpdateDraft function that merges new values with existing
   - Preserve thread context and reply-to headers
   - Add tool registration and request handler

   Resolves the limitation where drafts with agent instructions
   could not be programmatically updated."
   ```

### Phase 2: Push to Your Fork

```bash
# Push feature branch to your fork
git push origin feature/update-draft
```

### Phase 3: Create Pull Request to Your Fork's Main

This is optional but recommended for testing:

1. Go to `https://github.com/git-bafshar/Gmail-MCP-Server`
2. Click "Compare & pull request" for `feature/update-draft` branch
3. Set base repository: `git-bafshar/Gmail-MCP-Server` base: `main`
4. Review changes, add description
5. Create PR and merge to your main branch

**OR skip this and merge directly:**
```bash
git checkout main
git merge feature/update-draft
git push origin main
```

### Phase 4: Create Pull Request to Upstream

1. Go to `https://github.com/GongRzhe/Gmail-MCP-Server`
2. Click "Fork" → Navigate to your fork
3. Click "Contribute" → "Open pull request"
4. Set:
   - **base repository:** `GongRzhe/Gmail-MCP-Server`
   - **base branch:** `main` (or whatever their default branch is)
   - **head repository:** `git-bafshar/Gmail-MCP-Server`
   - **compare branch:** `main` (or `feature/update-draft` if you want to PR the branch directly)

5. Write comprehensive PR description (see template below)
6. Submit pull request

## Pull Request Description Template

```markdown
## Description

Adds `update_draft` tool to enable programmatic updates of existing Gmail drafts while preserving thread context and existing metadata.

## Motivation

The current Gmail MCP server supports creating drafts via `draft_email` but cannot update existing drafts. This creates friction in workflows where drafts are created with placeholder content (e.g., "Agent instructions: draft email about X") that needs to be replaced with final content.

## Changes

- **New Tool:** `update_draft`
  - Takes `draftId` (required) and optional fields for partial updates
  - Fetches existing draft to preserve unchanged fields
  - Merges new values with existing (new values take precedence)
  - Re-encodes and updates via `gmail.users.drafts.update()` API

- **Schema:** `UpdateDraftSchema` with optional fields for:
  - `to`, `cc`, `bcc` (recipients)
  - `subject`, `body`, `htmlBody` (content)
  - `mimeType` (content type)

- **Handler:** `handleUpdateDraft()` function with:
  - Error handling for invalid draft IDs
  - Header extraction and merging logic
  - Thread context preservation

## Use Case Example

```typescript
// 1. Create draft with placeholder
mcp__gmail__draft_email({
  to: ["support@example.com"],
  subject: "Support Request",
  body: "Agent instructions: Draft email about login issue"
})

// 2. Update draft with final content
mcp__gmail__update_draft({
  draftId: "r-1234567890",
  body: "I'm experiencing a login loop issue on my account..."
})
```

## Limitations

- **Attachments:** Current implementation does NOT preserve attachments when updating. Attachments from the original draft will be lost. This is a known limitation due to the complexity of re-encoding multipart MIME messages with attachments.
- **Future enhancement:** Could add attachment preservation using nodemailer for full MIME reconstruction.

## Testing

Tested locally with:
- [x] Update body only (preserves subject/recipients)
- [x] Update subject only (preserves body/recipients)
- [x] Update all fields simultaneously
- [x] Invalid draft ID returns clear error
- [x] Thread context preserved for reply drafts

## Compatibility

- No breaking changes to existing tools
- Uses existing utilities (`createEmailMessage`, header extraction patterns)
- Follows established error handling patterns
- Compatible with MCP SDK 1.24.0

## Related Issues

[Link to any related issues if they exist]
```

## Pre-PR Checklist

Before submitting the PR to upstream, verify:

### Code Quality
- [ ] Code follows existing style and patterns in `index.ts`
- [ ] Uses Zod schema validation consistently
- [ ] Error handling matches existing tool patterns
- [ ] Function is documented with JSDoc comments (if other functions have them)

### Testing
- [ ] Tested locally with real Gmail account
- [ ] Verified draft updates work with partial and full updates
- [ ] Confirmed thread context preservation
- [ ] Tested error cases (invalid draft ID, etc.)

### Documentation
- [ ] Update README.md if it lists available tools
- [ ] Add example usage to documentation
- [ ] Note limitations (attachment handling)

### Git Hygiene
- [ ] Feature branch is up-to-date with upstream main
- [ ] Commits are clean and well-described
- [ ] No merge conflicts
- [ ] No debugging code, console.logs, or commented code
- [ ] Package.json version NOT bumped (maintainer will do this)

### Build & Deploy
- [ ] `npm run build` succeeds without errors
- [ ] TypeScript compilation has no errors or warnings
- [ ] No new dependencies added (uses existing googleapis)

## Potential Issues & Mitigation

### Issue 1: Draft ID vs Message ID Confusion

**Problem:** The `read_email` tool returns message ID, but `update_draft` needs draft ID.

**Solution:** Document in PR that users should use `drafts.list()` to get draft IDs, not just `read_email`. Consider adding example or helper function.

**Could add to PR:**
```typescript
// Helper function to get draft ID from message ID
async function getDraftIdFromMessageId(messageId: string): Promise<string | null> {
    const drafts = await gmail.users.drafts.list({ userId: 'me' });
    const draft = drafts.data.drafts?.find(d => d.message?.id === messageId);
    return draft?.id || null;
}
```

### Issue 2: Attachment Loss

**Problem:** Updating drafts loses attachments.

**Mitigation:**
- Clearly document this limitation
- Suggest workaround: download attachments before update, re-attach after
- Or: mark as future enhancement

### Issue 3: Upstream May Request Changes

**Possible feedback:**
- Code style adjustments
- Additional error handling
- Tests (if they have a test suite)
- Documentation in specific format

**Response:**
- Be responsive to feedback
- Make requested changes in new commits to the same branch
- Push updates: `git push origin feature/update-draft`
- GitHub will automatically update the PR

## Post-Merge Actions

After upstream merges your PR:

1. **Sync your fork:**
   ```bash
   git checkout main
   git fetch upstream
   git merge upstream/main
   git push origin main
   ```

2. **Delete feature branch:**
   ```bash
   git branch -d feature/update-draft
   git push origin --delete feature/update-draft
   ```

3. **Update local MCP installation:**
   ```bash
   cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
   git pull origin main
   npm install
   npm run build
   ```

## Ongoing Maintenance

To keep your fork in sync with upstream:

```bash
# Periodically (weekly/monthly)
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
```

Set up a reminder to sync regularly, especially before starting new features.

## Alternative: Test Before Upstream PR

If you want to test thoroughly in your fork first:

1. Merge `feature/update-draft` → `git-bafshar/Gmail-MCP-Server:main`
2. Use it in production for a few days/weeks
3. Once stable, create PR to `GongRzhe/Gmail-MCP-Server:main`

This gives you confidence the feature works well before proposing to upstream.

## References

- Your fork: https://github.com/git-bafshar/Gmail-MCP-Server
- Upstream: https://github.com/GongRzhe/Gmail-MCP-Server
- GitHub PR docs: https://docs.github.com/en/pull-requests
- Contributing guide (check if upstream has one): https://github.com/GongRzhe/Gmail-MCP-Server/blob/main/CONTRIBUTING.md

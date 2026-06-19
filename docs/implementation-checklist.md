# Implementation Checklist: update_draft Feature

This checklist walks through implementing the `update_draft` tool from setup to upstream PR submission.

## Strategy Decision: Test in Fork First (Option A - PREFERRED)

**Chosen Approach:** Test the feature in your fork's main branch before submitting to upstream.

**Rationale:**
- Allows real-world testing with your Gmail workflow for several days/weeks
- Identifies edge cases and bugs before proposing to upstream
- Demonstrates stability and confidence in the implementation
- Reduces risk of upstream maintainer requesting changes due to bugs
- You can iterate quickly on your fork without waiting for upstream review

**Alternative (Option B - Not Chosen):** PR directly from feature branch to upstream
- Faster but riskier
- No production validation before proposing changes
- May require multiple rounds of fixes if issues found

**Workflow Path:**
1. Implement on feature branch
2. Merge to `git-bafshar/Gmail-MCP-Server:main`
3. Test in production for 1-2 weeks
4. Once stable, create PR to `GongRzhe/Gmail-MCP-Server:main`

## Phase 0: Pre-Implementation Setup

### Git Configuration

- [ ] **Add upstream remote**
  ```bash
  cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
  git remote add upstream https://github.com/GongRzhe/Gmail-MCP-Server.git
  git fetch upstream
  ```

- [ ] **Handle existing uncommitted changes**

  Current changes in: `src/index.ts`, `package.json`, `package-lock.json`

  **Decision:** Choose one option:

  - [ ] Option A: Commit them
    ```bash
    git add -A
    git commit -m "chore: remove empty capabilities object"
    git push origin main
    ```

  - [ ] Option B: Stash them
    ```bash
    git stash save "WIP: server config cleanup"
    ```

  - [ ] Option C: Discard them
    ```bash
    git restore src/index.ts package.json package-lock.json
    ```

- [ ] **Sync fork with upstream**
  ```bash
  git checkout main
  git fetch upstream
  git merge upstream/main
  git push origin main
  ```

- [ ] **Create feature branch**
  ```bash
  git checkout -b feature/update-draft
  git branch  # Verify you're on feature/update-draft
  ```

## Phase 1: Code Implementation

### Step 1: Add Schema Definition

- [ ] **Open** `/Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server/src/index.ts`

- [ ] **Locate** the schema definitions section (around line 194, after `SendEmailSchema`)

- [ ] **Add** the following code:
  ```typescript
  const UpdateDraftSchema = z.object({
      draftId: z.string().describe("ID of the draft to update"),
      to: z.array(z.string()).optional().describe("List of recipient email addresses"),
      subject: z.string().optional().describe("Email subject"),
      body: z.string().optional().describe("Email body content"),
      htmlBody: z.string().optional().describe("HTML version of the email body"),
      mimeType: z.enum(['text/plain', 'text/html', 'multipart/alternative']).optional().describe("Email content type"),
      cc: z.array(z.string()).optional().describe("List of CC recipients"),
      bcc: z.array(z.string()).optional().describe("List of BCC recipients"),
  });
  ```

### Step 2: Register Tool

- [ ] **Locate** the `ListToolsRequestSchema` handler (around line 340-438)

- [ ] **Find** the tools array and the `draft_email` tool entry (around line 349)

- [ ] **Add** after the `draft_email` entry:
  ```typescript
  {
      name: "update_draft",
      description: "Updates the content of an existing Gmail draft while preserving thread context and attachments metadata",
      inputSchema: zodToJsonSchema(UpdateDraftSchema),
  },
  ```

### Step 3: Implement Handler Function

- [ ] **Locate** the `CallToolRequestSchema` handler (around line 441)

- [ ] **Find** the existing `handleEmailAction` function (around line 444)

- [ ] **Add** the following function **BEFORE** `handleEmailAction`:
  ```typescript
  async function handleUpdateDraft(validatedArgs: any) {
      try {
          // 1. Fetch existing draft to get current values
          const existingDraftResponse = await gmail.users.drafts.get({
              userId: 'me',
              id: validatedArgs.draftId,
              format: 'full'
          });

          const existingMessage = existingDraftResponse.data.message;
          if (!existingMessage || !existingMessage.payload) {
              throw new Error('Could not retrieve existing draft');
          }

          const existingHeaders = existingMessage.payload.headers || [];

          // Helper to extract header value
          const getHeader = (name: string): string | undefined => {
              const header = existingHeaders.find(h => h.name?.toLowerCase() === name.toLowerCase());
              return header?.value;
          };

          // 2. Merge new values with existing values (new values take precedence)
          const to = validatedArgs.to || (getHeader('To')?.split(',').map((e: string) => e.trim()) || []);
          const cc = validatedArgs.cc || (getHeader('Cc')?.split(',').map((e: string) => e.trim()) || undefined);
          const bcc = validatedArgs.bcc || (getHeader('Bcc')?.split(',').map((e: string) => e.trim()) || undefined);
          const subject = validatedArgs.subject !== undefined ? validatedArgs.subject : (getHeader('Subject') || '');
          const body = validatedArgs.body !== undefined ? validatedArgs.body : '';
          const mimeType = validatedArgs.mimeType || 'text/plain';
          const threadId = existingMessage.threadId;

          // 3. Build the updated message using existing utility
          const emailData = {
              to,
              cc,
              bcc,
              subject,
              body,
              htmlBody: validatedArgs.htmlBody,
              mimeType,
              threadId,
              inReplyTo: getHeader('In-Reply-To')
          };

          const message = createEmailMessage(emailData);

          // 4. Encode and update the draft
          const encodedMessage = Buffer.from(message).toString('base64')
              .replace(/\+/g, '-')
              .replace(/\//g, '_')
              .replace(/=+$/, '');

          const response = await gmail.users.drafts.update({
              userId: 'me',
              id: validatedArgs.draftId,
              requestBody: {
                  message: {
                      raw: encodedMessage,
                      threadId: threadId
                  },
              },
          });

          return {
              content: [{
                  type: "text",
                  text: `Draft updated successfully with ID: ${response.data.id}`
              }]
          };

      } catch (error: any) {
          return {
              content: [{
                  type: "text",
                  text: `Error updating draft: ${error.message}`
              }],
              isError: true,
          };
      }
  }
  ```

### Step 4: Add Tool Invocation Case

- [ ] **Locate** the switch statement in the `CallToolRequestSchema` handler (around line 599)

- [ ] **Find** the `case "draft_email":` statement

- [ ] **Add** after the `draft_email` case (around line 604):
  ```typescript
  case "update_draft": {
      const validatedArgs = UpdateDraftSchema.parse(args);
      return await handleUpdateDraft(validatedArgs);
  }
  ```

### Step 5: Save and Verify Code

- [ ] **Save** `src/index.ts`

- [ ] **Quick syntax check** (optional but recommended):
  ```bash
  cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
  npx tsc --noEmit
  ```

## Phase 2: Build and Test

### Build

- [ ] **Build the project**
  ```bash
  cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
  npm run build
  ```

- [ ] **Verify build succeeded** (check for `dist/index.js`)
  ```bash
  ls -la dist/index.js
  ```

### Local Testing

- [ ] **Restart Claude Code** to reload MCP server

- [ ] **Test basic draft update:**

  Create a test draft in Gmail with placeholder content, then:
  ```typescript
  // Get draft list to find draft ID
  mcp__gmail__search_emails({ query: "in:drafts", maxResults: 5 })

  // Update the draft
  mcp__gmail__update_draft({
      draftId: "r-XXXXXXXXX",
      body: "This is updated content"
  })
  ```

- [ ] **Verify in Gmail** that draft content was updated

- [ ] **Test edge cases:**
  - [ ] Update only subject (body preserved)
  - [ ] Update only body (subject preserved)
  - [ ] Update with invalid draft ID (should error gracefully)
  - [ ] Update draft that's a reply (thread context preserved)

### Testing Checklist

- [ ] Draft updates successfully with new body
- [ ] Draft preserves existing subject when not provided
- [ ] Draft preserves existing recipients when not provided
- [ ] Error message is clear for invalid draft ID
- [ ] Thread context preserved for reply drafts
- [ ] No crashes or unhandled exceptions

## Phase 3: Commit and Push

### Commit Changes

- [ ] **Stage changes**
  ```bash
  git add src/index.ts
  ```

- [ ] **Review changes before committing**
  ```bash
  git diff --staged
  ```

- [ ] **Commit with descriptive message**
  ```bash
  git commit -m "feat: add update_draft tool for modifying existing Gmail drafts

  - Add UpdateDraftSchema with optional fields for partial updates
  - Implement handleUpdateDraft function that merges new values with existing
  - Preserve thread context and reply-to headers
  - Add tool registration and request handler

  Resolves the limitation where drafts with agent instructions
  could not be programmatically updated."
  ```

### Push to Fork

- [ ] **Push feature branch to your fork**
  ```bash
  git push origin feature/update-draft
  ```

- [ ] **Verify push succeeded**
  ```bash
  git log origin/feature/update-draft --oneline -1
  ```

## Phase 4: Merge to Your Fork's Main (PREFERRED - Option A)

**This is the recommended approach** - merge and test in your fork before submitting to upstream.

- [ ] **Navigate to** https://github.com/git-bafshar/Gmail-MCP-Server

- [ ] **Click** "Compare & pull request" button

- [ ] **Set PR parameters:**
  - Base repository: `git-bafshar/Gmail-MCP-Server`
  - Base branch: `main`
  - Compare branch: `feature/update-draft`

- [ ] **Add PR title:** "feat: add update_draft tool"

- [ ] **Add PR description** (use template from `pr-workflow-setup.md`)

- [ ] **Create pull request**

- [ ] **Review and merge** to your main branch

- [ ] **Sync local main**
  ```bash
  git checkout main
  git pull origin main
  ```

**OR skip PR and merge directly:**

- [ ] **Merge feature branch to main**
  ```bash
  git checkout main
  git merge feature/update-draft
  git push origin main
  ```

### Production Testing Period (1-2 Weeks)

- [ ] **Update local MCP installation with your fork's main**
  ```bash
  cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
  git checkout main
  git pull origin main
  npm install
  npm run build
  ```

- [ ] **Restart Claude Code** to load updated MCP server

- [ ] **Use update_draft in real workflow** over the next 1-2 weeks:
  - [ ] Update drafts with agent instructions multiple times
  - [ ] Test various email scenarios (replies, new emails, long content)
  - [ ] Monitor for any errors or edge cases
  - [ ] Verify performance is acceptable

- [ ] **Document any issues found** and fix them on feature branch if needed

- [ ] **Once stable and confident**, proceed to Phase 5

**Do not proceed to Phase 5 until you've tested thoroughly in production!**

## Phase 5: Pull Request to Upstream (After Production Validation)

### Pre-PR Verification

- [ ] **Code Quality:**
  - [ ] Follows existing code style
  - [ ] Uses consistent error handling patterns
  - [ ] No debugging code or console.logs
  - [ ] No commented-out code

- [ ] **Documentation:**
  - [ ] Function behavior is clear
  - [ ] Limitations documented (attachment loss)
  - [ ] Example usage available

- [ ] **Testing:**
  - [ ] All test cases passed
  - [ ] Tested with real Gmail account
  - [ ] Edge cases covered

- [ ] **Git:**
  - [ ] Branch is up-to-date with upstream/main
  - [ ] Commits are clean and descriptive
  - [ ] No merge conflicts

### Create Upstream PR

- [ ] **Navigate to** https://github.com/GongRzhe/Gmail-MCP-Server

- [ ] **Click** "Fork" dropdown → Navigate to your fork

- [ ] **Click** "Contribute" → "Open pull request"

- [ ] **Set PR parameters:**
  - Base repository: `GongRzhe/Gmail-MCP-Server`
  - Base branch: `main`
  - Head repository: `git-bafshar/Gmail-MCP-Server`
  - Compare branch: `main` (or `feature/update-draft`)

- [ ] **Add PR title:** "feat: add update_draft tool for programmatic draft updates"

- [ ] **Add comprehensive PR description** using template from `pr-workflow-setup.md`:
  - Description of feature
  - Motivation and use case
  - Changes made
  - Limitations (attachment handling)
  - Testing performed
  - Compatibility notes

- [ ] **Create pull request**

### Respond to Feedback

- [ ] **Monitor PR for maintainer comments**

- [ ] **If changes requested:**
  ```bash
  git checkout feature/update-draft
  # Make requested changes
  git add .
  git commit -m "fix: address PR feedback - [description]"
  git push origin feature/update-draft
  ```

- [ ] **PR will auto-update** with new commits

- [ ] **Respond to comments** professionally and promptly

## Phase 6: Post-Merge Cleanup

### After Upstream Merges

- [ ] **Sync your fork with upstream**
  ```bash
  git checkout main
  git fetch upstream
  git merge upstream/main
  git push origin main
  ```

- [ ] **Delete feature branch locally**
  ```bash
  git branch -d feature/update-draft
  ```

- [ ] **Delete feature branch on GitHub**
  ```bash
  git push origin --delete feature/update-draft
  ```

- [ ] **Update local MCP installation**
  ```bash
  cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
  git pull origin main
  npm install
  npm run build
  ```

- [ ] **Restart Claude Code** to use updated MCP server

### Update Workflow Documentation

- [ ] **Update** `/Users/ben-afshar/Tools/workflows/gmail-workflow/CLAUDE.md`:
  - Add `update_draft` to available tools list
  - Update "Emails in Draft" workflow to use `update_draft`
  - Document draft ID vs message ID distinction

## Ongoing Maintenance

- [ ] **Set reminder** to sync fork with upstream regularly (weekly/monthly)
  ```bash
  git checkout main
  git fetch upstream
  git merge upstream/main
  git push origin main
  ```

## Troubleshooting

### Build Errors

**Error:** `Cannot find name 'UpdateDraftSchema'`
- **Fix:** Ensure schema is defined before it's used in `zodToJsonSchema()`

**Error:** `'gmail' is not defined`
- **Fix:** Make sure you're adding code inside the proper scope where `gmail` is available (after `const gmail = google.gmail(...)`)

### Runtime Errors

**Error:** `Draft not found`
- **Fix:** Verify you're using draft ID (format: `r-xxx`) not message ID (format: `19bxxx`)

**Error:** `Insufficient permissions`
- **Fix:** Check OAuth scopes include `gmail.modify` or `gmail.compose`

### Git Issues

**Error:** `merge conflict`
- **Fix:**
  ```bash
  git fetch upstream
  git rebase upstream/main
  # Resolve conflicts
  git add .
  git rebase --continue
  ```

## Questions or Issues?

- Review `mcp-server-analysis.md` for technical details
- Review `pr-workflow-setup.md` for git workflow details
- Check upstream repo for contributing guidelines
- Ask maintainer questions in PR comments

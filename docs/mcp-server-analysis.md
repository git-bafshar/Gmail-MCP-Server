# Gmail MCP Server Analysis & Update Draft Implementation Plan

## Server Structure Overview

**Location:** `/Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server/`

**Project Type:** TypeScript ES2020 module (forked from gongrzhe/server-gmail-autoauth-mcp)

**Key Files:**
```
Gmail-MCP-Server/
├── src/
│   ├── index.ts              # Main server file (all tools defined here, ~1500 lines)
│   ├── label-manager.ts      # Label management utilities
│   ├── filter-manager.ts     # Filter management utilities
│   └── utl.ts               # Email creation utilities (used by send/draft)
├── dist/                     # Compiled JavaScript output
├── package.json             # Dependencies & build scripts
└── tsconfig.json            # TypeScript configuration
```

**Architecture Pattern:**
- Single monolithic `index.ts` file contains all tool implementations
- Tools are registered in a `ListToolsRequestSchema` handler
- Tool execution happens in a `CallToolRequestSchema` handler with a switch statement
- Uses Zod for schema validation with `zod-to-json-schema` for MCP schemas

## Current Draft Email Implementation

**Tool Registration** (line ~349):
```typescript
{
    name: "draft_email",
    description: "Draft a new email",
    inputSchema: zodToJsonSchema(SendEmailSchema),
}
```

**Schema** (line ~194):
```typescript
const SendEmailSchema = z.object({
    to: z.array(z.string()).describe("List of recipient email addresses"),
    subject: z.string().describe("Email subject"),
    body: z.string().describe("Email body content"),
    htmlBody: z.string().optional().describe("HTML version of the email body"),
    mimeType: z.enum(['text/plain', 'text/html', 'multipart/alternative']).optional(),
    cc: z.array(z.string()).optional(),
    bcc: z.array(z.string()).optional(),
    threadId: z.string().optional().describe("Thread ID to reply to"),
    inReplyTo: z.string().optional().describe("Message ID being replied to"),
    attachments: z.array(z.string()).optional().describe("List of file paths"),
});
```

**Handler Logic** (line ~444-550):
```typescript
async function handleEmailAction(action: "send" | "draft", validatedArgs: any) {
    // Two paths: with attachments (uses nodemailer) or without (uses simple message)

    if (validatedArgs.attachments && validatedArgs.attachments.length > 0) {
        message = await createEmailWithNodemailer(validatedArgs);
    } else {
        message = createEmailMessage(validatedArgs);
    }

    const encodedMessage = Buffer.from(message).toString('base64')
        .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');

    if (action === "draft") {
        const response = await gmail.users.drafts.create({
            userId: 'me',
            requestBody: {
                message: {
                    raw: encodedMessage,
                    ...(validatedArgs.threadId && { threadId: validatedArgs.threadId })
                },
            },
        });
        return {
            content: [{
                type: "text",
                text: `Email draft created successfully with ID: ${response.data.id}`
            }]
        };
    }
}
```

**Tool Invocation** (line ~599):
```typescript
case "send_email":
case "draft_email": {
    const validatedArgs = SendEmailSchema.parse(args);
    const action = name === "send_email" ? "send" : "draft";
    return await handleEmailAction(action, validatedArgs);
}
```

## Implementation Plan: Add update_draft Tool

### Step 1: Define Schema

Add after `SendEmailSchema` (around line 205):

```typescript
const UpdateDraftSchema = z.object({
    draftId: z.string().describe("ID of the draft to update"),
    to: z.array(z.string()).optional().describe("List of recipient email addresses"),
    subject: z.string().optional().describe("Email subject"),
    body: z.string().optional().describe("Email body content"),
    htmlBody: z.string().optional().describe("HTML version of the email body"),
    mimeType: z.enum(['text/plain', 'text/html', 'multipart/alternative']).optional(),
    cc: z.array(z.string()).optional().describe("List of CC recipients"),
    bcc: z.array(z.string()).optional().describe("List of BCC recipients"),
});
```

**Note:** All fields except `draftId` are optional to support partial updates.

### Step 2: Register Tool

Add to the tools array in `ListToolsRequestSchema` handler (around line 352):

```typescript
{
    name: "update_draft",
    description: "Updates the content of an existing Gmail draft while preserving attachments",
    inputSchema: zodToJsonSchema(UpdateDraftSchema),
},
```

### Step 3: Add Handler Function

Add new function before the `handleEmailAction` function (around line 440):

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

### Step 4: Add Case Statement

Add to the switch statement in `CallToolRequestSchema` handler (around line 605):

```typescript
case "update_draft": {
    const validatedArgs = UpdateDraftSchema.parse(args);
    return await handleUpdateDraft(validatedArgs);
}
```

### Step 5: Build and Deploy

```bash
cd /Users/ben-afshar/.gmail-mcp/Gmail-MCP-Server
npm run build
```

Then restart Claude Code to reload the MCP server.

## Important Notes

### OAuth Scopes
Verify that your OAuth configuration (`gcp-oauth.keys.json`) includes the necessary Gmail scopes:
- `https://www.googleapis.com/auth/gmail.modify` (for draft updates)
- `https://www.googleapis.com/auth/gmail.compose` (for creating/sending emails)

Current scopes can be found in the OAuth client credentials.

### Attachment Handling
**Limitation:** The Gmail API's `drafts.update` endpoint with a raw message will **replace the entire draft**, which means:
- **Attachments will be lost** if they're not re-encoded in the new message
- The current implementation above does NOT preserve attachments

**Solutions:**
1. **Accept the limitation** (recommended for v1): Document that `update_draft` loses attachments. Since "Draft with Agent Instructions" workflow typically starts without attachments, this isn't a blocker.

2. **Advanced: Preserve attachments**: Would require:
   - Extracting attachment data from existing draft
   - Re-encoding them in the updated message using nodemailer
   - Significantly more complex implementation

For your use case (updating draft body from agent instructions), option 1 is sufficient.

### Draft ID Discovery
When reading a draft via `read_email`, the response doesn't directly include the draft ID. To get it:

```typescript
// Search for drafts first to get draft IDs
const draftsResponse = await gmail.users.drafts.list({
    userId: 'me',
    maxResults: 10
});

// Each draft has: { id: "r-...", message: { id: "19bb..." } }
// Use draft.id for update_draft, not draft.message.id
```

**Update workflow:**
1. Search: `in:drafts` → returns message IDs
2. Read each message to check for agent instructions
3. Need to map message ID back to draft ID for update
   - **Problem:** `read_email` returns message, not draft object
   - **Solution:** Use `gmail.users.drafts.list()` first, which returns both IDs

### Alternative Workflow Without update_draft

If implementation is complex, the current approach works:
1. Agent reads draft with instructions
2. Agent drafts response in local file
3. User manually copies content into Gmail draft
4. User sends

The automation gain from `update_draft` is moderate but improves UX.

## Testing Checklist

Once implemented, test:
- [ ] Update draft body only (preserve subject/recipients)
- [ ] Update subject only (preserve body/recipients)
- [ ] Update all fields simultaneously
- [ ] Invalid draft ID returns clear error
- [ ] Draft with no recipients/subject (edge case)
- [ ] Verify thread context preserved for reply drafts

## References

- Gmail API - Drafts.update: https://developers.google.com/gmail/api/reference/rest/v1/users.drafts/update
- Current MCP SDK version: 1.24.0
- Project repo: https://github.com/gongrzhe/server-gmail-autoauth-mcp

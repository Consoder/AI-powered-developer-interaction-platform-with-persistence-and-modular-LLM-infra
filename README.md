# AI Chatbot — Full-Stack Production Application
### Built with Next.js 15 · Vercel AI SDK 5 · xAI Grok · PostgreSQL · Drizzle ORM

---

## What This Project Is

A **production-grade, full-stack AI chat application** — not a tutorial clone, but an implementation of real engineering patterns used at scale. It features streaming AI responses, a multi-model inference layer, real-time artifact generation (text, code, images, spreadsheets), resumable streams backed by Redis, a credential + guest auth system, message voting, document versioning, and a full E2E test suite with Playwright.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router, Server Actions, Route Handlers) |
| AI SDK | Vercel AI SDK v5 (beta) — `streamText`, `streamObject`, `generateImage` |
| LLM Provider | xAI Grok-2 Vision (chat), Grok-3-mini (reasoning), Grok-2-Image (image gen) |
| Database | PostgreSQL via Drizzle ORM (type-safe, migration-tracked) |
| Auth | NextAuth v5 — credential login + automatic guest sessions |
| Stream Resilience | Redis-backed `resumable-stream` (reconnect without re-generating) |
| File Storage | Vercel Blob |
| UI | React 19 RC, Tailwind CSS, Radix UI, Framer Motion |
| Rich Text | ProseMirror (collaborative-style text editor) |
| Code Editor | CodeMirror 6 (Python + JS highlighting) |
| Spreadsheet | react-data-grid + PapaParse (CSV) |
| Validation | Zod (request body + AI structured outputs) |
| Testing | Playwright E2E + route-level unit tests |
| Observability | OpenTelemetry + Vercel OTel integration |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        BROWSER (Client)                          │
│                                                                  │
│  ┌──────────────┐    ┌──────────────────┐   ┌───────────────┐  │
│  │  Chat Panel  │    │  Artifact Panel  │   │  App Sidebar  │  │
│  │              │    │  ┌────────────┐  │   │               │  │
│  │  useChat()   │    │  │ Text/Code/ │  │   │ Chat History  │  │
│  │  AI SDK hook │    │  │ Image/Sheet│  │   │ (paginated)   │  │
│  │              │    │  └────────────┘  │   │               │  │
│  │  SSE Stream  │    │  DataStream      │   │  Auth state   │  │
│  └──────┬───────┘    └───────┬──────────┘   └───────────────┘  │
│         │                   │                                    │
└─────────┼───────────────────┼────────────────────────────────────┘
          │ HTTP POST          │ SSE deltas
          ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      NEXT.JS SERVER                              │
│                                                                  │
│  middleware.ts ──► Auth check (JWT token)                        │
│  /api/auth     ──► NextAuth credential + guest providers         │
│                                                                  │
│  POST /api/chat ─────────────────────────────────────────────►  │
│    │  1. Validate request (Zod schema)                           │
│    │  2. Auth session check                                       │
│    │  3. Rate limit check (message count per 24h window)         │
│    │  4. Get or create chat row in DB                            │
│    │  5. Save user message to DB                                 │
│    │  6. Create stream ID → save to DB                          │
│    │  7. streamText() with 4 AI tools                           │
│    │  8. Pipe through ResumableStream → SSE response            │
│    └─────────────────────────────────────────────────────────►  │
│                                                                  │
│  GET /api/chat/[id]/stream ──► Resume interrupted stream        │
│  GET /api/history          ──► Paginated chat list              │
│  GET/PATCH /api/vote       ──► Message up/down voting           │
│  GET/POST /api/document    ──► Document CRUD                    │
│  GET /api/suggestions      ──► AI-generated edit suggestions    │
│  POST /api/files/upload    ──► Blob storage upload              │
│                                                                  │
└──────────────────────┬──────────────────────────────────────────┘
                       │
          ┌────────────┼─────────────────┐
          ▼            ▼                 ▼
    ┌──────────┐  ┌─────────┐    ┌─────────────┐
    │PostgreSQL│  │  Redis  │    │  xAI Grok   │
    │          │  │         │    │  (4 models) │
    │ Drizzle  │  │Resumable│    │             │
    │ ORM      │  │Streams  │    │ grok-2-vision│
    └──────────┘  └─────────┘    │ grok-3-mini │
                                 │ grok-2-1212  │
                                 │ grok-2-image │
                                 └─────────────┘
```

---

## Database ER Diagram

```
┌──────────────┐
│     User     │
├──────────────┤
│ id (PK, UUID)│◄────────────────────────────────────┐
│ email        │                                      │
│ password     │                                      │
└──────┬───────┘                                      │
       │ 1:N                                          │ 1:N
       ▼                                              │
┌──────────────┐         ┌───────────────────┐   ┌──────────────┐
│     Chat     │         │    Message_v2     │   │   Document   │
├──────────────┤  1:N    ├───────────────────┤   ├──────────────┤
│ id (PK, UUID)│◄───────►│ id (PK, UUID)     │   │ id (UUID)    │
│ userId (FK)  │         │ chatId (FK)       │   │ createdAt    │◄─ composite PK
│ title        │         │ role              │   │ title        │
│ createdAt    │         │ parts (JSON)      │   │ content      │
│ visibility   │         │ attachments (JSON)│   │ kind (enum)  │
└──────┬───────┘         │ createdAt         │   │ userId (FK)  │
       │                 └─────────┬─────────┘   └──────┬───────┘
       │ 1:N                       │ 1:1                 │ 1:N
       ▼                           ▼                     ▼
┌──────────────┐         ┌───────────────────┐   ┌──────────────────┐
│    Stream    │         │     Vote_v2       │   │   Suggestion     │
├──────────────┤         ├───────────────────┤   ├──────────────────┤
│ id (PK, UUID)│         │ chatId (FK)       │   │ id (PK, UUID)    │
│ chatId (FK)  │         │ messageId (FK)    │   │ documentId (FK)  │
│ createdAt    │         │ isUpvoted (bool)  │   │ documentCreatedAt│
└──────────────┘         │ PK: chatId+msgId  │   │ originalText     │
                         └───────────────────┘   │ suggestedText    │
                                                  │ description      │
                                                  │ isResolved       │
                                                  │ userId (FK)      │
                                                  └──────────────────┘

Note: Message and Vote have deprecated v1 tables (kept for migration history).
Document uses a composite PK (id + createdAt) enabling versioning — 
multiple versions of the same document ID can coexist, ordered by timestamp.
```

---

## Request Lifecycle — Full Flow

```
User types message → clicks Send
         │
         ▼
MultimodalInput.submitForm()
  ├── Attaches any uploaded files (Blob URLs)
  ├── Calls sendMessage({ role: 'user', parts: [...] })
  └── Clears input, updates URL to /chat/{id}
         │
         ▼
useChat (AI SDK) → POST /api/chat
  Body: { id, message, selectedChatModel, selectedVisibilityType }
         │
         ▼ Server
  1. Zod validation (postRequestBodySchema)
  2. auth() → check JWT session
  3. getMessageCountByUserId() → rate limit (20/day guest, 100/day regular)
  4. getChatById() → if null, generateTitleFromUserMessage() → saveChat()
  5. getMessagesByChatId() → hydrate full conversation history
  6. geolocation(request) → extract lat/lon/city/country for system prompt
  7. saveMessages([userMessage]) → persist user turn to DB
  8. createStreamId() → save new stream record to DB
         │
         ▼
  createUIMessageStream({
    execute: ({ writer }) => {
      streamText({
        model: myProvider.languageModel(selectedChatModel),
        system: systemPrompt(model, geoHints),
        messages: convertToModelMessages(uiMessages),
        stopWhen: stepCountIs(5),         ← max 5 agentic steps
        tools: {
          getWeather,          ← OpenWeatherMap via tool call
          createDocument,      ← spawns artifact stream
          updateDocument,      ← patches existing artifact
          requestSuggestions,  ← AI reviews document, streams suggestions
        },
        experimental_transform: smoothStream({ chunking: 'word' })
      })
    },
    onFinish: async ({ messages }) => saveMessages(assistantMessages)
  })
         │
         ▼
  ResumableStream (Redis-backed)
  └── If Redis available: stream keyed by streamId (survives reconnect)
  └── If no Redis: direct SSE pipe
         │
         ▼ Client receives SSE deltas
  DataStreamHandler processes:
    'data-kind'      → set artifact type
    'data-id'        → set document ID
    'data-title'     → set artifact title
    'data-clear'     → clear artifact content
    'data-textDelta' → append to text artifact
    'data-codeDelta' → append to code artifact
    'data-imageDelta'→ set base64 image
    'data-sheetDelta'→ update CSV content
    'data-suggestion'→ add inline suggestion markers
    'data-finish'    → mark artifact as idle
         │
         ▼
  UI updates in real time:
    Messages panel ← AI text renders with Markdown
    Artifact panel ← content streams in live
    Sidebar        ← chat title appears (SWR revalidates)
```

---

## AI Tool System — How Artifacts Work

The AI has 4 registered tools it can autonomously invoke during a response:

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI Tool Architecture                          │
│                                                                  │
│  Tool: createDocument                                            │
│  ├── AI decides: "user wants a document"                        │
│  ├── Input: { title: string, kind: 'text'|'code'|'image'|'sheet'}│
│  ├── Streams: data-kind, data-id, data-title, data-clear        │
│  ├── Dispatches to DocumentHandler by kind:                     │
│  │   ├── text  → streamText(artifact-model) → data-textDelta    │
│  │   ├── code  → streamObject(schema:{code}) → data-codeDelta   │
│  │   ├── image → generateImage(grok-2-image) → data-imageDelta  │
│  │   └── sheet → streamObject(schema:{csv}) → data-sheetDelta   │
│  └── Saves final content to Document table                      │
│                                                                  │
│  Tool: updateDocument                                            │
│  ├── Input: { id: string, description: string }                 │
│  ├── Fetches existing document from DB                          │
│  ├── Re-runs same handler with updateDocumentPrompt()           │
│  │   (passes current content for targeted edits)                │
│  └── Saves new version (new createdAt → version history)        │
│                                                                  │
│  Tool: requestSuggestions                                        │
│  ├── Input: { documentId: string }                              │
│  ├── streamObject() → array of {original, suggested, desc}     │
│  ├── Streams each suggestion via data-suggestion               │
│  └── Saves all suggestions to Suggestion table                  │
│                                                                  │
│  Tool: getWeather                                                │
│  └── Fetches current weather for a city                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Authentication Architecture

```
Request hits middleware.ts
        │
        ▼
getToken(request) → check JWT in cookie
        │
   ┌────┴────┐
   │ No token │──► redirect to /api/auth/guest
   └─────────┘          │
                        ▼
                 createGuestUser()
                 ├── email: "guest-{timestamp}"
                 ├── password: hash(UUID)
                 └── type: 'guest' (20 msg/day limit)
                        │
                        ▼
                 Set-Cookie: session JWT
                 Redirect to original URL

   ┌──────────┐
   │ Has token │──► check if /login or /register
   └──────────┘    ├── if regular user → redirect to /
                   └── else → NextResponse.next()

Auth Providers (NextAuth v5):
  1. Credentials (email+password)
     ├── getUser(email) → bcrypt.compare()
     ├── timing-safe: always runs compare (even for missing users)
     └── returns { ...user, type: 'regular' }

  2. Credentials (guest)
     └── createGuestUser() → returns { ...user, type: 'guest' }

JWT Callbacks:
  jwt()     → embeds token.id + token.type
  session() → exposes session.user.id + session.user.type

Entitlements by UserType:
  guest:   20 messages/day
  regular: 100 messages/day
```

---

## Resumable Stream Architecture

A key production feature: if the user's browser disconnects mid-generation, the stream resumes from where it left off.

```
POST /api/chat
  ├── Generate UUID streamId
  ├── Save streamId → DB (Stream table, linked to chatId)
  ├── streamContext.resumableStream(streamId, streamFactory)
  │       └── If Redis: stores stream in Redis keyed by streamId
  └── Return SSE Response

Browser disconnects mid-stream...

Browser reconnects → GET /api/chat/[id]/stream
  ├── Auth check
  ├── getStreamIdsByChatId() → get latest streamId
  ├── streamContext.resumableStream(streamId, emptyFactory)
  │       └── Redis has the buffered stream → resume from offset
  └── If stream expired (>15s since last message):
        └── Reconstruct from DB: fetch mostRecentMessage → stream it
```

---

## Model Configuration

```
myProvider (customProvider)
├── 'chat-model'          → xai('grok-2-vision-1212')
│     Tools: all 4 enabled, smoothStream word-chunking
│     Use: standard chat + artifact creation
│
├── 'chat-model-reasoning'→ wrapLanguageModel(xai('grok-3-mini-beta'))
│     Middleware: extractReasoningMiddleware({ tagName: 'think' })
│     Tools: NONE (reasoning model runs uninterrupted)
│     Use: complex problems, step-by-step reasoning
│
├── 'title-model'         → xai('grok-2-1212')
│     Use: auto-generate chat title from first message
│
└── 'artifact-model'      → xai('grok-2-1212')
      Use: all artifact generation/update (text, code, sheet)
      Image: xai.imageModel('grok-2-image') → generateImage()
```

---

## Migration History (7 Iterations)

The project shows a real evolution of data design decisions:

| Migration | Change | Why |
|---|---|---|
| 0000 | `Chat` + `User` tables (messages stored as JSON blob in Chat) | Simple start |
| 0001 | Extract `Message` table (normalized) | Separation of concerns |
| 0002 | Add `Document` table (versioned by composite PK) | Artifact persistence |
| 0003 | Add `Suggestion` table | AI review feature |
| 0004 | Add `visibility` to Chat, `Vote` table | Public sharing + feedback |
| 0005 | `Message_v2` + `Vote_v2` (parts/attachments split) | Multimodal support |
| 0006 | `Stream` table | Resumable stream IDs |

---

## Code Quality Highlights

### Type Safety End-to-End
- Drizzle ORM uses `InferSelectModel<>` — DB types flow from schema to queries to API responses automatically
- Zod validates every API request body before any business logic runs
- NextAuth session extended with custom types (`UserType`, `user.id`) via declaration merging

### Error Handling Pattern
```typescript
// Custom error class with typed error codes
class ChatSDKError extends Error {
  constructor(errorCode: `${ErrorType}:${Surface}`, cause?: string)
  toResponse(): Response   // → structured JSON with correct HTTP status
}

// Surface-aware visibility:
// 'database' errors → logged server-side, generic message to client
// 'chat' errors     → full error code + message returned to client
```

### Security
- Timing-safe password comparison (bcrypt always runs even for missing users — prevents user enumeration)
- JWT over HTTP-only cookies via NextAuth
- Ownership checks on every chat/document mutation (`chat.userId !== session.user.id`)
- Rate limiting enforced server-side (not client-trusting)
- File upload type validation (only `image/jpeg`, `image/png` accepted)

### Performance
- `smoothStream({ chunking: 'word' })` — prevents single-character flicker in streamed output
- `experimental_throttle: 100` on `useChat` — batches re-renders to 10fps max
- `memo()` on `MultimodalInput` and `Artifact` — prevents re-renders during streaming
- `fast-deep-equal` for shallow message comparison in memo guards
- SWR for chat history with `unstable_serialize` for infinite pagination
- `server-only` import guard on `queries.ts` — prevents DB code leaking to client bundle

---

## Component Tree

```
app/layout.tsx
└── app/(chat)/layout.tsx
    ├── AppSidebar
    │   ├── SidebarHistory (SWR paginated)
    │   │   └── SidebarHistoryItem
    │   └── SidebarUserNav
    │
    └── app/(chat)/chat/[id]/page.tsx
        └── Chat (main orchestrator)
            ├── ChatHeader
            │   ├── ModelSelector
            │   └── VisibilitySelector
            │
            ├── Messages
            │   └── Message (per message)
            │       ├── Markdown renderer
            │       ├── MessageReasoning (for reasoning model)
            │       ├── MessageActions (vote up/down, edit, copy)
            │       └── DocumentPreview (inline artifact link)
            │
            ├── MultimodalInput
            │   ├── Textarea (auto-resizing)
            │   ├── SuggestedActions (first message only)
            │   └── PreviewAttachment (image thumbnails)
            │
            └── Artifact (right panel, animated slide-in)
                ├── ArtifactMessages (chat within artifact)
                ├── TextEditor (ProseMirror)
                ├── CodeEditor (CodeMirror 6)
                ├── ImageEditor
                ├── SheetEditor (react-data-grid)
                ├── Toolbar (AI actions: suggest, update)
                ├── VersionFooter (document versioning nav)
                └── ArtifactActions (copy, download)
```

---

## What I Built and Key Decisions

### 1. Artifact Handler Registry Pattern
Rather than a monolithic switch statement, artifact handlers are registered in an array (`documentHandlersByArtifactKind`). Adding a new artifact type means creating a `server.ts` + `client.tsx` pair and registering it — zero changes to the core routing logic. This is the **Open/Closed Principle** applied to AI tool dispatch.

### 2. Document Versioning via Composite Primary Key
`Document` uses `(id, createdAt)` as a composite PK. Every time a document is updated, a new row is inserted with the same `id` but a new `createdAt`. The `getDocumentsById` query returns all versions ordered by time — enabling a full version history without a separate versions table.

### 3. Stream ID as a Resumption Primitive
Each chat request generates a `streamId` (UUID) stored in the `Stream` table. The resumable-stream library uses this as a Redis key to buffer the SSE output. If a user's network drops, the client calls `GET /api/chat/[id]/stream`, looks up the most recent `streamId`, and Redis delivers the buffered content without re-hitting the LLM. This is a real production reliability pattern.

### 4. Guest-First UX
The middleware auto-provisions a guest account for any unauthenticated visitor — no friction before they can use the product. Guests get a lower rate limit (20 messages/day vs 100). This pattern is used by products like v0 and ChatGPT.

### 5. Reasoning Model Isolation
The `chat-model-reasoning` model has `experimental_activeTools: []` — tools are completely disabled. This is intentional: reasoning models think through their chain of thought inside `<think>` tags, and tool interruptions break the reasoning flow. The `extractReasoningMiddleware` strips the think blocks before streaming to the client and exposes them separately for the `MessageReasoning` component.

---

## Running Locally

```bash
# Prerequisites: Node.js 18+, pnpm, PostgreSQL, Redis (optional)

git clone <repo>
cd nextjs-ai-chatbot-main
pnpm install

# Set up environment
cp .env.example .env.local
# Fill in: POSTGRES_URL, XAI_API_KEY, AUTH_SECRET, BLOB_READ_WRITE_TOKEN
# Optional: REDIS_URL (for resumable streams)

# Run migrations
pnpm db:migrate

# Start dev server
pnpm dev
```

```bash
# Run E2E tests
pnpm test

# DB studio (visual DB browser)
pnpm db:studio
```

---

## Project Stats

| Metric | Count |
|---|---|
| TypeScript files | 110+ |
| API routes | 8 |
| DB tables | 8 (including v1 deprecated) |
| DB migrations | 7 |
| AI tools | 4 |
| Artifact types | 4 (text, code, image, sheet) |
| Custom React hooks | 6 |
| E2E test files | 4 |
| Route test files | 2 |

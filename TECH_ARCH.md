# wrrk.ai Technical Architecture (MVP)

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                  │
│   Browser (React) ←──── WebSocket ────→ Socket.io Server        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      VERCEL PLATFORM                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  Next.js App    │  │  API Routes     │  │  Custom Server  │  │
│  │  (App Router)   │  │  /app/api/*     │  │  (Socket.io)    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   Supabase    │    │    Upstash    │    │  AWS Bedrock  │
│  PostgreSQL   │    │     Redis     │    │   (Claude)    │
└───────────────┘    └───────────────┘    └───────────────┘
```

### Request Flows
- **HTTP**: Client → Vercel Edge → API Routes → Prisma → PostgreSQL
- **Real-time**: Client ↔ Socket.io ↔ Redis Pub/Sub ↔ Other Clients
- **AI**: API Route → AWS Bedrock → Claude 3 Sonnet → Response

---

## Infrastructure Stack

| Component | Service | Purpose | Config Location |
|-----------|---------|---------|-----------------|
| Hosting | Vercel | App + API + Serverless | `vercel.json` |
| Database | Supabase PostgreSQL | Primary data store | `DATABASE_URL` env |
| Cache/Pub-Sub | Upstash Redis | Socket.io scaling | `REDIS_URL` env |
| Email | SendGrid | Inbound/outbound email | `SENDGRID_API_KEY` env |
| AI | AWS Bedrock | Copilot (Claude 3 Sonnet) | `AWS_*` env vars |
| Auth | NextAuth + Cognito | User authentication | `lib/auth/config.ts` |
| File Storage | Vercel Blob | Attachments | `BLOB_*` env vars |

---

## Monorepo Structure

```
wrrk.ai/
├── apps/
│   └── web/                    # Next.js 14 application
│       ├── app/                # App Router pages & API
│       │   ├── api/            # REST API endpoints
│       │   ├── (auth)/         # Auth pages (login, register)
│       │   └── (dashboard)/    # Protected dashboard pages
│       ├── components/         # React components
│       ├── lib/                # Core business logic
│       │   ├── ai/             # AWS Bedrock integration
│       │   ├── auth/           # NextAuth configuration
│       │   ├── rbac/           # Role-based access control
│       │   ├── socket/         # Socket.io server & hooks
│       │   └── validations/    # Zod schemas
│       └── server.ts           # Custom server (Socket.io)
├── packages/
│   ├── database/               # @wrrk/database
│   │   └── prisma/
│   │       └── schema.prisma   # Database schema
│   └── config/                 # @wrrk/config (shared TS configs)
└── package.json                # Turborepo workspace root
```

---

## Database Schema (Key Models)

### Multi-Tenant Design
All data is scoped to `organizationId`. Every query MUST filter by organization.

```prisma
// Core Entities
Organization    # Tenant root - billing, settings
User            # Members with roles (OWNER, MANAGER, AGENT) + createdById hierarchy
Team            # Optional groups within organization (alongside hierarchy)
TeamMember      # User-Team assignments

// Customer Support
Customer        # External customers
Ticket          # Support tickets (status, priority, channel)
Message         # Ticket messages (from customer or agent)

// Internal Collaboration
Conversation    # Internal chat threads
ConversationParticipant
FeedPost        # Internal feed posts
FeedComment, FeedReaction

// Integrations
Integration     # Email channel configs (type: EMAIL)

// Tracking
AuditLog        # All state changes
CopilotInteraction  # AI usage tracking
```

### Key Enums
- **Role**: `OWNER | MANAGER | AGENT` (hierarchical - see User Hierarchy section)
- **TicketStatus**: `OPEN | IN_PROGRESS | WAITING | RESOLVED | CLOSED`
- **Priority**: `LOW | MEDIUM | HIGH | URGENT`
- **Channel**: `EMAIL | CHAT | PHONE | SOCIAL | PORTAL`

### User Hierarchy Schema
```prisma
model User {
  // ... existing fields ...

  // Hierarchy tracking - who created this user
  createdById   String?
  createdBy     User?    @relation("CreatedUsers", fields: [createdById], references: [id])
  createdUsers  User[]   @relation("CreatedUsers")

  role          Role     @default(AGENT)
}

enum Role {
  OWNER     // Top-level, can promote others to Owner
  MANAGER   // Mid-level, manages own subtree
  AGENT     // Handles tickets/chats
}
```

---

## API Design Pattern

### Standard API Route Structure
```typescript
// app/api/[resource]/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { z } from 'zod'
import { prisma } from '@wrrk/database'
import { hasPermission } from '@/lib/rbac/permissions'
import { authOptions } from '@/lib/auth/config'

export async function GET(req: NextRequest) {
  // 1. Auth check
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // 2. RBAC check
  if (!hasPermission(session.user.role, 'resource:read')) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }

  // 3. Organization-scoped query
  const data = await prisma.resource.findMany({
    where: { organizationId: session.user.organizationId }
  })

  return NextResponse.json({ data })
}

export async function POST(req: NextRequest) {
  // 1. Auth + RBAC
  // 2. Validate with Zod
  const body = await req.json()
  const validated = schema.parse(body)

  // 3. Create with audit log
  const result = await prisma.$transaction([
    prisma.resource.create({ data: { ...validated, organizationId } }),
    prisma.auditLog.create({ data: { action: 'CREATE', ... } })
  ])

  return NextResponse.json({ data: result[0] }, { status: 201 })
}
```

### RBAC Permissions (Hierarchical)
Defined in `lib/rbac/permissions.ts`:

| Permission | OWNER | MANAGER | AGENT |
|------------|-------|---------|-------|
| Promote to Owner | ✅ | ❌ | ❌ |
| Create Manager | ✅ | ✅ | ❌ |
| Create Agent | ✅ | ✅ | ❌ |
| See all users | ✅ | ❌ (own subtree) | ❌ (self only) |
| See all tickets | ✅ | ❌ (own subtree) | ❌ (assigned only) |
| System Settings | ✅ | ❌ | ❌ |
| Analytics | ✅ (all) | ✅ (own agents) | ❌ |

**Note**: Visibility is hierarchy-based via `createdById`. See "User Hierarchy & Visibility" section below.

---

## User Hierarchy & Visibility

### Hierarchy Structure
```
OWNER(s) ─────────────────────────────┐
├── MANAGER A                         │
│   ├── Agent 1                       │  Organization
│   └── Agent 2                       │
├── MANAGER B                         │
│   ├── MANAGER C (nested)            │
│   │   └── Agent 3                   │
│   └── Agent 4                       │
└── OWNER 2 (promoted by Owner 1)     │
    └── ...                           │
──────────────────────────────────────┘
```

### Visibility Rules

| Role | Users Can See | Tickets Can See |
|------|---------------|-----------------|
| OWNER | All users in org | All tickets |
| MANAGER | Self + users they created (recursive) | Tickets assigned to their subtree |
| AGENT | Only self | Only their assigned tickets |

### Helper Function: Get Subtree User IDs
```typescript
// lib/rbac/hierarchy.ts
async function getSubtreeUserIds(userId: string): Promise<string[]> {
  const result: string[] = [userId]

  async function collectChildren(parentId: string) {
    const children = await prisma.user.findMany({
      where: { createdById: parentId },
      select: { id: true }
    })
    for (const child of children) {
      result.push(child.id)
      await collectChildren(child.id)  // Recursively get their children
    }
  }

  await collectChildren(userId)
  return result
}
```

### Transfer & Assignment Rules
- **OWNER**: Can assign to anyone in org
- **MANAGER**: Can assign only to users in their subtree
- **AGENT**: Cannot assign directly - must escalate to their Manager

### Teams + Hierarchy
Teams are **optional groupings** that work alongside hierarchy:
```
Manager A's Subtree
├── Agent 1 ──┬── Team "Support"
├── Agent 2 ──┘
└── Agent 3 ───── Team "Sales"
```

- Managers can create Teams within their subtree
- Teams enable group messaging/conversations
- Teams do NOT override hierarchy visibility rules
- Owners can create org-wide Teams

---

## AI-First Customer Flow

```
Customer Message (Chat/Email/Widget)
         │
         ▼
   ┌───────────┐
   │ AI Copilot│ ◄── Tries to resolve automatically
   └─────┬─────┘
         │
    Can resolve?
    ┌────┴────┐
   YES       NO
    │         │
    ▼         ▼
 Resolved   Create Ticket
            │
            ▼
      Round-Robin
      Assignment
            │
            ▼
    Agent Handles
```

### Key Points
- **AI is FIRST point of contact** for all customer queries
- AI attempts to answer from knowledge base automatically
- Only creates ticket if AI cannot resolve or customer requests human
- Round-robin assigns tickets to available agents in org
- Agent handles ticket, can use AI for response suggestions

### Implementation
```typescript
// POST /api/widget/message or /api/webhooks/email/inbound
async function handleCustomerMessage(message: string, context: Context) {
  // 1. Try AI resolution first
  const aiResponse = await copilot.tryResolve(message, context.orgId)

  if (aiResponse.resolved) {
    // AI handled it - return response, no ticket created
    return { type: 'ai_resolved', response: aiResponse.answer }
  }

  // 2. AI couldn't resolve - create ticket
  const ticket = await createTicket({
    ...context,
    channel: context.channel,
    aiAttempted: true
  })

  // 3. Round-robin assign to available agent
  const agent = await getNextAvailableAgent(context.orgId)
  await assignTicket(ticket.id, agent.id)

  return { type: 'escalated', ticketId: ticket.id }
}
```

---

## Real-time Architecture

### Socket.io Server (`lib/socket/server.ts`)
- Runs on custom server alongside Next.js
- Uses Redis adapter for horizontal scaling
- Authentication via handshake token

### Room Structure
```
org:{organizationId}      # Org-wide events (new tickets, presence)
user:{userId}             # Direct notifications
ticket:{ticketId}         # Ticket-specific updates
conversation:{id}         # Internal chat messages
```

### Key Events
| Event | Direction | Purpose |
|-------|-----------|---------|
| `message:new` | Server → Client | New message in ticket/conversation |
| `ticket:updated` | Server → Client | Ticket status/assignment change |
| `typing:indicator` | Bidirectional | User is typing |
| `presence:update` | Server → Client | User online/away/offline |

### Emitting from API Routes
```typescript
import { emitToTicket } from '@/lib/socket/server'

// After creating a message
emitToTicket(ticketId, 'message:new', { message: newMessage })
```

---

## Website Chat Widget

### Architecture
```
Customer Website                         wrrk.ai Backend
┌─────────────────┐                    ┌─────────────────┐
│  Chat Widget    │◄──── WebSocket ───►│  Socket.io      │
│  (JS Embed)     │                    │  Server         │
└─────────────────┘                    └─────────────────┘
        │                                      │
        │ REST API                             │
        ▼                                      ▼
┌─────────────────┐                    ┌─────────────────┐
│  /api/widget/*  │───────────────────►│  Ticket/Message │
│  endpoints      │                    │  Database       │
└─────────────────┘                    └─────────────────┘
```

### Widget Embed Code
```html
<!-- Customer adds this to their website -->
<script
  src="https://app.wrrk.ai/widget.js"
  data-org-id="org_xxx"
  data-position="bottom-right"
  data-primary-color="#4F46E5">
</script>
```

### Widget Features
| Feature | Description |
|---------|-------------|
| Anonymous Chat | Start conversation without login |
| Customer Identification | Optional email/name collection via pre-chat form |
| Real-time Messaging | WebSocket via Socket.io |
| Typing Indicators | Show when agent/customer is typing |
| File Attachments | Upload images/documents |
| Offline Mode | Leave message when no agents online |
| Chat History | Persist via localStorage + server sync |

### API Endpoints
```
POST /api/widget/init           # Get widget config, check agent availability
POST /api/widget/start-chat     # Create ticket (channel: CHAT) + Customer
POST /api/widget/message        # Send message (alternative to WebSocket)
GET  /api/widget/history        # Fetch previous messages (by session token)
WS   /socket.io                 # Real-time messaging (room: widget:{sessionId})
```

### Session Management
```typescript
// Widget generates sessionId, stored in localStorage
// Links to Customer record when email provided

interface WidgetSession {
  sessionId: string        // UUID, stored in browser
  ticketId?: string        // Created on first message
  customerId?: string      // Created/linked when email provided
  organizationId: string   // From embed code
}
```

### Socket.io Integration
- Widget connects with `{ type: 'widget', sessionId, orgId }`
- Joins room `widget:{sessionId}` for real-time updates
- Agent dashboard subscribes to ticket room when viewing

### Widget Bundle (`public/widget.js`)
- Standalone JS bundle (~50KB gzipped)
- No dependencies on customer's site
- Creates iframe to isolate styles
- Communicates via postMessage with parent

---

## Email Integration (SendGrid)

### Inbound Flow
```
Email arrives → SendGrid Inbound Parse → POST /api/webhooks/email/inbound
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    ▼                         ▼                         ▼
            New ticket?              Reply to existing?          Unknown sender?
            Create Ticket            Add Message to Ticket       Create Customer + Ticket
```

### Outbound Flow
```typescript
// When agent replies to ticket
import { sendEmail } from '@/lib/email/sendgrid'

await sendEmail({
  to: customer.email,
  subject: `Re: [Ticket #${ticket.number}] ${ticket.subject}`,
  html: messageContent,
  headers: {
    'In-Reply-To': originalMessageId,
    'References': threadMessageIds
  }
})
```

### Thread Matching
- Store `emailMessageId` on each Message
- Parse `In-Reply-To` header to find parent ticket
- Use `References` header for full thread context

---

## AI Copilot (`lib/ai/copilot.ts`)

### Available Functions
| Function | Purpose | Input |
|----------|---------|-------|
| `suggestResponse` | Draft reply to customer | Ticket + latest message |
| `summarizeTicket` | Summarize conversation | All messages |
| `detectSentiment` | Analyze customer mood | Message text |
| `categorizeTicket` | Auto-assign category | Subject + description |
| `generateKnowledgeBaseAnswer` | Answer from KB | Question + articles |

### API Endpoint
```typescript
// POST /api/copilot/suggest
{
  "ticketId": "...",
  "messageId": "...",
  "action": "suggest_response" | "summarize" | "categorize"
}
```

### Usage Tracking
All AI interactions logged to `CopilotInteraction` for:
- Usage analytics
- Response quality tracking
- Cost monitoring

---

## Authentication Flow

### NextAuth Configuration (`lib/auth/config.ts`)
- **Providers**: AWS Cognito (production), Credentials (development)
- **Session**: JWT-based, includes `organizationId` and `role`
- **Callbacks**: Sync user data from database on sign-in

### Protected Routes
```typescript
// Middleware or page-level
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth/config'

const session = await getServerSession(authOptions)
if (!session) redirect('/login')
```

---

## Environment Variables

```bash
# Database
DATABASE_URL="postgresql://..."

# Redis (Socket.io adapter)
REDIS_URL="redis://..."

# Auth
NEXTAUTH_SECRET="..."
NEXTAUTH_URL="https://app.wrrk.ai"
COGNITO_CLIENT_ID="..."
COGNITO_CLIENT_SECRET="..."
COGNITO_ISSUER="https://cognito-idp.{region}.amazonaws.com/{poolId}"

# AWS (AI + other services)
AWS_REGION="us-east-1"
AWS_ACCESS_KEY_ID="..."
AWS_SECRET_ACCESS_KEY="..."

# Email
SENDGRID_API_KEY="..."
SENDGRID_WEBHOOK_SECRET="..."

# App
NEXT_PUBLIC_APP_URL="https://app.wrrk.ai"
```

---

## Deployment Checklist

1. **Database**: Create Supabase project, run `pnpm db:push`
2. **Redis**: Create Upstash Redis instance
3. **Vercel**: Connect repo, set environment variables
4. **SendGrid**: Configure inbound parse webhook URL
5. **AWS**: Set up IAM user with Bedrock access
6. **DNS**: Point domain to Vercel

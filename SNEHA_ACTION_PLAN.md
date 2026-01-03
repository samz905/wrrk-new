# Sneha's Action Plan

This is your guide for building 3 features. Work through them in order.

---

## Your Features

| # | Feature | Priority |
|---|---------|----------|
| 2 | Ticketing System | Do first (with visibility rules!) |
| 3 | Messaging | Do second |
| 5 | AI Copilot | Do third (AI-first flow!) |

---

## CRITICAL: Understanding Hierarchy & Visibility

**READ THIS FIRST** - Your features must respect user hierarchy:

```
OWNER(s) ─────────────────────────────┐
├── MANAGER A                         │
│   ├── Agent 1                       │  Organization
│   └── Agent 2                       │
├── MANAGER B                         │
│   └── Agent 3                       │
└── OWNER 2 (promoted)                │
──────────────────────────────────────┘
```

**Ticket Visibility Rules**:
- OWNER: Sees ALL tickets in org
- MANAGER: Sees tickets assigned to users in their subtree
- AGENT: Sees ONLY their assigned tickets

**Assignment Rules**:
- OWNER: Can assign to anyone
- MANAGER: Can only assign to their subtree
- AGENT: Cannot assign - must ESCALATE to their Manager

---

## CRITICAL: AI-First Flow

Your AI Copilot is the **FIRST point of contact** for customers:

```
Customer Message → AI Tries to Resolve → Can't? → Create Ticket → Round-Robin Assign
```

You need to create a `tryAIResolve()` function that Suraj will call from Email and Widget.

---

## Before You Start

### 1. Set up your environment
```bash
# Clone and install
cd wrrk-new/wrrk.ai
pnpm install

# Copy environment file
cp .env.example .env.local

# Ask Samarth for these values and add to .env.local:
# - DATABASE_URL
# - REDIS_URL
# - NEXTAUTH_SECRET
# - AWS credentials (for AI Copilot)
```

### 2. Start the dev server
```bash
pnpm dev
```

### 3. Test the app loads
- Open http://localhost:3000
- You should see the login page

---

## Feature 2: Ticketing System (WITH VISIBILITY!)

### What you're building
Agents can create, view, filter, assign, and manage support tickets.
**WITH HIERARCHY** - users only see tickets assigned to their subtree.

### Files you'll work on
```
apps/web/app/api/tickets/route.ts           # List & create tickets (filtered!)
apps/web/app/api/tickets/[id]/route.ts      # Get/update single ticket
apps/web/app/api/customers/route.ts         # List & create customers
apps/web/app/api/customers/[id]/route.ts    # Get/update single customer
```

### Step-by-step

#### Step 2.1: Make GET /api/tickets work WITH VISIBILITY

Open `apps/web/app/api/tickets/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
import { getSubtreeUserIds } from '@/lib/rbac/hierarchy'  // From Suraj!

export async function GET(req: NextRequest) {
  // 1. Check if user is logged in
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  // 2. Get filter parameters from URL
  const { searchParams } = new URL(req.url)
  const status = searchParams.get('status')
  const priority = searchParams.get('priority')
  const assigneeId = searchParams.get('assigneeId')
  const customerId = searchParams.get('customerId')
  const search = searchParams.get('search')
  const page = parseInt(searchParams.get('page') || '1')
  const limit = parseInt(searchParams.get('limit') || '20')

  // 3. Get visible user IDs based on hierarchy (CRITICAL!)
  const visibleUserIds = await getSubtreeUserIds(
    session.user.id,
    session.user.role,
    session.user.organizationId
  )

  // 4. Build the filter object WITH VISIBILITY
  const where: any = {
    organizationId: session.user.organizationId,
    // HIERARCHY FILTER: Only see tickets assigned to visible users!
    assigneeId: { in: visibleUserIds }
  }

  if (status) where.status = status
  if (priority) where.priority = priority
  if (assigneeId) where.assigneeId = assigneeId  // Additional filter within visible
  if (customerId) where.customerId = customerId
  if (search) {
    where.subject = { contains: search, mode: 'insensitive' }
  }

  // 5. Fetch tickets with related data
  const [tickets, total] = await Promise.all([
    prisma.ticket.findMany({
      where,
      include: {
        customer: { select: { id: true, name: true, email: true } },
        assignee: { select: { id: true, name: true, avatar: true } }
      },
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { updatedAt: 'desc' }
    }),
    prisma.ticket.count({ where })
  ])

  return NextResponse.json({
    data: tickets,
    pagination: { page, limit, total, totalPages: Math.ceil(total / limit) }
  })
}
```

#### Step 2.2: Make POST /api/tickets work (create ticket)

Add this to the same file:

```typescript
export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const body = await req.json()
  const { subject, description, priority = 'MEDIUM', customerId, assigneeId } = body

  // Validate required fields
  if (!subject) {
    return NextResponse.json({ error: 'Subject is required' }, { status: 400 })
  }

  const organizationId = session.user.organizationId

  // Generate ticket number (e.g., ACME-0001)
  const org = await prisma.organization.findUnique({
    where: { id: organizationId }
  })

  const lastTicket = await prisma.ticket.findFirst({
    where: { organizationId },
    orderBy: { createdAt: 'desc' }
  })

  // Create prefix from org name (first 4 letters, uppercase)
  const prefix = org?.name?.substring(0, 4).toUpperCase() || 'TKT'

  // Get next number
  let nextNumber = 1
  if (lastTicket?.ticketNumber) {
    const parts = lastTicket.ticketNumber.split('-')
    nextNumber = parseInt(parts[1]) + 1
  }

  const ticketNumber = `${prefix}-${nextNumber.toString().padStart(4, '0')}`

  // Create the ticket
  const ticket = await prisma.ticket.create({
    data: {
      ticketNumber,
      subject,
      description,
      status: 'OPEN',
      priority,
      channel: 'PORTAL',  // Default channel for manually created tickets
      customerId,
      assigneeId,
      organizationId
    },
    include: {
      customer: true,
      assignee: true
    }
  })

  // Create audit log
  await prisma.auditLog.create({
    data: {
      action: 'TICKET_CREATED',
      entityType: 'TICKET',
      entityId: ticket.id,
      userId: session.user.id,
      organizationId,
      metadata: { ticketNumber }
    }
  })

  return NextResponse.json({ data: ticket }, { status: 201 })
}
```

#### Step 2.3: Test ticket creation
```bash
# In Postman or browser console:
POST http://localhost:3000/api/tickets
Content-Type: application/json

{
  "subject": "My first test ticket",
  "description": "This is a test",
  "priority": "HIGH"
}

# Should return the created ticket with a number like ACME-0001
```

#### Step 2.4: Make GET /api/tickets/[id] work

Create `apps/web/app/api/tickets/[id]/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'

export async function GET(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const ticket = await prisma.ticket.findFirst({
    where: {
      id: params.id,
      organizationId: session.user.organizationId  // Security: only own org's tickets
    },
    include: {
      customer: true,
      assignee: { select: { id: true, name: true, avatar: true, email: true } },
      messages: {
        include: {
          user: { select: { id: true, name: true, avatar: true } },
          customer: { select: { id: true, name: true } }
        },
        orderBy: { createdAt: 'asc' }
      }
    }
  })

  if (!ticket) {
    return NextResponse.json({ error: 'Ticket not found' }, { status: 404 })
  }

  return NextResponse.json({ data: ticket })
}
```

#### Step 2.5: Make PATCH /api/tickets/[id] work WITH ASSIGNMENT RULES

Add to the same file:

```typescript
import { getSubtreeUserIds, getManager } from '@/lib/rbac/hierarchy'

export async function PATCH(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const body = await req.json()
  const { status, priority, assigneeId, subject, escalate } = body

  // First check ticket exists and user can see it
  const visibleUserIds = await getSubtreeUserIds(
    session.user.id,
    session.user.role,
    session.user.organizationId
  )

  const existingTicket = await prisma.ticket.findFirst({
    where: {
      id: params.id,
      organizationId: session.user.organizationId,
      assigneeId: { in: visibleUserIds }  // Must be visible to user!
    }
  })

  if (!existingTicket) {
    return NextResponse.json({ error: 'Ticket not found or not visible' }, { status: 404 })
  }

  // ASSIGNMENT RULES
  if (assigneeId) {
    // AGENT cannot assign - must escalate
    if (session.user.role === 'AGENT') {
      return NextResponse.json({
        error: 'Agents cannot assign tickets. Use escalate instead.'
      }, { status: 403 })
    }

    // MANAGER can only assign to their subtree
    if (session.user.role === 'MANAGER') {
      if (!visibleUserIds.includes(assigneeId)) {
        return NextResponse.json({
          error: 'Can only assign to users in your team'
        }, { status: 403 })
      }
    }
    // OWNER can assign to anyone (no check needed)
  }

  // ESCALATION: Agent wants to escalate to manager
  let newAssigneeId = assigneeId
  if (escalate && session.user.role === 'AGENT') {
    const managerId = await getManager(session.user.id)
    if (!managerId) {
      return NextResponse.json({ error: 'No manager found' }, { status: 400 })
    }
    newAssigneeId = managerId
  }

  // Update the ticket
  const ticket = await prisma.ticket.update({
    where: { id: params.id },
    data: {
      ...(status && { status }),
      ...(priority && { priority }),
      ...(newAssigneeId !== undefined && { assigneeId: newAssigneeId }),
      ...(subject && { subject }),
      updatedAt: new Date()
    },
    include: { customer: true, assignee: true }
  })

  // Create audit log
  await prisma.auditLog.create({
    data: {
      action: escalate ? 'TICKET_ESCALATED' : 'TICKET_UPDATED',
      entityType: 'TICKET',
      entityId: ticket.id,
      userId: session.user.id,
      organizationId: session.user.organizationId,
      metadata: { changes: { status, priority, assigneeId: newAssigneeId, escalate } }
    }
  })

  // Emit socket event (Suraj's feature)
  // emitToTicket(ticket.id, 'ticket:updated', ticket)

  return NextResponse.json({ data: ticket })
}
```

#### Step 2.6: Create customer endpoints

Create `apps/web/app/api/customers/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'

export async function GET(req: NextRequest) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const { searchParams } = new URL(req.url)
  const search = searchParams.get('search')
  const page = parseInt(searchParams.get('page') || '1')
  const limit = parseInt(searchParams.get('limit') || '20')

  const where: any = {
    organizationId: session.user.organizationId
  }

  if (search) {
    where.OR = [
      { name: { contains: search, mode: 'insensitive' } },
      { email: { contains: search, mode: 'insensitive' } }
    ]
  }

  const [customers, total] = await Promise.all([
    prisma.customer.findMany({
      where,
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { createdAt: 'desc' }
    }),
    prisma.customer.count({ where })
  ])

  return NextResponse.json({
    data: customers,
    pagination: { page, limit, total, totalPages: Math.ceil(total / limit) }
  })
}

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const { name, email, phone, company } = await req.json()

  if (!email) {
    return NextResponse.json({ error: 'Email is required' }, { status: 400 })
  }

  // Check if customer already exists
  const existing = await prisma.customer.findFirst({
    where: {
      email,
      organizationId: session.user.organizationId
    }
  })

  if (existing) {
    return NextResponse.json({ error: 'Customer with this email already exists' }, { status: 409 })
  }

  const customer = await prisma.customer.create({
    data: {
      name: name || email,
      email,
      phone,
      company,
      organizationId: session.user.organizationId
    }
  })

  return NextResponse.json({ data: customer }, { status: 201 })
}
```

### How to Test Feature 2 (Visibility + Assignment)

**Setup** (work with Suraj to create these):
1. Owner creates Manager M1
2. M1 creates Agent A1, A2
3. Owner creates Manager M2
4. M2 creates Agent A3
5. Create tickets assigned to A1, A2, A3

**Test Cases**:
| # | Login As | Action | Expected |
|---|----------|--------|----------|
| 1 | Owner | GET /api/tickets | See ALL tickets |
| 2 | M1 | GET /api/tickets | See only A1 and A2's tickets |
| 3 | A1 | GET /api/tickets | See only A1's tickets |
| 4 | A1 | Assign ticket to A2 | **FAIL** - Agents can't assign |
| 5 | A1 | Escalate ticket | Works - goes to M1 |
| 6 | M1 | Assign ticket to A1 | Works |
| 7 | M1 | Assign ticket to A3 | **FAIL** - A3 not in M1's subtree |

**Browser Steps**:
1. Login as Owner → Go to /tickets → See all tickets
2. Logout, login as Agent A1
3. Go to /tickets → Should only see A1's assigned tickets
4. Try to assign ticket → Should only see "Escalate" option
5. Click Escalate → Ticket goes to Manager

### How to know Feature 2 is done
- [ ] Ticket list respects hierarchy (verified with 3 different users)
- [ ] Agents can only escalate, not assign
- [ ] Managers can only assign within subtree
- [ ] Owners can assign to anyone
- [ ] Filters work within visible tickets

---

## Feature 3: Messaging

### What you're building
Agents and customers can exchange messages within a ticket.

### Files you'll work on
```
apps/web/app/api/tickets/[id]/messages/route.ts   # List & send messages
apps/web/app/api/upload/route.ts                   # File uploads
```

### Step-by-step

#### Step 3.1: Create messages endpoint

Create `apps/web/app/api/tickets/[id]/messages/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
// Import from Suraj's socket server (ask him for the path)
// import { emitToTicket } from '@/lib/socket/server'

export async function GET(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  // Verify ticket belongs to user's org
  const ticket = await prisma.ticket.findFirst({
    where: {
      id: params.id,
      organizationId: session.user.organizationId
    }
  })

  if (!ticket) {
    return NextResponse.json({ error: 'Ticket not found' }, { status: 404 })
  }

  // Get messages
  const messages = await prisma.message.findMany({
    where: { ticketId: params.id },
    include: {
      user: { select: { id: true, name: true, avatar: true } },
      customer: { select: { id: true, name: true, email: true } }
    },
    orderBy: { createdAt: 'asc' }  // Oldest first (like a chat)
  })

  return NextResponse.json({ data: messages })
}

export async function POST(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const { content, attachments } = await req.json()

  if (!content && (!attachments || attachments.length === 0)) {
    return NextResponse.json({ error: 'Message cannot be empty' }, { status: 400 })
  }

  // Verify ticket exists and get info
  const ticket = await prisma.ticket.findFirst({
    where: {
      id: params.id,
      organizationId: session.user.organizationId
    },
    include: { customer: true }
  })

  if (!ticket) {
    return NextResponse.json({ error: 'Ticket not found' }, { status: 404 })
  }

  // Create the message
  const message = await prisma.message.create({
    data: {
      content,
      ticketId: params.id,
      senderType: 'AGENT',  // Message is from logged-in agent
      userId: session.user.id,
      // attachments: attachments  // If you have attachments relation
    },
    include: {
      user: { select: { id: true, name: true, avatar: true } }
    }
  })

  // Update ticket's updatedAt
  await prisma.ticket.update({
    where: { id: params.id },
    data: { updatedAt: new Date() }
  })

  // Emit socket event for real-time (Suraj's feature)
  // Uncomment when Suraj's socket server is ready:
  // emitToTicket(params.id, 'message:new', {
  //   id: message.id,
  //   content: message.content,
  //   senderType: 'AGENT',
  //   sender: { id: session.user.id, name: session.user.name },
  //   createdAt: message.createdAt
  // })

  // If this is an EMAIL channel ticket, send email to customer
  // (Suraj's email feature - uncomment when ready)
  // if (ticket.channel === 'EMAIL' && ticket.customer?.email) {
  //   await sendTicketReply(ticket, message, ticket.customer)
  // }

  return NextResponse.json({ data: message }, { status: 201 })
}
```

#### Step 3.2: Create file upload endpoint

Create `apps/web/app/api/upload/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { authOptions } from '@/lib/auth/config'
import { put } from '@vercel/blob'  // Using Vercel Blob storage

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const formData = await req.formData()
  const file = formData.get('file') as File

  if (!file) {
    return NextResponse.json({ error: 'No file provided' }, { status: 400 })
  }

  // Validate file size (max 10MB)
  if (file.size > 10 * 1024 * 1024) {
    return NextResponse.json({ error: 'File too large (max 10MB)' }, { status: 400 })
  }

  // Validate file type
  const allowedTypes = [
    'image/jpeg', 'image/png', 'image/gif', 'image/webp',
    'application/pdf',
    'application/msword',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    'text/plain'
  ]

  if (!allowedTypes.includes(file.type)) {
    return NextResponse.json({ error: 'File type not allowed' }, { status: 400 })
  }

  try {
    // Upload to Vercel Blob
    const blob = await put(file.name, file, {
      access: 'public',
    })

    return NextResponse.json({
      url: blob.url,
      filename: file.name,
      size: file.size,
      mimeType: file.type
    })
  } catch (error) {
    console.error('Upload error:', error)
    return NextResponse.json({ error: 'Upload failed' }, { status: 500 })
  }
}

// Configure body size limit
export const config = {
  api: {
    bodyParser: false,
  },
}
```

**Note**: If Vercel Blob is not set up, you can use a simpler local storage for testing:

```typescript
// Simple version without Vercel Blob (for testing)
import { writeFile } from 'fs/promises'
import { join } from 'path'

export async function POST(req: NextRequest) {
  // ... auth and validation same as above ...

  const bytes = await file.arrayBuffer()
  const buffer = Buffer.from(bytes)

  // Save to public/uploads folder
  const filename = `${Date.now()}-${file.name}`
  const path = join(process.cwd(), 'public', 'uploads', filename)
  await writeFile(path, buffer)

  return NextResponse.json({
    url: `/uploads/${filename}`,
    filename: file.name,
    size: file.size,
    mimeType: file.type
  })
}
```

#### Step 3.3: Test messaging
```bash
# Create a message
POST http://localhost:3000/api/tickets/{ticketId}/messages
Content-Type: application/json

{
  "content": "Hello, how can I help you today?"
}

# Should return the created message
```

### How to know Feature 3 is done
- [ ] GET /api/tickets/[id]/messages returns message list
- [ ] POST /api/tickets/[id]/messages creates new message
- [ ] POST /api/upload handles file uploads
- [ ] Messages show sender name and timestamp
- [ ] Frontend ticket view shows messages
- [ ] Frontend has message composer that works
- [ ] Can attach files to messages

---

## Feature 5: AI Copilot (AI-FIRST!)

### What you're building
AI is the **FIRST point of contact** for customers!
Also helps agents by suggesting responses and analyzing sentiment.

### The AI-First Flow
```
Customer Message (Widget/Email)
         │
         ▼
   ┌───────────────┐
   │ tryAIResolve()│ ◄── YOU CREATE THIS
   └───────┬───────┘
           │
      Can resolve?
      ┌────┴────┐
     YES       NO
      │         │
      ▼         ▼
   Return    Return
   response  { resolved: false }
   (no ticket)  (Suraj creates ticket)
```

### Files you'll work on
```
apps/web/lib/ai/copilot.ts                    # AI functions (including tryAIResolve!)
apps/web/app/api/copilot/suggest/route.ts     # Suggest response for agents
apps/web/app/api/copilot/resolve/route.ts     # AI auto-resolve for customers
apps/web/app/api/copilot/categorize/route.ts  # Categorize ticket
```

### Step-by-step

#### Step 5.1: Understand the AI library

Look at `apps/web/lib/ai/copilot.ts`. It should use AWS Bedrock:

```typescript
import { BedrockRuntimeClient, InvokeModelCommand } from '@aws-sdk/client-bedrock-runtime'

const client = new BedrockRuntimeClient({
  region: process.env.AWS_REGION || 'us-east-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!
  }
})

// Call Claude via Bedrock
async function callClaude(prompt: string): Promise<string> {
  const response = await client.send(new InvokeModelCommand({
    modelId: 'anthropic.claude-3-sonnet-20240229-v1:0',
    contentType: 'application/json',
    body: JSON.stringify({
      anthropic_version: 'bedrock-2023-05-31',
      max_tokens: 1000,
      messages: [
        { role: 'user', content: prompt }
      ]
    })
  }))

  const result = JSON.parse(new TextDecoder().decode(response.body))
  return result.content[0].text
}

// Generate a reply suggestion
export async function suggestResponse(
  ticketSubject: string,
  customerMessage: string,
  previousMessages: string[] = []
): Promise<string> {
  const context = previousMessages.length > 0
    ? `Previous conversation:\n${previousMessages.join('\n')}\n\n`
    : ''

  const prompt = `You are a helpful customer support agent. A customer has contacted support.

${context}Subject: ${ticketSubject}

Customer's latest message:
${customerMessage}

Write a helpful, professional, and friendly response to help the customer. Keep it concise but thorough.`

  return callClaude(prompt)
}

// Analyze sentiment of a message
export async function analyzeSentiment(message: string): Promise<{
  sentiment: 'positive' | 'neutral' | 'negative'
  confidence: number
}> {
  const prompt = `Analyze the sentiment of this customer message. Reply with ONLY a JSON object like {"sentiment": "positive", "confidence": 0.85}

Message: "${message}"

Respond with only the JSON, no other text.`

  const response = await callClaude(prompt)

  try {
    return JSON.parse(response)
  } catch {
    return { sentiment: 'neutral', confidence: 0.5 }
  }
}

// Categorize a ticket
export async function categorizeTicket(
  subject: string,
  description: string,
  categories: string[]
): Promise<{ category: string; confidence: number }> {
  const prompt = `Categorize this support ticket into one of these categories: ${categories.join(', ')}

Subject: ${subject}
Description: ${description}

Reply with ONLY a JSON object like {"category": "billing", "confidence": 0.9}

Respond with only the JSON, no other text.`

  const response = await callClaude(prompt)

  try {
    return JSON.parse(response)
  } catch {
    return { category: categories[0], confidence: 0.5 }
  }
}

// Summarize a ticket conversation
export async function summarizeTicket(messages: string[]): Promise<string> {
  const prompt = `Summarize this customer support conversation in 2-3 sentences:

${messages.join('\n\n')}

Provide a brief summary of the issue and current status.`

  return callClaude(prompt)
}

// ============================================
// AI-FIRST: Try to resolve customer query automatically
// Suraj calls this from Widget and Email endpoints!
// ============================================
export async function tryAIResolve(
  customerMessage: string,
  organizationId: string
): Promise<{
  resolved: boolean
  response?: string
  confidence: number
}> {
  // Get knowledge base or FAQs for this org (if available)
  const knowledgeBase = await prisma.knowledgeArticle?.findMany({
    where: { organizationId },
    take: 10
  }).catch(() => [])  // Ignore if table doesn't exist

  const kbContext = knowledgeBase.length > 0
    ? `\nKnowledge Base:\n${knowledgeBase.map(a => `- ${a.title}: ${a.content}`).join('\n')}`
    : ''

  const prompt = `You are an AI support assistant. A customer has sent a message.
${kbContext}

Customer message: "${customerMessage}"

Can you fully answer this question? If yes, provide a helpful response.
If no (question is too complex, needs human, or you're unsure), say you cannot.

Reply with JSON only:
{
  "canResolve": true/false,
  "confidence": 0.0-1.0,
  "response": "your response if canResolve is true"
}

Important:
- Only resolve simple, common questions you're confident about
- If customer asks for human/agent/person, set canResolve to false
- If question requires account lookup, set canResolve to false
- Be helpful but honest about limitations`

  try {
    const result = await callClaude(prompt)
    const parsed = JSON.parse(result)

    return {
      resolved: parsed.canResolve && parsed.confidence > 0.7,
      response: parsed.response,
      confidence: parsed.confidence
    }
  } catch (error) {
    console.error('AI resolve error:', error)
    return { resolved: false, confidence: 0 }
  }
}
```

#### Step 5.2: Create suggest response endpoint

Create `apps/web/app/api/copilot/suggest/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
import { suggestResponse } from '@/lib/ai/copilot'

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const { ticketId } = await req.json()

  // Get ticket with messages
  const ticket = await prisma.ticket.findFirst({
    where: {
      id: ticketId,
      organizationId: session.user.organizationId
    },
    include: {
      messages: {
        orderBy: { createdAt: 'desc' },
        take: 10,  // Last 10 messages for context
        include: {
          user: { select: { name: true } },
          customer: { select: { name: true } }
        }
      }
    }
  })

  if (!ticket) {
    return NextResponse.json({ error: 'Ticket not found' }, { status: 404 })
  }

  // Get the last customer message
  const lastCustomerMessage = ticket.messages.find(m => m.senderType === 'CUSTOMER')

  if (!lastCustomerMessage) {
    return NextResponse.json({ error: 'No customer message to respond to' }, { status: 400 })
  }

  // Format previous messages for context
  const previousMessages = ticket.messages
    .reverse()
    .slice(0, -1)  // Exclude the last message
    .map(m => {
      const sender = m.senderType === 'CUSTOMER'
        ? m.customer?.name || 'Customer'
        : m.user?.name || 'Agent'
      return `${sender}: ${m.content}`
    })

  try {
    // Get AI suggestion
    const suggestion = await suggestResponse(
      ticket.subject,
      lastCustomerMessage.content,
      previousMessages
    )

    // Log the interaction
    await prisma.copilotInteraction.create({
      data: {
        action: 'SUGGEST_RESPONSE',
        ticketId,
        userId: session.user.id,
        organizationId: session.user.organizationId,
        input: lastCustomerMessage.content,
        output: suggestion
      }
    })

    return NextResponse.json({ suggestion })

  } catch (error) {
    console.error('AI error:', error)
    return NextResponse.json(
      { error: 'AI service unavailable', fallback: true },
      { status: 503 }
    )
  }
}
```

#### Step 5.3: Create categorize endpoint

Create `apps/web/app/api/copilot/categorize/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
import { categorizeTicket } from '@/lib/ai/copilot'

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const { ticketId, categories } = await req.json()

  // Default categories if not provided
  const categoryList = categories || [
    'billing',
    'technical',
    'account',
    'feature-request',
    'general'
  ]

  // Get ticket
  const ticket = await prisma.ticket.findFirst({
    where: {
      id: ticketId,
      organizationId: session.user.organizationId
    }
  })

  if (!ticket) {
    return NextResponse.json({ error: 'Ticket not found' }, { status: 404 })
  }

  try {
    const result = await categorizeTicket(
      ticket.subject,
      ticket.description || '',
      categoryList
    )

    // Log the interaction
    await prisma.copilotInteraction.create({
      data: {
        action: 'CATEGORIZE',
        ticketId,
        userId: session.user.id,
        organizationId: session.user.organizationId,
        input: ticket.subject,
        output: JSON.stringify(result)
      }
    })

    return NextResponse.json(result)

  } catch (error) {
    console.error('AI error:', error)
    return NextResponse.json(
      { error: 'AI service unavailable' },
      { status: 503 }
    )
  }
}
```

#### Step 5.4: Add sentiment to message creation

In your POST /api/tickets/[id]/messages endpoint, add sentiment analysis:

```typescript
import { analyzeSentiment } from '@/lib/ai/copilot'

// In POST handler, after creating message:

// If message is from customer, analyze sentiment
if (senderType === 'CUSTOMER') {
  try {
    const sentiment = await analyzeSentiment(content)

    // Update message with sentiment
    await prisma.message.update({
      where: { id: message.id },
      data: {
        metadata: { sentiment: sentiment.sentiment }
      }
    })

    // Or store in a separate field if available
  } catch (error) {
    console.log('Sentiment analysis failed, continuing without it')
  }
}
```

#### Step 5.5: Test AI features
```bash
# Test suggestion
POST http://localhost:3000/api/copilot/suggest
Content-Type: application/json

{
  "ticketId": "your-ticket-id"
}

# Should return { "suggestion": "..." }

# Test categorization
POST http://localhost:3000/api/copilot/categorize
Content-Type: application/json

{
  "ticketId": "your-ticket-id",
  "categories": ["billing", "technical", "general"]
}

# Should return { "category": "...", "confidence": 0.8 }
```

### How to Test Feature 5 (AI-First + Copilot)

**Test AI-First (with Suraj)**:
| # | Test | Expected |
|---|------|----------|
| 1 | Widget: "What are your hours?" | AI responds, no ticket |
| 2 | Widget: "Talk to a human" | Ticket created, no AI response |
| 3 | Widget: Complex question | AI can't resolve → ticket created |
| 4 | Email: Simple FAQ | AI responds via email |
| 5 | Email: Complex issue | Ticket created |

**Test Agent Copilot**:
| # | Test | Expected |
|---|------|----------|
| 1 | Click "Suggest Response" on ticket | AI drafts response |
| 2 | Edit and send suggestion | Message sent with edits |
| 3 | New customer message arrives | Sentiment badge appears |
| 4 | Call categorize on new ticket | Category returned |

**Browser Steps**:
1. Create a ticket with a customer message
2. Go to ticket view
3. Click "Suggest Response" button
4. Verify AI suggestion appears
5. Edit the suggestion
6. Click Send → message should be sent
7. Check customer message → sentiment badge should show

### How to know Feature 5 is done
- [ ] `tryAIResolve()` works (Suraj tests via widget/email)
- [ ] Simple questions get AI response, complex create tickets
- [ ] "Suggest Response" generates reply for agents
- [ ] Sentiment analysis works on customer messages
- [ ] Auto-categorization works on new tickets

---

## Connecting with Suraj's Features

Your features connect with Suraj's work in several places:

### 1. Suraj imports YOUR `tryAIResolve()` function
Suraj's email webhook and widget will call your function:
```typescript
// In Suraj's code (email/widget):
import { tryAIResolve } from '@/lib/ai/copilot'  // YOUR function!

const result = await tryAIResolve(message, organizationId)
if (result.resolved) {
  // AI handled it, no ticket created
} else {
  // Create ticket...
}
```
**Coordinate with Suraj**: Make sure your `tryAIResolve()` function is exported!

### 2. You import Suraj's hierarchy helper
For ticket visibility filtering:
```typescript
// In your tickets API:
import { getSubtreeUserIds, getManager } from '@/lib/rbac/hierarchy'  // Suraj creates this!
```
**Coordinate with Suraj**: Make sure he creates this helper first!

### 3. Real-time messages
When you create a message, emit a socket event:
```typescript
import { emitToTicket } from '@/lib/socket/server'  // Suraj's

emitToTicket(ticketId, 'message:new', messageData)
```

### 4. Email replies
When agent replies to an EMAIL channel ticket:
```typescript
import { sendTicketReply } from '@/lib/email/sendgrid'  // Suraj's

if (ticket.channel === 'EMAIL') {
  await sendTicketReply(ticket, message, ticket.customer)
}
```

**Talk to Suraj when you need these integrations!**

---

## Daily Checklist

### Every morning
- [ ] Pull latest code: `git pull`
- [ ] Check if any PRs need your review
- [ ] Run `pnpm dev` and make sure app works

### Before committing
- [ ] Test your changes manually
- [ ] Run `pnpm lint` to check for errors
- [ ] Write clear commit message

### When stuck
1. Check the error message carefully
2. Search the codebase for similar code
3. Ask Samarth for help
4. Don't stay stuck for more than 30 minutes

---

## Quick Reference

### Common imports
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
```

### API response patterns
```typescript
// Success
return NextResponse.json({ data: result })
return NextResponse.json({ data: result }, { status: 201 })  // Created

// Errors
return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
return NextResponse.json({ error: 'Not allowed' }, { status: 403 })
return NextResponse.json({ error: 'Not found' }, { status: 404 })
return NextResponse.json({ error: 'Bad request' }, { status: 400 })
```

### Ticket statuses
- `OPEN` - New ticket, not yet worked on
- `IN_PROGRESS` - Agent is working on it
- `WAITING` - Waiting for customer response
- `RESOLVED` - Issue fixed, waiting for confirmation
- `CLOSED` - Ticket complete

### Priority levels
- `LOW` - Can wait
- `MEDIUM` - Normal priority
- `HIGH` - Should be addressed soon
- `URGENT` - Needs immediate attention

Good luck Sneha! Ask Samarth if you're stuck.

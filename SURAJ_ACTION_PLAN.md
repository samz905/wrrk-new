# Suraj's Action Plan

This is your guide for building 4 features. Work through them in order.

---

## Your Features

| # | Feature | Priority |
|---|---------|----------|
| 1 | User & Org Management | Do first (includes hierarchy!) |
| 4 | Real-time Features | Do second |
| 6 | Email Channel | Do third (AI-first!) |
| 7 | Website Chat Widget | Do fourth (AI-first!) |

---

## CRITICAL: User Hierarchy

**READ THIS FIRST** - The app uses hierarchical user management:

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

**Key Rules**:
- Roles: `OWNER > MANAGER > AGENT` (3 roles, not 4)
- Users only see users they created (`createdById` field)
- OWNER sees everyone, MANAGER sees subtree, AGENT sees only self
- When inviting, set `createdById` to the inviter's ID

---

## CRITICAL: AI-First Flow

For Email and Chat Widget, **AI handles customers first**:

```
Customer Message → AI Tries to Resolve → Can't? → Create Ticket → Round-Robin Assign
```

See Feature 6 and 7 for implementation details.

---

## Schema Changes YOU Must Do

Before starting, update `packages/database/prisma/schema.prisma`:

```prisma
// 1. Change Role enum (was 4 roles, now 3)
enum Role {
  OWNER     // Was: ADMIN
  MANAGER   // Was: MANAGER + SUPERVISOR
  AGENT     // Same
}

// 2. Add to User model:
model User {
  // ... existing fields ...

  // ADD THESE:
  createdById   String?
  createdBy     User?    @relation("CreatedUsers", fields: [createdById], references: [id])
  createdUsers  User[]   @relation("CreatedUsers")
}
```

Then run: `pnpm db:push`

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
# - AWS credentials
# - SENDGRID_API_KEY
```

### 2. Start the dev server
```bash
pnpm dev
```

### 3. Test the app loads
- Open http://localhost:3000
- You should see the login page

---

## Feature 1: User & Org Management (WITH HIERARCHY)

### What you're building
Users can log in, update their profile, and invite teammates. **WITH HIERARCHY** - users only see who they created.

### Files you'll work on
```
apps/web/lib/rbac/hierarchy.ts            # NEW: Hierarchy helper (create this!)
apps/web/app/api/users/route.ts           # List users (filtered by hierarchy!)
apps/web/app/api/users/[id]/route.ts      # Get/update single user
apps/web/app/api/organizations/[id]/route.ts  # Org settings
apps/web/app/api/teams/route.ts           # List/create teams
apps/web/app/api/invitations/route.ts     # Send invitations (sets createdById!)
```

### Step-by-step

#### Step 1.0: Create Hierarchy Helper (FIRST!)

Create `apps/web/lib/rbac/hierarchy.ts`:

```typescript
import { prisma } from '@wrrk/database'

/**
 * Get all user IDs in a user's subtree (themselves + all users they created, recursively)
 * OWNER: Returns all users in org
 * MANAGER: Returns self + users they created (recursively)
 * AGENT: Returns only self
 */
export async function getSubtreeUserIds(
  userId: string,
  role: string,
  organizationId: string
): Promise<string[]> {
  // OWNER sees everyone
  if (role === 'OWNER') {
    const allUsers = await prisma.user.findMany({
      where: { organizationId },
      select: { id: true }
    })
    return allUsers.map(u => u.id)
  }

  // AGENT sees only self
  if (role === 'AGENT') {
    return [userId]
  }

  // MANAGER sees self + their subtree
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

/**
 * Check if user can see/manage another user
 */
export async function canManageUser(
  managerId: string,
  managerRole: string,
  targetUserId: string,
  organizationId: string
): Promise<boolean> {
  const subtree = await getSubtreeUserIds(managerId, managerRole, organizationId)
  return subtree.includes(targetUserId)
}

/**
 * Get the manager (createdBy) of a user
 */
export async function getManager(userId: string): Promise<string | null> {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: { createdById: true }
  })
  return user?.createdById || null
}
```

#### Step 1.1: Make GET /api/users work WITH HIERARCHY

Open `apps/web/app/api/users/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
import { getSubtreeUserIds } from '@/lib/rbac/hierarchy'

export async function GET(req: NextRequest) {
  // 1. Check if user is logged in
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  // 2. Get query params for filtering
  const { searchParams } = new URL(req.url)
  const search = searchParams.get('search') || ''
  const page = parseInt(searchParams.get('page') || '1')
  const limit = parseInt(searchParams.get('limit') || '20')

  // 3. Get user IDs this user can see (HIERARCHY!)
  const visibleUserIds = await getSubtreeUserIds(
    session.user.id,
    session.user.role,
    session.user.organizationId
  )

  // 4. Fetch users from database (filtered by hierarchy!)
  const users = await prisma.user.findMany({
    where: {
      id: { in: visibleUserIds },  // HIERARCHY FILTER!
      organizationId: session.user.organizationId,
      OR: search ? [
        { name: { contains: search, mode: 'insensitive' } },
        { email: { contains: search, mode: 'insensitive' } }
      ] : undefined
    },
    skip: (page - 1) * limit,
    take: limit,
    orderBy: { createdAt: 'desc' }
  })

  return NextResponse.json({ data: users })
}
```

#### Step 1.2: Test it (IMPORTANT!)

**Test the hierarchy by logging in as different users:**

| # | Login As | Expected Result |
|---|----------|-----------------|
| 1 | Owner | See ALL users in org |
| 2 | Manager M1 | See only M1 + agents M1 created |
| 3 | Agent A1 | See only A1 (self) |

```bash
# In browser console:
GET http://localhost:3000/api/users

# Verify: Only see users you should see!
```

#### Step 1.3: Make PATCH /api/users/[id] work WITH HIERARCHY

Open `apps/web/app/api/users/[id]/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
import { canManageUser } from '@/lib/rbac/hierarchy'

export async function PATCH(
  req: NextRequest,
  { params }: { params: { id: string } }
) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  const body = await req.json()
  const { name, avatar, role } = body

  // Users can update their own profile
  const isOwnProfile = params.id === session.user.id

  // Otherwise, check hierarchy - can only update users in your subtree
  if (!isOwnProfile) {
    const canManage = await canManageUser(
      session.user.id,
      session.user.role,
      params.id,
      session.user.organizationId
    )
    if (!canManage) {
      return NextResponse.json({ error: 'Not allowed - not in your subtree' }, { status: 403 })
    }
  }

  // Only OWNER can change roles
  if (role && session.user.role !== 'OWNER') {
    return NextResponse.json({ error: 'Only owner can change roles' }, { status: 403 })
  }

  const updated = await prisma.user.update({
    where: {
      id: params.id,
      organizationId: session.user.organizationId
    },
    data: {
      ...(name && { name }),
      ...(avatar && { avatar }),
      ...(role && { role })
    }
  })

  return NextResponse.json({ data: updated })
}
```

#### Step 1.4: Make invitations work WITH createdById

Open `apps/web/app/api/invitations/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
import { randomBytes } from 'crypto'

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  // Only OWNER and MANAGER can invite
  if (session.user.role === 'AGENT') {
    return NextResponse.json({ error: 'Agents cannot invite' }, { status: 403 })
  }

  const { email, role = 'AGENT' } = await req.json()

  // Validate role - can only create users at or below your level
  if (session.user.role === 'MANAGER' && role === 'OWNER') {
    return NextResponse.json({ error: 'Managers cannot create Owners' }, { status: 403 })
  }

  // Generate secure token
  const token = randomBytes(32).toString('hex')
  const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days

  // Create invitation - IMPORTANT: Store invitedById for hierarchy!
  const invitation = await prisma.invitation.create({
    data: {
      email,
      role,
      token,
      expiresAt,
      invitedById: session.user.id,  // This becomes createdById when accepted!
      organizationId: session.user.organizationId
    }
  })

  const inviteLink = `${process.env.NEXT_PUBLIC_APP_URL}/invite/${token}`

  return NextResponse.json({
    data: invitation,
    inviteLink  // Remove in production
  }, { status: 201 })
}
```

#### Step 1.5: Make accept invitation set createdById

When user accepts invite, their `createdById` should be set to whoever invited them:

```typescript
// In POST /api/invitations/accept
const newUser = await prisma.user.create({
  data: {
    email: invitation.email,
    name: body.name,
    role: invitation.role,
    organizationId: invitation.organizationId,
    createdById: invitation.invitedById  // CRITICAL: Sets hierarchy!
  }
})
```

### How to Test Feature 1 (User Journey Based)

**Setup** (create these test users first):
1. Create Owner (owner@test.com)
2. Owner invites Manager M1 → M1.createdById = Owner.id
3. M1 invites Agent A1 → A1.createdById = M1.id

**Test Cases**:
| # | Login As | Action | Expected |
|---|----------|--------|----------|
| 1 | Owner | GET /api/users | See all users |
| 2 | M1 | GET /api/users | See M1 + A1 only |
| 3 | A1 | GET /api/users | See A1 only |
| 4 | M1 | PATCH A1's profile | Works (A1 in M1's subtree) |
| 5 | A1 | PATCH M1's profile | **Fails** (M1 not in A1's subtree) |
| 6 | M1 | Invite with role=OWNER | **Fails** (can't create owner) |
| 7 | Owner | Invite with role=OWNER | Works |

### How to know Feature 1 is done
- [ ] GET /api/users respects hierarchy (verified with 3 different users)
- [ ] PATCH /api/users/[id] only works for subtree users
- [ ] Invitations set correct createdById
- [ ] Frontend /settings/team page shows hierarchy-filtered users

---

## Feature 4: Real-time Features

### What you're building
When someone sends a message, it appears instantly for everyone viewing that ticket.

### Files you'll work on
```
apps/web/lib/socket/server.ts     # Socket.io server
apps/web/lib/socket/hooks.ts      # React hooks for frontend
apps/web/server.ts                # Custom server entry
```

### Step-by-step

#### Step 4.1: Understand the socket server

Look at `apps/web/lib/socket/server.ts`. It should already have basic setup. Make sure it has:

```typescript
import { Server } from 'socket.io'
import { createAdapter } from '@socket.io/redis-adapter'
import { createClient } from 'redis'

let io: Server | null = null

export function initSocketServer(httpServer: any) {
  if (io) return io

  io = new Server(httpServer, {
    cors: {
      origin: process.env.NEXT_PUBLIC_APP_URL,
      credentials: true
    }
  })

  // Redis adapter for scaling (multiple server instances)
  if (process.env.REDIS_URL) {
    const pubClient = createClient({ url: process.env.REDIS_URL })
    const subClient = pubClient.duplicate()
    Promise.all([pubClient.connect(), subClient.connect()]).then(() => {
      io!.adapter(createAdapter(pubClient, subClient))
    })
  }

  // Handle connections
  io.on('connection', (socket) => {
    console.log('Client connected:', socket.id)

    // Get user info from auth
    const { userId, organizationId, userName } = socket.handshake.auth

    if (!userId || !organizationId) {
      socket.disconnect()
      return
    }

    // Join organization room (for org-wide events)
    socket.join(`org:${organizationId}`)
    socket.join(`user:${userId}`)

    // Store user data on socket
    socket.data.userId = userId
    socket.data.organizationId = organizationId
    socket.data.userName = userName

    // Subscribe to a ticket's updates
    socket.on('ticket:subscribe', (ticketId: string) => {
      socket.join(`ticket:${ticketId}`)
      console.log(`${userName} subscribed to ticket:${ticketId}`)
    })

    // Unsubscribe from ticket
    socket.on('ticket:unsubscribe', (ticketId: string) => {
      socket.leave(`ticket:${ticketId}`)
    })

    // Typing indicator
    socket.on('typing:start', ({ ticketId }) => {
      socket.to(`ticket:${ticketId}`).emit('typing:indicator', {
        ticketId,
        userName,
        isTyping: true
      })
    })

    socket.on('typing:stop', ({ ticketId }) => {
      socket.to(`ticket:${ticketId}`).emit('typing:indicator', {
        ticketId,
        userName,
        isTyping: false
      })
    })

    // Handle disconnect
    socket.on('disconnect', () => {
      io!.to(`org:${organizationId}`).emit('presence:update', {
        userId,
        status: 'offline',
        lastSeenAt: new Date()
      })
    })

    // Announce user is online
    io!.to(`org:${organizationId}`).emit('presence:update', {
      userId,
      status: 'online'
    })
  })

  return io
}

// Helper functions to emit from API routes
export function getIO() {
  return io
}

export function emitToTicket(ticketId: string, event: string, data: any) {
  if (io) {
    io.to(`ticket:${ticketId}`).emit(event, data)
  }
}

export function emitToOrg(orgId: string, event: string, data: any) {
  if (io) {
    io.to(`org:${orgId}`).emit(event, data)
  }
}
```

#### Step 4.2: Emit events from API routes

When Sneha's message API creates a message, it needs to emit a socket event. Tell her to add this:

```typescript
// In POST /api/tickets/[id]/messages after creating message:
import { emitToTicket } from '@/lib/socket/server'

// After creating the message in database:
emitToTicket(ticketId, 'message:new', {
  id: newMessage.id,
  content: newMessage.content,
  sender: { id: user.id, name: user.name },
  createdAt: newMessage.createdAt
})
```

#### Step 4.3: Create frontend hook

Check `apps/web/lib/socket/hooks.ts`:

```typescript
'use client'
import { useEffect, useState, useCallback } from 'react'
import { io, Socket } from 'socket.io-client'
import { useSession } from 'next-auth/react'

let socket: Socket | null = null

export function useSocket() {
  const { data: session } = useSession()
  const [isConnected, setIsConnected] = useState(false)

  useEffect(() => {
    if (!session?.user) return

    // Connect to socket server
    socket = io(process.env.NEXT_PUBLIC_APP_URL!, {
      auth: {
        userId: session.user.id,
        organizationId: session.user.organizationId,
        userName: session.user.name
      }
    })

    socket.on('connect', () => {
      setIsConnected(true)
      console.log('Socket connected')
    })

    socket.on('disconnect', () => {
      setIsConnected(false)
    })

    return () => {
      socket?.disconnect()
    }
  }, [session])

  return { socket, isConnected }
}

// Hook for subscribing to ticket updates
export function useTicketSubscription(ticketId: string) {
  const { socket } = useSocket()
  const [messages, setMessages] = useState<any[]>([])

  useEffect(() => {
    if (!socket || !ticketId) return

    // Subscribe to ticket room
    socket.emit('ticket:subscribe', ticketId)

    // Listen for new messages
    socket.on('message:new', (message) => {
      setMessages(prev => [...prev, message])
    })

    return () => {
      socket.emit('ticket:unsubscribe', ticketId)
      socket.off('message:new')
    }
  }, [socket, ticketId])

  return { messages }
}
```

### How to know Feature 4 is done
- [ ] Open same ticket in 2 browser tabs
- [ ] Send message in one tab
- [ ] Message appears in other tab instantly (no refresh)
- [ ] Typing indicator shows when someone types
- [ ] Online/offline status works

---

## Feature 6: Email Channel (AI-FIRST!)

### What you're building
Customers email support@yourcompany.com → **AI tries to respond first**.
Only if AI can't resolve → ticket is created and assigned.
Agent replies → customer receives email.

### The AI-First Flow
```
Email arrives → AI Copilot tries to answer → Can't? → Create ticket → Round-robin assign
```

### Files you'll work on
```
apps/web/app/api/webhooks/email/inbound/route.ts   # Receive emails (AI-first!)
apps/web/lib/email/sendgrid.ts                      # Send emails
apps/web/lib/ai/copilot.ts                          # (Sneha's) AI resolution
```

### Step-by-step

#### Step 6.1: Create inbound webhook WITH AI-FIRST

Create `apps/web/app/api/webhooks/email/inbound/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@wrrk/database'
import { tryAIResolve } from '@/lib/ai/copilot'  // Sneha creates this
import { sendEmail } from '@/lib/email/sendgrid'
import { getNextAvailableAgent } from '@/lib/rbac/assignment'

export async function POST(req: NextRequest) {
  // SendGrid sends form data
  const formData = await req.formData()

  const from = formData.get('from') as string        // "John Doe <john@example.com>"
  const to = formData.get('to') as string            // "support@yourcompany.com"
  const subject = formData.get('subject') as string
  const text = formData.get('text') as string
  const html = formData.get('html') as string

  // Extract email from "Name <email>" format
  const emailMatch = from.match(/<(.+)>/) || [null, from]
  const senderEmail = emailMatch[1]?.trim() || from.trim()
  const senderName = from.replace(/<.+>/, '').trim() || senderEmail

  // Find which organization this email is for
  const integration = await prisma.integration.findFirst({
    where: {
      type: 'EMAIL',
      config: {
        path: ['inboundEmail'],
        equals: to.toLowerCase()
      }
    }
  })

  if (!integration) {
    console.log('No integration found for:', to)
    return NextResponse.json({ error: 'Unknown email address' }, { status: 404 })
  }

  const organizationId = integration.organizationId
  const messageContent = text || html || ''

  // Check if this is a reply to existing ticket
  const ticketMatch = subject.match(/\[#([A-Z]+-\d+)\]/)

  if (ticketMatch) {
    // Reply to existing ticket - just add message, no AI
    const ticket = await prisma.ticket.findFirst({
      where: { ticketNumber: ticketMatch[1], organizationId }
    })

    if (ticket) {
      await prisma.message.create({
        data: {
          content: messageContent,
          ticketId: ticket.id,
          senderType: 'CUSTOMER'
        }
      })
      return NextResponse.json({ success: true, ticketId: ticket.id })
    }
  }

  // NEW EMAIL: Try AI resolution FIRST!
  const aiResult = await tryAIResolve(messageContent, organizationId)

  if (aiResult.resolved) {
    // AI handled it! Send response email, no ticket created
    await sendEmail({
      to: senderEmail,
      subject: `Re: ${subject}`,
      html: aiResult.response
    })

    // Log AI resolution for analytics
    await prisma.copilotInteraction.create({
      data: {
        organizationId,
        type: 'AUTO_RESOLVE',
        prompt: messageContent,
        response: aiResult.response,
        success: true
      }
    })

    return NextResponse.json({ success: true, aiResolved: true })
  }

  // AI couldn't resolve - create ticket
  // Find or create customer
  let customer = await prisma.customer.findFirst({
    where: { email: senderEmail, organizationId }
  })

  if (!customer) {
    customer = await prisma.customer.create({
      data: {
        email: senderEmail,
        name: senderName,
        organizationId
      }
    })
  }

  // Generate ticket number
  const lastTicket = await prisma.ticket.findFirst({
    where: { organizationId },
    orderBy: { createdAt: 'desc' }
  })
  const org = await prisma.organization.findUnique({ where: { id: organizationId } })
  const prefix = org?.name?.substring(0, 4).toUpperCase() || 'TKT'
  const nextNum = lastTicket ? parseInt(lastTicket.ticketNumber.split('-')[1]) + 1 : 1
  const ticketNumber = `${prefix}-${nextNum.toString().padStart(4, '0')}`

  // Round-robin assign to available agent
  const assignee = await getNextAvailableAgent(organizationId)

  // Create ticket
  const ticket = await prisma.ticket.create({
    data: {
      ticketNumber,
      subject: subject || 'No subject',
      status: 'OPEN',
      priority: 'MEDIUM',
      channel: 'EMAIL',
      customerId: customer.id,
      assigneeId: assignee?.id,  // Auto-assigned!
      organizationId,
      metadata: { aiAttempted: true, aiConfidence: aiResult.confidence }
    }
  })

  // Add message to ticket
  await prisma.message.create({
    data: {
      content: messageContent,
      ticketId: ticket.id,
      senderType: 'CUSTOMER',
      customerId: customer.id
    }
  })

  return NextResponse.json({ success: true, ticketId: ticket.id, aiResolved: false })
}
```

#### Step 6.2: Create round-robin assignment helper

Create `apps/web/lib/rbac/assignment.ts`:

```typescript
import { prisma } from '@wrrk/database'

// Simple round-robin: Get next available agent
let lastAssignedIndex: Record<string, number> = {}

export async function getNextAvailableAgent(organizationId: string) {
  // Get all agents in org
  const agents = await prisma.user.findMany({
    where: {
      organizationId,
      role: 'AGENT'
    },
    orderBy: { createdAt: 'asc' }
  })

  if (agents.length === 0) return null

  // Round-robin
  const index = (lastAssignedIndex[organizationId] || 0) % agents.length
  lastAssignedIndex[organizationId] = index + 1

  return agents[index]
}
```

#### Step 6.2: Create email sending utility

Create `apps/web/lib/email/sendgrid.ts`:

```typescript
import sgMail from '@sendgrid/mail'

sgMail.setApiKey(process.env.SENDGRID_API_KEY!)

interface SendEmailOptions {
  to: string
  subject: string
  text?: string
  html?: string
  replyTo?: string
}

export async function sendEmail(options: SendEmailOptions) {
  const msg = {
    to: options.to,
    from: process.env.SENDGRID_FROM_EMAIL || 'support@yourdomain.com',
    subject: options.subject,
    text: options.text,
    html: options.html,
    replyTo: options.replyTo
  }

  try {
    await sgMail.send(msg)
    return { success: true }
  } catch (error) {
    console.error('Email send error:', error)
    throw error
  }
}

// Send reply to a ticket (called from message creation)
export async function sendTicketReply(
  ticket: { ticketNumber: string; subject: string },
  message: { content: string },
  customer: { email: string }
) {
  await sendEmail({
    to: customer.email,
    subject: `Re: [#${ticket.ticketNumber}] ${ticket.subject}`,
    html: message.content
  })
}
```

#### Step 6.3: Tell Sneha to call email on reply

When an agent replies to an EMAIL channel ticket, send email:

```typescript
// In Sneha's POST /api/tickets/[id]/messages
// After creating the message, if ticket channel is EMAIL:

if (ticket.channel === 'EMAIL' && senderType === 'AGENT') {
  await sendTicketReply(ticket, newMessage, ticket.customer)
}
```

### How to Test Feature 6 (AI-First Email)

**Test Cases**:
| # | Test | Expected |
|---|------|----------|
| 1 | Email simple question (e.g., "What are your hours?") | AI responds via email, NO ticket created |
| 2 | Email complex question AI can't answer | Ticket created, auto-assigned to agent |
| 3 | Agent replies to ticket | Customer receives email reply |
| 4 | Customer replies to agent email | Message added to same ticket |
| 5 | Check ticket metadata | Shows `aiAttempted: true` |

**Browser Steps**:
1. Send email to support@yourdomain.com with simple question
2. Check inbox - should get AI response (no ticket in dashboard)
3. Send another email with complex question
4. Check dashboard - ticket should appear, assigned to an agent
5. Reply as agent - customer should get email

### How to know Feature 6 is done
- [ ] Simple emails get AI response (no ticket)
- [ ] Complex emails create ticket with auto-assignment
- [ ] Agent replies send email to customer
- [ ] Customer replies add to existing ticket

---

## Feature 7: Website Chat Widget (AI-FIRST!)

### What you're building
A chat bubble that customers can embed on their website.
**AI responds first**, only creates ticket if AI can't resolve.

### The AI-First Flow
```
Customer message → AI Copilot responds → Can't resolve OR "talk to human" → Create ticket
```

### Files you'll work on
```
apps/web/app/api/widget/init/route.ts        # Widget initialization
apps/web/app/api/widget/message/route.ts     # Send message (AI-first!)
apps/web/public/widget.js                     # The embed script
```

### Step-by-step

#### Step 7.1: Create widget init endpoint

Create `apps/web/app/api/widget/init/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@wrrk/database'

export async function POST(req: NextRequest) {
  const { organizationId } = await req.json()

  // Find organization
  const org = await prisma.organization.findUnique({
    where: { id: organizationId }
  })

  if (!org) {
    return NextResponse.json({ error: 'Organization not found' }, { status: 404 })
  }

  // Check if any agents are online (simplified - you can make this smarter)
  const onlineAgents = await prisma.user.count({
    where: {
      organizationId,
      role: { in: ['AGENT', 'SUPERVISOR', 'MANAGER', 'ADMIN'] },
      // In real app, check actual online status
    }
  })

  return NextResponse.json({
    organizationName: org.name,
    agentsOnline: onlineAgents > 0,
    welcomeMessage: 'Hi! How can we help you today?'
  })
}
```

#### Step 7.2: Create message endpoint WITH AI-FIRST

Create `apps/web/app/api/widget/message/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@wrrk/database'
import { emitToTicket } from '@/lib/socket/server'
import { tryAIResolve } from '@/lib/ai/copilot'  // Sneha creates this
import { getNextAvailableAgent } from '@/lib/rbac/assignment'

export async function POST(req: NextRequest) {
  const { sessionId, organizationId, content, customerInfo } = await req.json()

  // Check for escalation keywords
  const wantsHuman = /talk to (human|agent|person)|speak to someone|real person/i.test(content)

  // Check if there's an existing ticket for this session
  let ticket = await prisma.ticket.findFirst({
    where: {
      organizationId,
      metadata: { path: ['widgetSessionId'], equals: sessionId }
    }
  })

  // If ticket exists, this is a follow-up message
  if (ticket) {
    // Add message to existing ticket
    const message = await prisma.message.create({
      data: {
        content,
        ticketId: ticket.id,
        senderType: 'CUSTOMER'
      }
    })

    emitToTicket(ticket.id, 'message:new', {
      id: message.id,
      content: message.content,
      senderType: 'CUSTOMER',
      createdAt: message.createdAt
    })

    return NextResponse.json({
      messageId: message.id,
      ticketId: ticket.id,
      type: 'ticket_message'
    })
  }

  // NEW CONVERSATION: Try AI first (unless they want human)
  if (!wantsHuman) {
    const aiResult = await tryAIResolve(content, organizationId)

    if (aiResult.resolved) {
      // AI handled it! Return response, no ticket created
      return NextResponse.json({
        type: 'ai_response',
        response: aiResult.response,
        resolved: true
      })
    }
  }

  // AI couldn't resolve OR customer wants human - create ticket
  // Find or create customer
  let customer = null
  if (customerInfo?.email) {
    customer = await prisma.customer.findFirst({
      where: { email: customerInfo.email, organizationId }
    })

    if (!customer) {
      customer = await prisma.customer.create({
        data: {
          email: customerInfo.email,
          name: customerInfo.name || customerInfo.email,
          organizationId
        }
      })
    }
  }

  // Generate ticket number
  const lastTicket = await prisma.ticket.findFirst({
    where: { organizationId },
    orderBy: { createdAt: 'desc' }
  })
  const org = await prisma.organization.findUnique({ where: { id: organizationId } })
  const prefix = org?.name?.substring(0, 4).toUpperCase() || 'TKT'
  const nextNum = lastTicket ? parseInt(lastTicket.ticketNumber.split('-')[1]) + 1 : 1
  const ticketNumber = `${prefix}-${nextNum.toString().padStart(4, '0')}`

  // Round-robin assign
  const assignee = await getNextAvailableAgent(organizationId)

  // Create ticket
  ticket = await prisma.ticket.create({
    data: {
      ticketNumber,
      subject: 'Chat conversation',
      status: 'OPEN',
      priority: 'MEDIUM',
      channel: 'CHAT',
      customerId: customer?.id,
      assigneeId: assignee?.id,
      organizationId,
      metadata: { widgetSessionId: sessionId, aiAttempted: !wantsHuman }
    }
  })

  // Add first message
  const message = await prisma.message.create({
    data: {
      content,
      ticketId: ticket.id,
      senderType: 'CUSTOMER',
      customerId: customer?.id
    }
  })

  // Emit to agents
  emitToTicket(ticket.id, 'message:new', {
    id: message.id,
    content: message.content,
    senderType: 'CUSTOMER',
    createdAt: message.createdAt
  })

  return NextResponse.json({
    type: 'escalated',
    ticketId: ticket.id,
    ticketNumber: ticket.ticketNumber,
    message: wantsHuman
      ? "I'm connecting you with a support agent. Please wait..."
      : "I'm connecting you with a support agent who can better help. Please wait..."
  })
}
```

#### Step 7.4: Create basic widget script

Create `apps/web/public/widget.js`:

```javascript
(function() {
  // Get config from script tag
  const script = document.currentScript
  const orgId = script.getAttribute('data-org-id')
  const position = script.getAttribute('data-position') || 'bottom-right'
  const primaryColor = script.getAttribute('data-primary-color') || '#4F46E5'

  const API_URL = script.src.replace('/widget.js', '')

  // Session management
  let sessionId = localStorage.getItem('wrrk_session_id')
  if (!sessionId) {
    sessionId = 'sess_' + Math.random().toString(36).substr(2, 9)
    localStorage.setItem('wrrk_session_id', sessionId)
  }

  let ticketId = localStorage.getItem('wrrk_ticket_id')
  let socket = null

  // Create widget HTML
  const widgetHTML = `
    <div id="wrrk-widget" style="
      position: fixed;
      ${position.includes('bottom') ? 'bottom: 20px' : 'top: 20px'};
      ${position.includes('right') ? 'right: 20px' : 'left: 20px'};
      z-index: 99999;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    ">
      <!-- Chat button -->
      <button id="wrrk-button" style="
        width: 60px;
        height: 60px;
        border-radius: 50%;
        background: ${primaryColor};
        border: none;
        cursor: pointer;
        box-shadow: 0 4px 12px rgba(0,0,0,0.15);
        display: flex;
        align-items: center;
        justify-content: center;
      ">
        <svg width="24" height="24" fill="white" viewBox="0 0 24 24">
          <path d="M20 2H4c-1.1 0-2 .9-2 2v18l4-4h14c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2z"/>
        </svg>
      </button>

      <!-- Chat window (hidden initially) -->
      <div id="wrrk-window" style="
        display: none;
        position: absolute;
        bottom: 70px;
        right: 0;
        width: 350px;
        height: 500px;
        background: white;
        border-radius: 12px;
        box-shadow: 0 4px 24px rgba(0,0,0,0.2);
        overflow: hidden;
        flex-direction: column;
      ">
        <!-- Header -->
        <div style="
          background: ${primaryColor};
          color: white;
          padding: 16px;
          font-weight: 600;
        ">
          Support Chat
        </div>

        <!-- Messages -->
        <div id="wrrk-messages" style="
          flex: 1;
          overflow-y: auto;
          padding: 16px;
        "></div>

        <!-- Input -->
        <div style="
          padding: 12px;
          border-top: 1px solid #eee;
          display: flex;
          gap: 8px;
        ">
          <input id="wrrk-input" type="text" placeholder="Type a message..." style="
            flex: 1;
            padding: 8px 12px;
            border: 1px solid #ddd;
            border-radius: 20px;
            outline: none;
          ">
          <button id="wrrk-send" style="
            background: ${primaryColor};
            color: white;
            border: none;
            padding: 8px 16px;
            border-radius: 20px;
            cursor: pointer;
          ">Send</button>
        </div>
      </div>
    </div>
  `

  // Add widget to page
  document.body.insertAdjacentHTML('beforeend', widgetHTML)

  // Elements
  const button = document.getElementById('wrrk-button')
  const window = document.getElementById('wrrk-window')
  const messages = document.getElementById('wrrk-messages')
  const input = document.getElementById('wrrk-input')
  const sendBtn = document.getElementById('wrrk-send')

  // Toggle window
  button.onclick = () => {
    const isOpen = window.style.display === 'flex'
    window.style.display = isOpen ? 'none' : 'flex'
    if (!isOpen) input.focus()
  }

  // Send message
  async function sendMessage() {
    const content = input.value.trim()
    if (!content) return

    // Start chat if needed
    if (!ticketId) {
      const res = await fetch(`${API_URL}/api/widget/start-chat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ sessionId, organizationId: orgId })
      })
      const data = await res.json()
      ticketId = data.ticketId
      localStorage.setItem('wrrk_ticket_id', ticketId)
    }

    // Send message
    await fetch(`${API_URL}/api/widget/message`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ ticketId, sessionId, content })
    })

    // Show message locally
    addMessage(content, 'customer')
    input.value = ''
  }

  sendBtn.onclick = sendMessage
  input.onkeypress = (e) => {
    if (e.key === 'Enter') sendMessage()
  }

  // Add message to UI
  function addMessage(content, type) {
    const div = document.createElement('div')
    div.style.cssText = `
      padding: 8px 12px;
      margin: 4px 0;
      border-radius: 12px;
      max-width: 80%;
      ${type === 'customer'
        ? `background: ${primaryColor}; color: white; margin-left: auto;`
        : 'background: #f0f0f0; color: #333;'}
    `
    div.textContent = content
    messages.appendChild(div)
    messages.scrollTop = messages.scrollHeight
  }

  // TODO: Connect to Socket.io for real-time agent responses
  // This is a simplified version - add socket connection for production

})()
```

### How to Test Feature 7 (AI-First Widget)

**Test Cases**:
| # | Test | Expected |
|---|------|----------|
| 1 | Ask simple question in widget | AI responds, NO ticket created |
| 2 | Say "talk to human" | Ticket created, assigned to agent |
| 3 | Ask complex question AI can't answer | Ticket created, assigned to agent |
| 4 | Agent responds in dashboard | Message appears in widget |
| 5 | Customer sends follow-up | Message added to same ticket |

**Browser Steps**:
1. Create test HTML with widget embed
2. Open test page, click chat bubble
3. Ask "What are your hours?" → AI should respond
4. Type "talk to human" → should see escalation message
5. Check dashboard → ticket should exist with your messages
6. Reply as agent → customer should see response in widget

### How to know Feature 7 is done
- [ ] Widget appears on test page
- [ ] Simple questions get AI response (no ticket)
- [ ] "Talk to human" creates ticket
- [ ] Complex questions create ticket
- [ ] Agent replies appear in widget in real-time

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
import { hasPermission } from '@/lib/rbac/permissions'
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
```

### Testing with Postman
1. First login at http://localhost:3000/login
2. Copy the session cookie
3. Add cookie to Postman requests

Good luck Suraj! Ask Samarth if you're stuck.

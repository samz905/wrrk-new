# Suraj's Action Plan

This is your guide for building 4 features. Work through them in order. 

---

## Your Features

| # | Feature | Priority |
|---|---------|----------|
| 1 | User & Org Management | Do first |
| 4 | Real-time Features | Do second |
| 6 | Email Channel | Do third |
| 7 | Website Chat Widget | Do fourth |

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

## Feature 1: User & Org Management

### What you're building
Users can log in, update their profile, and invite teammates.

### Files you'll work on
```
apps/web/app/api/users/route.ts           # List users
apps/web/app/api/users/[id]/route.ts      # Get/update single user
apps/web/app/api/organizations/[id]/route.ts  # Org settings
apps/web/app/api/teams/route.ts           # List/create teams
apps/web/app/api/teams/[id]/route.ts      # Update/delete team
apps/web/app/api/teams/[id]/members/route.ts  # Add/remove members
apps/web/app/api/invitations/route.ts     # Send invitations
```

### Step-by-step

#### Step 1.1: Make GET /api/users work

Open `apps/web/app/api/users/route.ts` and make sure it:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'

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

  // 3. Fetch users from database (only from user's organization!)
  const users = await prisma.user.findMany({
    where: {
      organizationId: session.user.organizationId,  // IMPORTANT: Always filter by org
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

#### Step 1.2: Test it
```bash
# In browser console or Postman:
GET http://localhost:3000/api/users

# Should return list of users in your org
```

#### Step 1.3: Make PATCH /api/users/[id] work

Open `apps/web/app/api/users/[id]/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
import { hasPermission } from '@/lib/rbac/permissions'

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
  // Only ADMIN/MANAGER can update others or change roles
  const isOwnProfile = params.id === session.user.id
  const canUpdateOthers = hasPermission(session.user.role, 'users:update')

  if (!isOwnProfile && !canUpdateOthers) {
    return NextResponse.json({ error: 'Not allowed' }, { status: 403 })
  }

  // Only ADMIN can change roles
  if (role && session.user.role !== 'ADMIN') {
    return NextResponse.json({ error: 'Only admin can change roles' }, { status: 403 })
  }

  const updated = await prisma.user.update({
    where: {
      id: params.id,
      organizationId: session.user.organizationId  // Security: only update users in same org
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

#### Step 1.4: Make invitations work

Open `apps/web/app/api/invitations/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { getServerSession } from 'next-auth'
import { prisma } from '@wrrk/database'
import { authOptions } from '@/lib/auth/config'
import { hasPermission } from '@/lib/rbac/permissions'
import { randomBytes } from 'crypto'

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions)
  if (!session?.user) {
    return NextResponse.json({ error: 'Not logged in' }, { status: 401 })
  }

  // Only ADMIN/MANAGER can invite
  if (!hasPermission(session.user.role, 'invitations:create')) {
    return NextResponse.json({ error: 'Not allowed' }, { status: 403 })
  }

  const { email, role = 'AGENT' } = await req.json()

  // Generate secure token
  const token = randomBytes(32).toString('hex')
  const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days

  // Create invitation
  const invitation = await prisma.invitation.create({
    data: {
      email,
      role,
      token,
      expiresAt,
      invitedById: session.user.id,
      organizationId: session.user.organizationId
    }
  })

  // TODO: Send email with invitation link
  // For now, just return the token (Samarth will help with email)
  const inviteLink = `${process.env.NEXT_PUBLIC_APP_URL}/invite/${token}`

  return NextResponse.json({
    data: invitation,
    inviteLink  // Remove this in production, just for testing
  }, { status: 201 })
}
```

### How to know Feature 1 is done
- [ ] Can call GET /api/users and see user list
- [ ] Can call PATCH /api/users/[id] to update profile
- [ ] Can call POST /api/invitations to create invite
- [ ] Frontend /settings/profile page loads and saves changes
- [ ] Frontend /settings/team page shows team members

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

## Feature 6: Email Channel

### What you're building
Customers email support@yourcompany.com → ticket is created.
Agent replies → customer receives email.

### Files you'll work on
```
apps/web/app/api/webhooks/email/inbound/route.ts   # Receive emails
apps/web/lib/email/sendgrid.ts                      # Send emails
```

### Step-by-step

#### Step 6.1: Create inbound webhook

Create `apps/web/app/api/webhooks/email/inbound/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@wrrk/database'

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
  // (Based on the 'to' address - you'll configure this in settings)
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

  // Check if this is a reply to existing ticket
  // Look for ticket number in subject like "Re: [#ACME-0042] Your request"
  const ticketMatch = subject.match(/\[#([A-Z]+-\d+)\]/)
  let ticket

  if (ticketMatch) {
    // Find existing ticket
    ticket = await prisma.ticket.findFirst({
      where: {
        ticketNumber: ticketMatch[1],
        organizationId
      }
    })
  }

  if (!ticket) {
    // Create new ticket
    const lastTicket = await prisma.ticket.findFirst({
      where: { organizationId },
      orderBy: { createdAt: 'desc' }
    })

    const org = await prisma.organization.findUnique({
      where: { id: organizationId }
    })
    const prefix = org?.name?.substring(0, 4).toUpperCase() || 'TKT'
    const nextNum = lastTicket
      ? parseInt(lastTicket.ticketNumber.split('-')[1]) + 1
      : 1
    const ticketNumber = `${prefix}-${nextNum.toString().padStart(4, '0')}`

    ticket = await prisma.ticket.create({
      data: {
        ticketNumber,
        subject: subject || 'No subject',
        status: 'OPEN',
        priority: 'MEDIUM',
        channel: 'EMAIL',
        customerId: customer.id,
        organizationId
      }
    })
  }

  // Add message to ticket
  await prisma.message.create({
    data: {
      content: text || html || '',
      ticketId: ticket.id,
      senderType: 'CUSTOMER',
      customerId: customer.id
    }
  })

  // Emit socket event so agents see it
  // (Import from your socket server)
  // emitToOrg(organizationId, 'ticket:updated', { ticketId: ticket.id })

  return NextResponse.json({ success: true, ticketId: ticket.id })
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

### How to know Feature 6 is done
- [ ] Send email to configured support address
- [ ] Ticket appears in dashboard
- [ ] Reply to ticket from dashboard
- [ ] Customer receives email reply
- [ ] Customer replies → message added to same ticket

---

## Feature 7: Website Chat Widget

### What you're building
A chat bubble that customers can embed on their website.

### Files you'll work on
```
apps/web/app/api/widget/init/route.ts        # Widget initialization
apps/web/app/api/widget/start-chat/route.ts  # Start new chat
apps/web/app/api/widget/message/route.ts     # Send message
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

#### Step 7.2: Create start-chat endpoint

Create `apps/web/app/api/widget/start-chat/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@wrrk/database'

export async function POST(req: NextRequest) {
  const { sessionId, organizationId, customerInfo } = await req.json()

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

  const org = await prisma.organization.findUnique({
    where: { id: organizationId }
  })
  const prefix = org?.name?.substring(0, 4).toUpperCase() || 'TKT'
  const nextNum = lastTicket
    ? parseInt(lastTicket.ticketNumber.split('-')[1]) + 1
    : 1
  const ticketNumber = `${prefix}-${nextNum.toString().padStart(4, '0')}`

  // Create ticket
  const ticket = await prisma.ticket.create({
    data: {
      ticketNumber,
      subject: 'Chat conversation',
      status: 'OPEN',
      priority: 'MEDIUM',
      channel: 'CHAT',
      customerId: customer?.id,
      organizationId,
      metadata: { widgetSessionId: sessionId }
    }
  })

  return NextResponse.json({
    ticketId: ticket.id,
    ticketNumber: ticket.ticketNumber,
    customerId: customer?.id
  })
}
```

#### Step 7.3: Create message endpoint

Create `apps/web/app/api/widget/message/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@wrrk/database'
import { emitToTicket } from '@/lib/socket/server'

export async function POST(req: NextRequest) {
  const { ticketId, content, sessionId } = await req.json()

  // Verify ticket exists and matches session
  const ticket = await prisma.ticket.findFirst({
    where: {
      id: ticketId,
      metadata: {
        path: ['widgetSessionId'],
        equals: sessionId
      }
    }
  })

  if (!ticket) {
    return NextResponse.json({ error: 'Ticket not found' }, { status: 404 })
  }

  // Create message
  const message = await prisma.message.create({
    data: {
      content,
      ticketId,
      senderType: 'CUSTOMER',
      customerId: ticket.customerId
    }
  })

  // Emit for real-time
  emitToTicket(ticketId, 'message:new', {
    id: message.id,
    content: message.content,
    senderType: 'CUSTOMER',
    createdAt: message.createdAt
  })

  return NextResponse.json({ messageId: message.id })
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

### How to know Feature 7 is done
- [ ] Create test HTML file with widget embed
- [ ] Widget bubble appears
- [ ] Click opens chat window
- [ ] Send message creates ticket in dashboard
- [ ] Agent reply appears in widget (need socket integration)

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

# wrrk.ai MVP Development Plan

## Team & Ownership

| Owner | Features | Responsibility |
|-------|----------|----------------|
| **Suraj** | 1, 4, 6, 7 | User/Org, Real-time, Email, Chat Widget |
| **Sneha** | 2, 3, 5 | Ticketing, Messaging, AI Copilot |
| **Samarth** | Supervisor | Infrastructure, code review, deployment |

**Key Principle**: Each owner ensures their feature works **end-to-end** (API + Frontend). You own the full vertical slice.

---

## User Journeys

These journeys describe how users interact with the system. Use them to understand and test features.

### Journey 1: Owner Onboarding
```
1. Owner signs up → organization created
2. Owner goes to Settings > Team
3. Owner clicks "Add Manager"
4. Enters email, role=MANAGER
5. Manager receives invite, joins
6. Manager now sees empty team (only themselves)
```

### Journey 2: Manager Building Team
```
1. Manager logs in
2. Goes to Settings > Team (sees only self)
3. Clicks "Add Agent"
4. Agent joins via invite
5. Manager now sees themselves + agent
6. Manager can create Teams for grouping
```

### Journey 3: Customer → AI → Agent
```
1. Customer opens chat widget
2. Types question
3. AI Copilot responds with suggestion
4. Customer says "talk to human"
5. Ticket created, round-robin to Agent
6. Agent sees ticket in their queue
7. Agent responds
8. Customer gets response in widget
```

### Journey 4: Agent Escalates to Manager
```
1. Agent handling difficult ticket
2. Clicks "Escalate"
3. System shows their Manager (not other agents)
4. Agent transfers to Manager
5. Manager sees escalated ticket
6. Manager can reassign or handle
```

### Journey 5: Transfer Between Agents
```
1. Agent A needs to transfer to Agent B
2. Agent A clicks "Transfer"
3. System shows Manager (not other agents directly)
4. Agent A transfers to Manager
5. Manager reassigns to Agent B
```

---

## Feature Ownership

### Suraj's Features

| # | Feature | What it does |
|---|---------|--------------|
| 1 | User & Org Management | Users can log in, manage profile, invite teammates |
| 4 | Real-time Features | Live updates, presence, typing indicators |
| 6 | Email Channel | Emails create tickets, replies sync back |
| 7 | Website Chat Widget | Customers chat via embedded widget |

### Sneha's Features

| # | Feature | What it does |
|---|---------|--------------|
| 2 | Ticketing System | Create, assign, update, close tickets |
| 3 | Messaging | Send messages in tickets, attach files |
| 5 | AI Copilot | AI suggests responses, analyzes sentiment |

### Samarth's Responsibilities

- Set up Supabase, Upstash Redis, AWS Bedrock, SendGrid
- Review all PRs before merge
- Help unblock any issues
- Final production deployment

---

## Feature Details

### Feature 1: User & Org Management (Suraj)

**Goal**: Users can sign in, see their profile, manage team members with **hierarchical visibility**.

**Hierarchy Rules** (CRITICAL):
- Roles: `OWNER > MANAGER > AGENT`
- Users can only see/manage users they created (via `createdById`)
- OWNER sees everyone, MANAGER sees their subtree, AGENT sees only self
- See `TECH_ARCH.md` for `getSubtreeUserIds()` helper

**API Endpoints**:
- `GET /api/users` - List users (filtered by hierarchy!)
- `GET/PATCH /api/users/[id]` - Get/update user
- `GET/PATCH /api/organizations/[id]` - Get/update org settings
- `GET/POST /api/teams` - List/create teams
- `POST/DELETE /api/teams/[id]/members` - Add/remove team members
- `POST /api/invitations` - Send invite (sets `createdById` to inviter)
- `POST /api/invitations/accept` - Accept invitation

**Frontend Pages**:
- `/settings/profile` - User profile page
- `/settings/team` - Team management page (shows hierarchy-filtered users)

**Done when**:
- [ ] User can log in and see dashboard
- [ ] User can update their profile
- [ ] Owner/Manager can invite new users via email
- [ ] Invited user accepts and joins with correct `createdById`
- [ ] User list respects hierarchy (test: Manager only sees their agents)

**How to Test**:
| # | As | Do | Expect |
|---|----|----|--------|
| 1 | Owner | List users | See all users in org |
| 2 | Manager M1 | List users | See only M1 + agents M1 created |
| 3 | Agent A1 | List users | See only A1 (self) |
| 4 | Owner | Create Manager M1 | M1.createdById = Owner.id |
| 5 | M1 | Create Agent A1 | A1.createdById = M1.id |

---

### Feature 2: Ticketing System (Sneha)

**Goal**: Agents can create, view, assign, and manage support tickets with **hierarchical visibility**.

**Visibility Rules** (CRITICAL):
- OWNER: Sees all tickets in org
- MANAGER: Sees tickets assigned to users in their subtree
- AGENT: Sees only their assigned tickets
- Use `getSubtreeUserIds()` to filter tickets by `assigneeId`

**API Endpoints**:
- `GET /api/tickets` - List tickets (filtered by hierarchy!)
- `POST /api/tickets` - Create ticket (auto-generates number, auto-assigns via round-robin)
- `GET /api/tickets/[id]` - Get ticket details (check visibility!)
- `PATCH /api/tickets/[id]` - Update status/priority/assignee
- `GET /api/customers` - List customers
- `POST /api/customers` - Create customer
- `GET/PATCH /api/customers/[id]` - Get/update customer

**Assignment Rules**:
- OWNER: Can assign to anyone
- MANAGER: Can assign only to their subtree
- AGENT: Cannot assign - must escalate to Manager

**Frontend Pages**:
- `/tickets` - Ticket list with filters (hierarchy-filtered)
- `/tickets/[id]` - Single ticket view
- `/customers` - Customer list
- `/customers/[id]` - Customer profile

**Done when**:
- [ ] Can create a new ticket with customer
- [ ] Ticket list shows with status filters
- [ ] Can assign ticket (respecting hierarchy rules)
- [ ] Can change ticket status (Open → In Progress → Resolved → Closed)
- [ ] Ticket visibility respects hierarchy

**How to Test**:
| # | As | Do | Expect |
|---|----|----|--------|
| 1 | Owner | List tickets | See all tickets in org |
| 2 | Manager M1 | List tickets | See tickets assigned to M1's subtree |
| 3 | Agent A1 | List tickets | See only A1's assigned tickets |
| 4 | Agent A1 | Try to assign ticket | Only see "Escalate to Manager" option |
| 5 | M1 | Assign ticket | Can only assign to agents in M1's subtree |

---

### Feature 3: Messaging (Sneha)

**Goal**: Agents and customers can exchange messages within tickets.

**API Endpoints**:
- `GET /api/tickets/[id]/messages` - List messages
- `POST /api/tickets/[id]/messages` - Send message
- `POST /api/upload` - Upload file attachment

**Frontend Components**:
- Message list in ticket view
- Message composer with send button
- File attachment upload

**Done when**:
- [ ] Messages display in ticket view
- [ ] Can send new message
- [ ] Can attach files to messages
- [ ] Messages show sender name and timestamp

---

### Feature 4: Real-time Features (Suraj)

**Goal**: Updates appear instantly without page refresh.

**Socket.io Events**:
- `message:new` - New message arrives
- `ticket:updated` - Ticket status/assignment changes
- `typing:indicator` - Someone is typing
- `presence:update` - User online/offline status

**Frontend Integration**:
- Connect to Socket.io on app load
- Subscribe to ticket room when viewing ticket
- Show typing indicator
- Show online/offline badges

**Done when**:
- [ ] Open ticket in 2 tabs, send message in one → appears in other
- [ ] Typing indicator shows when someone types
- [ ] Online status shows for team members
- [ ] Ticket updates reflect immediately

---

### Feature 5: AI Copilot (Sneha)

**Goal**: AI is the **FIRST point of contact** for customers, escalates to humans when needed.

**AI-First Flow** (CRITICAL):
```
Customer Message → AI Tries to Resolve → Can't? → Create Ticket → Round-Robin Assign
```
- AI attempts to answer from knowledge base FIRST
- Only creates ticket if AI can't resolve OR customer asks for human
- See `TECH_ARCH.md` for full flow diagram

**API Endpoints**:
- `POST /api/copilot/suggest` - Get AI response suggestion (for agents)
- `POST /api/copilot/resolve` - Try to auto-resolve customer query (for widget/email)
- `POST /api/copilot/categorize` - Auto-categorize ticket
- (Auto) Sentiment analysis on new customer messages

**Frontend Integration**:
- "Suggest Response" button in ticket view
- Display AI suggestion, allow edit before sending
- Show sentiment indicator (positive/neutral/negative)
- Widget shows AI response first before escalating

**Done when**:
- [ ] Click "Suggest Response" → AI drafts a reply
- [ ] Agent can edit and send the suggestion
- [ ] Customer messages show sentiment badge
- [ ] New tickets get auto-categorized
- [ ] Widget/Email: AI responds first, escalates if needed

**How to Test**:
| # | As | Do | Expect |
|---|----|----|--------|
| 1 | Agent | Click "Suggest Response" | AI drafts reply based on conversation |
| 2 | Agent | Edit AI suggestion, send | Message sent with edited content |
| 3 | Customer (widget) | Ask simple question | AI responds without creating ticket |
| 4 | Customer (widget) | Say "talk to human" | Ticket created, assigned to agent |
| 5 | Customer (email) | Send question | AI tries first, escalates if needed |

---

### Feature 6: Email Channel (Suraj)

**Goal**: Customers can email support, **AI handles first**, agents reply from dashboard.

**API Endpoints**:
- `POST /api/webhooks/email/inbound` - Receive incoming emails (AI-first!)
- Email sending happens in message creation endpoint

**Setup Required**:
- SendGrid account with inbound parse
- Webhook URL configured in SendGrid

**Flow** (AI-First):
1. Customer emails `support@yourdomain.com`
2. SendGrid forwards to webhook
3. **AI Copilot tries to resolve first**
4. If AI can't resolve → creates ticket (or adds to existing)
5. Round-robin assigns to agent
6. Agent sees ticket, replies
7. Reply sent as email to customer

**Done when**:
- [ ] Send email to support address → AI tries to respond
- [ ] AI can't resolve → ticket appears
- [ ] Reply to ticket → customer receives email
- [ ] Customer replies → message added to same ticket

**How to Test**:
| # | As | Do | Expect |
|---|----|----|--------|
| 1 | Customer | Email simple question | AI responds via email |
| 2 | Customer | Email complex question | Ticket created, assigned |
| 3 | Agent | Reply to ticket | Customer receives email reply |
| 4 | Customer | Reply to agent's email | Message added to ticket |

---

### Feature 7: Website Chat Widget (Suraj)

**Goal**: Customers can chat live from any website, **AI handles first**.

**API Endpoints**:
- `POST /api/widget/init` - Initialize widget, check if agents online
- `POST /api/widget/message` - Send message (AI-first processing!)

**Widget Bundle**:
- `public/widget.js` - Embeddable script

**Flow** (AI-First):
1. Customer visits website with widget embedded
2. Clicks chat bubble, enters name/email (optional)
3. Sends message → **AI Copilot responds first**
4. If AI can't resolve OR customer asks for human:
   - Creates ticket (channel: CHAT)
   - Round-robin assigns to agent
5. Agent sees ticket, responds
6. Response appears in widget instantly

**Done when**:
- [ ] Widget appears on test page
- [ ] Customer message → AI responds first
- [ ] AI can't resolve → ticket created
- [ ] Messages sync in real-time
- [ ] Agent can respond from dashboard

**How to Test**:
| # | As | Do | Expect |
|---|----|----|--------|
| 1 | Customer | Ask simple question in widget | AI responds, no ticket created |
| 2 | Customer | Say "talk to human" | Ticket created, assigned to agent |
| 3 | Customer | Ask complex question | AI can't resolve, ticket created |
| 4 | Agent | Respond in dashboard | Message appears in widget instantly |

---

## Development Timeline

### Days 1-2: Foundation
- **All**: Environment setup, database, auth testing
- **Suraj**: Features 1 (User/Org APIs)
- **Sneha**: Features 2 & 3 (Tickets, Messaging APIs)

### Days 3-4: Integrations
- **Suraj**: Features 4, 6, 7 (Real-time, Email, Widget)
- **Sneha**: Feature 5 (AI Copilot)

### Day 5: Testing & Deploy
- **All**: End-to-end testing
- **Samarth**: Production deployment

---

## Testing Checklist

### Suraj Tests
- [ ] Login works with test credentials
- [ ] Can update user profile
- [ ] Can invite new user
- [ ] Real-time messages work (2 tabs test)
- [ ] Email creates ticket
- [ ] Widget chat creates ticket

### Sneha Tests
- [ ] Can create ticket with customer
- [ ] Ticket filters work (status, priority)
- [ ] Can send messages with attachments
- [ ] AI suggestions work
- [ ] Sentiment shows on messages

### Samarth Tests
- [ ] All environment variables set
- [ ] Database accessible
- [ ] Redis connected
- [ ] AWS Bedrock responding
- [ ] SendGrid webhook receiving

---

## Schema Updates Required

**IMPORTANT**: Devs must update the Prisma schema themselves. Key changes needed:

```prisma
// Change Role enum from 4 roles to 3
enum Role {
  OWNER     // Was: ADMIN
  MANAGER   // Was: MANAGER + SUPERVISOR combined
  AGENT     // Same
}

// Add hierarchy tracking to User
model User {
  createdById   String?
  createdBy     User?    @relation("CreatedUsers", fields: [createdById], references: [id])
  createdUsers  User[]   @relation("CreatedUsers")
}
```

After updating: `pnpm db:push` to sync with database.

---

## Definition of Done

MVP is complete when all these work:

- [ ] User can sign up / log in
- [ ] Hierarchy works (Owner > Manager > Agent visibility)
- [ ] Tickets can be created, assigned (respecting hierarchy)
- [ ] Messages appear in real-time
- [ ] AI is first point of contact (widget/email)
- [ ] AI can suggest responses for agents
- [ ] Emails create tickets (after AI can't resolve)
- [ ] Website chat widget works (AI-first)
- [ ] No critical bugs in production

---

## Resources

- [TECH_ARCH.md](./TECH_ARCH.md) - Technical architecture
- [SURAJ_ACTION_PLAN.md](./SURAJ_ACTION_PLAN.md) - Suraj's detailed guide
- [SNEHA_ACTION_PLAN.md](./SNEHA_ACTION_PLAN.md) - Sneha's detailed guide
- [Prisma Schema](./wrrk.ai/packages/database/prisma/schema.prisma) - Database models

# wrrk.ai MVP Development Plan

## Team & Ownership

| Owner | Features | Responsibility |
|-------|----------|----------------|
| **Suraj** | 1, 4, 6, 7 | User/Org, Real-time, Email, Chat Widget |
| **Sneha** | 2, 3, 5 | Ticketing, Messaging, AI Copilot |
| **Samarth** | Supervisor | Infrastructure, code review, deployment |

**Key Principle**: Each owner ensures their feature works **end-to-end** (API + Frontend). You own the full vertical slice.

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

**Goal**: Users can sign in, see their profile, manage team members, send invitations.

**API Endpoints**:
- `GET /api/users` - List users in org
- `GET/PATCH /api/users/[id]` - Get/update user
- `GET/PATCH /api/organizations/[id]` - Get/update org settings
- `GET/POST /api/teams` - List/create teams
- `POST/DELETE /api/teams/[id]/members` - Add/remove team members
- `POST /api/invitations` - Send invite email
- `POST /api/invitations/accept` - Accept invitation

**Frontend Pages**:
- `/settings/profile` - User profile page
- `/settings/team` - Team management page

**Done when**:
- [ ] User can log in and see dashboard
- [ ] User can update their profile
- [ ] Admin can invite new users via email
- [ ] Invited user can accept and join org

---

### Feature 2: Ticketing System (Sneha)

**Goal**: Agents can create, view, assign, and manage support tickets.

**API Endpoints**:
- `GET /api/tickets` - List tickets with filters
- `POST /api/tickets` - Create ticket (auto-generates number)
- `GET /api/tickets/[id]` - Get ticket details
- `PATCH /api/tickets/[id]` - Update status/priority/assignee
- `GET /api/customers` - List customers
- `POST /api/customers` - Create customer
- `GET/PATCH /api/customers/[id]` - Get/update customer

**Frontend Pages**:
- `/tickets` - Ticket list with filters
- `/tickets/[id]` - Single ticket view
- `/customers` - Customer list
- `/customers/[id]` - Customer profile

**Done when**:
- [ ] Can create a new ticket with customer
- [ ] Ticket list shows with status filters
- [ ] Can assign ticket to an agent
- [ ] Can change ticket status (Open → In Progress → Resolved → Closed)

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

**Goal**: AI helps agents respond faster and smarter.

**API Endpoints**:
- `POST /api/copilot/suggest` - Get AI response suggestion
- `POST /api/copilot/categorize` - Auto-categorize ticket
- (Auto) Sentiment analysis on new customer messages

**Frontend Integration**:
- "Suggest Response" button in ticket view
- Display AI suggestion, allow edit before sending
- Show sentiment indicator (positive/neutral/negative)

**Done when**:
- [ ] Click "Suggest Response" → AI drafts a reply
- [ ] Agent can edit and send the suggestion
- [ ] Customer messages show sentiment badge
- [ ] New tickets get auto-categorized

---

### Feature 6: Email Channel (Suraj)

**Goal**: Customers can email support, agents reply from dashboard.

**API Endpoints**:
- `POST /api/webhooks/email/inbound` - Receive incoming emails
- Email sending happens in message creation endpoint

**Setup Required**:
- SendGrid account with inbound parse
- Webhook URL configured in SendGrid

**Flow**:
1. Customer emails `support@yourdomain.com`
2. SendGrid forwards to webhook
3. Webhook creates ticket (or adds to existing)
4. Agent sees ticket, replies
5. Reply sent as email to customer

**Done when**:
- [ ] Send email to support address → ticket appears
- [ ] Reply to ticket → customer receives email
- [ ] Customer replies → message added to same ticket

---

### Feature 7: Website Chat Widget (Suraj)

**Goal**: Customers can chat live from any website.

**API Endpoints**:
- `POST /api/widget/init` - Initialize widget, check if agents online
- `POST /api/widget/start-chat` - Create ticket from widget
- `POST /api/widget/message` - Send message from widget

**Widget Bundle**:
- `public/widget.js` - Embeddable script

**Flow**:
1. Customer visits website with widget embedded
2. Clicks chat bubble, enters name/email (optional)
3. Sends message → creates ticket (channel: CHAT)
4. Agent sees ticket, responds
5. Response appears in widget instantly

**Done when**:
- [ ] Widget appears on test page
- [ ] Starting chat creates a ticket
- [ ] Messages sync in real-time
- [ ] Agent can respond from dashboard

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

## Definition of Done

MVP is complete when all these work:

- [ ] User can sign up / log in
- [ ] Tickets can be created, assigned, updated
- [ ] Messages appear in real-time
- [ ] AI can suggest responses
- [ ] Emails create tickets and replies sync back
- [ ] Website chat widget works
- [ ] No critical bugs in production

---

## Resources

- [TECH_ARCH.md](./TECH_ARCH.md) - Technical architecture
- [SURAJ_ACTION_PLAN.md](./SURAJ_ACTION_PLAN.md) - Suraj's detailed guide
- [SNEHA_ACTION_PLAN.md](./SNEHA_ACTION_PLAN.md) - Sneha's detailed guide
- [Prisma Schema](./wrrk.ai/packages/database/prisma/schema.prisma) - Database models

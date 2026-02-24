<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/whatsapp.png" 
    alt="WhatsApp logo" 
    width="200"
  />
</p>

# How WhatsApp Works — Product & System Architecture

**Product:** WhatsApp
**Audience:** Product Managers / Developers / Curious Minds
**Goal:** Explain how WhatsApp delivers fast, reliable messaging at massive scale (PM-friendly), without going deep into infrastructure jargon.

**Related docs:**
- Teardown: https://github.com/004mayank/product-teardowns/blob/main/whatsapp-messaging-teardown.md
- PRD: https://github.com/004mayank/product-prd/blob/main/whatsapp-messaging-prd.md

---

## Version history
- **v2 (current):** added a PM-view system map for the PRD “conversation health” initiatives (TTFR/CCR), including instrumentation, notification decisioning, and rollout/experimentation considerations.
- **v1:** baseline explanation of WhatsApp messaging architecture, encryption, delivery, and reliability concepts.

---

## 1) The core product problem WhatsApp solves

WhatsApp’s primary job is deceptively simple:

> “Deliver messages instantly and reliably, even on poor networks, for billions of users.”

At scale, this translates into a few hard requirements:
- Extremely low latency
- High reliability
- Graceful handling of offline users
- Strong privacy guarantees (end-to-end encryption)

Everything in WhatsApp’s architecture exists to serve these needs.

---

## 2) Key product requirements

WhatsApp optimizes for:

- **Speed:** Messages should feel instant
- **Reliability:** Messages should not be lost
- **Ordering:** Messages should arrive in the right sequence
- **Offline support:** Users frequently switch networks
- **Privacy:** WhatsApp servers should not read messages

These requirements directly shape the technical design.

---

## 3) High-level architecture (conceptual)

Important PM insight:
- WhatsApp servers act as **routers and temporary storage**
- They are **not permanent message stores**
- They cannot read message content due to encryption

```
Sender Device (encrypts)
   |
   v
+--------------------------+
| WhatsApp Servers         |
| (routing + queue)        |
+--------------------------+
   |
   v
Recipient Device (decrypts)
```

---

## 4) Message flow — step by step

### Step 1: Message creation
- User types a message
- Message is **encrypted on the sender’s device**
- Encryption keys are generated and stored on devices, not servers

PM takeaway:
> Privacy is enforced at the product level, not as a backend feature.

### Step 2: Message sent to WhatsApp servers
- Encrypted message is sent to WhatsApp servers
- Server identifies the recipient and routes the message

Key point:
- Servers **cannot decrypt** or inspect message content

### Step 3: Delivery or queueing
- If recipient is online → message is delivered immediately
- If recipient is offline → message is temporarily stored

Important constraint:
- Messages are stored only **until delivery**, not forever

### Step 4: Delivery acknowledgement
- Recipient device confirms receipt
- WhatsApp servers notify sender (✓✓)

PM takeaway:
> Read receipts are a product feature built on delivery acknowledgements.

### Step 5: Message deletion from server
- Once delivered, the message is deleted from WhatsApp servers

This reduces:
- Storage cost
- Privacy risk
- Long-term liability

---

## 5) Offline & poor network handling

WhatsApp is designed for unreliable connectivity.

When a user:
- Switches networks
- Loses signal
- Turns the phone off

The system:
- Queues messages server-side
- Retries delivery when the device reconnects
- Maintains message order using timestamps and IDs

PM insight:
> Reliability is achieved through retries and acknowledgements, not through permanent storage.

---

## 6) End-to-end encryption (PM explanation)

End-to-end encryption means:
- Messages are encrypted on the sender’s device
- Only the recipient’s device can decrypt them
- WhatsApp servers only see encrypted blobs

Product implications:
- WhatsApp cannot read messages
- Features like search, moderation, or backups become harder
- Trade-off between privacy and server-side intelligence

This is a **deliberate product choice**, not a technical limitation.

---

## 7) Group messaging (conceptual difference)

In group chats:
- Sender encrypts the message **separately for each participant**
- WhatsApp servers fan out the encrypted messages

PM takeaway:
> Group size directly impacts server load and delivery complexity.

This is why:
- Extremely large groups have limits
- Group features evolve more cautiously

---

## 8) Failure scenarios & how WhatsApp handles them

### Recipient offline for long time
- Messages remain queued temporarily
- If undelivered for too long, messages may expire

### Sender changes phone
- Encryption keys change
- Old messages may not sync
- Backup behavior becomes important

### Server outage
- Messages retry automatically
- Users may see delayed delivery, not message loss

PM insight:
> WhatsApp optimizes for “eventual delivery” over perfect real-time guarantees.

---

## 9) Key trade-offs WhatsApp makes

| Trade-off | Decision |
|---|---|
| Privacy vs server intelligence | Privacy |
| Speed vs rich server features | Speed |
| Simplicity vs flexibility | Simplicity |
| Cloud storage vs device ownership | Device ownership |

These trade-offs explain many WhatsApp product decisions, including limitations users sometimes complain about.

---

## 10) Why WhatsApp feels “instant”

From a PM perspective, WhatsApp feels fast because:
- Message payloads are small
- Servers act as routers, not processors
- Delivery acknowledgements are lightweight
- No heavy server-side computation per message

Speed is a **product principle**, not an accident.

---

## 11) System addendum for the PRD: “Conversation Health” (TTFR/CCR)

The PRD proposes three lightweight product changes:
1) Chat list **“Needs reply”** highlight state
2) Intent-based **notification prompts**
3) Group **“Since you were away”** micro-summary based on explicit signals

This section maps those to a PM-view architecture.

### 11.1 What needs to be measured (instrumentation)
To improve **TTFR** and **CCR**, you need consistent definitions and events.

**Metric definitions (from teardown/PRD):**
- **Conversation start (CS):** first outbound message after ≥X hours inactivity
- **TTFR:** time from CS to first inbound reply
- **CCR (24h):** reply received within 24 hours of CS

**Minimum event taxonomy (illustrative):**
- `messaging.message_sent`
- `messaging.message_delivered`
- `messaging.message_read`
- `conversation.conversation_start`
- `conversation.reply_received`
- `ui.chatlist.needs_reply_shown`
- `ui.chatlist.needs_reply_dismissed`
- `permissions.notifications_prompt_shown`
- `permissions.notifications_enabled`
- `groups.away_summary_shown`
- `groups.mention_clicked`

PM note:
> The system should make it easy to compute TTFR/CCR by joining “start” and “reply” events per conversation.

### 11.2 “Needs reply” state — where it can live
There are two reasonable approaches:

**Option A: On-device computation (preferred for simplicity + privacy)**
- The client uses local chat state to decide “needs reply.”
- Pros: privacy-friendly; fast; no server dependency.
- Cons: cross-device consistency is harder; heuristics may differ.

**Option B: Server-assisted hints (only with minimal metadata)**
- Server can provide “unreplied incoming” hints based on message headers/acks, not content.
- Pros: consistent across devices.
- Cons: more complexity; must be careful with privacy posture.

Given WhatsApp’s product posture, a v1 would typically lean on **Option A**.

### 11.3 Notification decisioning (delivery + relevance)
TTFR often fails because notifications fail or get ignored.

A PM-view architecture to support notification improvements:
- **Permission state manager** (client): knows if notifications are allowed, muted, or restricted.
- **Rate limiting** (client + server): prevents spammy prompting.
- **Delivery receipt hooks**: leverage delivered/read state to avoid redundant nudges.

The PRD’s “intent-based prompt” is essentially:
- detect **high-intent moments** (first-send, first-join)
- show a prompt once
- measure opt-in + impact on A2/TTFR

### 11.4 “Since you were away” micro-summary (explicit signals only)
To avoid “feed-ification,” keep summaries anchored to explicit, user-visible signals:
- mentions of you
- replies to your messages
- messages pinned by admins (explicit)

Implementation shape:
- Client computes an “away session” window (last opened timestamp)
- On open, it compiles counts and jump links to mentions/replies
- Cache locally so it’s fast even on poor networks

### 11.5 Experimentation & rollout plumbing
Shipping conversation-health features safely requires:
- **Feature flags** (enable/disable quickly)
- **A/B assignment** (deterministic per user/device)
- **Guardrail monitoring** (notification disablement, blocks/reports, group mute/leave)

Rollout should be staged:
1% → 10% → 50% with clear abort conditions.

---

## 12) Takeaways
- System architecture directly shapes user experience.
- Privacy choices constrain feature design.
- Reliability is built through retries and acknowledgements.
- For “conversation health,” the highest-leverage systems are **instrumentation + notification decisioning + lightweight UI prioritization**.

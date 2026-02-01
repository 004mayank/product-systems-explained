<p align="center">
  <img 
    src="https://raw.githubusercontent.com/004mayank/product-teardowns/main/images/zomato-logo.png" 
    alt="Zomato logo" 
    width="200"
  />
</p>

# How Zomato Works - Ordering System Architecture (PM View)


**Product:** Zomato  
**Audience:** Product Managers / Developers / Foodies  
**Goal:** Explain how Zomato moves an order from discovery to delivery at scale,
focusing on product behavior, constraints, and trade-offs.


---

## 1. The Core Product Problem

Zomato must reliably convert *intent* into a *delivered meal* across:
- Millions of users
- Thousands of restaurants
- A volatile delivery supply
- Real-time payments and ETAs

From a PM lens, success = **order placed → fulfilled → delivered on time** with trust intact.

---

## 2. Key Product Requirements 

Zomato optimizes for:
- **Conversion:** Reduce drop-offs from menu → order
- **Reliability:** Orders should complete once placed
- **Speed:** Accurate ETAs, fast confirmation
- **Coordination:** Restaurant + delivery partner sync
- **Trust:** Clear pricing, predictable outcomes

These requirements drive the system design.

---

## 3. High-Level Ordering Architecture (Conceptual)

**Flow:**
- User App  
- Zomato Platform *(Pricing, Orders, Payments)*  
- Restaurant ↔ Delivery Partner  

**PM Insight:**  
Zomato acts as a **coordinator**, not the executor of cooking or delivery.

---

## 4. Ordering Flow - Step by Step

### Step 1: Discovery & Menu
- User browses restaurants and menus
- Prices, ratings, ETA estimates shown

**PM takeaway:** Early expectations are set here—errors ripple downstream.

---

### Step 2: Cart & Pricing
- Items added to cart
- Platform fees, delivery fees, offers applied

**Key risk:** Late price revelation → checkout abandonment  
(Directly tied to your teardown + PRD.)

---

### Step 3: Order Placement & Payment
- User confirms order
- Payment is authorized (not yet settled in all cases)

**PM insight:** This is the *point of no return*—failures here hurt trust most.

---

### Step 4: Restaurant Acceptance
- Order sent to restaurant
- Restaurant accepts or rejects (capacity, stock)

**Failure mode:** Rejection → refund + user disappointment  
**Design choice:** Fast rejection > delayed cancellation.

---

### Step 5: Delivery Partner Assignment
- System finds an available delivery partner
- ETA recalculated based on distance and load

**Trade-off:** Speed of assignment vs delivery quality.

---

### Step 6: Preparation, Pickup & Delivery
- Restaurant prepares food
- Partner picks up and delivers
- Live tracking + notifications

**PM insight:** Transparency reduces anxiety even when delays happen.

---

## 5. Payments & Refunds (Conceptual)


- Payments are often **authorized early**
- Final settlement depends on order completion
- Refunds are critical trust moments

---

## 6. Failure Scenarios & Handling

### Restaurant Rejects Order
- Immediate user notification
- Auto-refund
- Offer compensation (sometimes)

### Delivery Partner Unavailable
- Reassignment
- ETA update
- Cancellation if SLA breached

### Delays After Acceptance
- ETA recalculations
- Proactive notifications
- Support intervention if needed

**PM insight:** Clear communication often matters more than speed.

---

## 7. Key Trade-offs Zomato Makes

| Trade-off | Decision |
|---------|----------|
| Speed vs Accuracy (ETA) | Balance |
| AOV vs Conversion | Conversion-first |
| Choice vs Overload | Heavy curation |
| Automation vs Human Ops | Hybrid |

These explain many UX and policy decisions users notice.

---

## 8. Why Zomato Feels “Reliable” (When It Works)

- Clear order states (placed, accepted, picked up)
- Real-time status updates
- Fast failure handling with refunds
- Strong operational playbooks

Reliability is a **system + ops** outcome, not just tech.

---

## 9. PM Takeaways

- Zomato’s hardest problem is **coordination**, not discovery
- Late surprises cause the biggest trust loss
- Systems must optimize for *graceful failure*
- Clear expectations beat aggressive optimization

Understanding this architecture helps PMs:
- Choose the right metrics
- Prioritize where to reduce friction
- Communicate trade-offs clearly to stakeholders


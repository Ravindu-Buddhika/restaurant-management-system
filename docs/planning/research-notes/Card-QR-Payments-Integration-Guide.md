# Card & QR Payments Integration — POS System Guide

This guide explains how a restaurant POS (Point of Sale) system connects to card machines and QR payment methods. It covers the integration types, the webhook (bank notification) process, and the software architecture used to support many different card machine brands at once.

---

## 1. Card Payments Integration

There are **2 main ways** a card machine can connect to a POS system:

### A. Integrated POS Terminal (Automated)

**How it works (step by step):**
```
POS System calculates the bill amount
        │
        ▼
System auto-sends the amount to the Card Machine (EDC)
        │
        ▼
Card Machine communicates directly with the Bank
        │
        ▼
Bank sends back a Success Response / Approval Code
        │
        ▼
POS System automatically marks the bill as PAID
```

**Real-world example:** A modern restaurant has a card machine that's electronically wired/paired to the POS software. When the cashier presses "Pay by Card," the exact bill amount (say Rs. 2,430) is sent straight to the card machine — the cashier never types the amount manually. Once the customer taps their card and the bank approves it, the POS screen updates to "PAID" on its own.

**Pros:**
- No risk of the cashier mistyping the amount
- Very fast, smooth checkout experience

**Cons:**
- Technically more complex to set up — the POS software and the card machine hardware/software need to be properly integrated (wired together at a software level), which usually needs a developer or a partnership with the bank/terminal provider

### B. Non-Integrated Terminal (Manual)

**How it works (step by step):**
```
Cashier looks at the bill total on the POS screen
        │
        ▼
Cashier manually types the amount into the Card Machine
        │
        ▼
Customer taps/inserts/swipes their card
        │
        ▼
Card Machine prints a Slip with an Approval Code (Reference No.)
        │
        ▼
Cashier reads that Approval Code and types it into the POS
        │
        ▼
Cashier clicks "Mark as Paid" in the POS
```

**Real-world example:** A smaller restaurant has a basic card machine that isn't connected to the POS software at all — it's a completely separate device, like the type many small shops use. The cashier sees "Rs. 2,430" on the POS screen, then manually types "2430" into the card machine themselves. After the customer pays, a small receipt slip prints out with a reference number on it (e.g., "APR-88214"). The cashier reads that number off the slip and types it into the POS so there's a record of which card transaction paid for this specific bill.

**Pros:**
- Very simple and cheap to set up — no technical integration needed at all, any card machine works

**Cons:**
- Higher chance of human error — the cashier could type the wrong amount into the card machine, or type the wrong reference number into the POS

---

## 2. QR Payments Integration (LankaQR & App Payments)

There are **2 ways** QR payments connect to a POS system:

### A. Dynamic QR Code (Shown on the System Screen / Customer Display)

This is a QR code that changes every time — a fresh one is generated for each individual order, showing the exact amount owed.

**How it works:**
```
POS generates a Dynamic QR code for this specific order's exact amount
        │
        ▼
QR code is shown on the customer-facing screen
        │
        ▼
Customer scans it using their banking/payment app and pays
        │
        ▼
Bank's Payment Gateway sends a Notification (Webhook) to the POS Backend
        │
        ▼
POS Backend receives it and refreshes the screen automatically
   (using WebSockets or SSE — explained below)
        │
        ▼
POS UI shows "Payment Success" — no one had to click anything
```

**Real-world example:** A café has a small screen facing the customer at checkout. When the bill is finalized at Rs. 1,850, a unique QR code appears on that screen showing exactly Rs. 1,850. The customer opens their banking app, scans it, and pays. Within a second or two — without the cashier doing anything — the POS screen automatically flips to "Payment Received," because the bank told the POS system directly that the payment went through.

*(What are WebSockets / SSE? These are two different technical methods that let a server "push" updates to a screen instantly, instead of the screen having to keep asking "is it paid yet? is it paid yet?" every few seconds. Think of it like getting a push notification on your phone instead of having to manually refresh an app.)*

### B. Static QR Code (Printed Counter QR)

This is a **single, unchanging QR code** — usually a printed sticker stuck on the counter — that doesn't know the specific bill amount in advance.

**How it works:**
```
Customer scans the printed QR sticker at the counter
        │
        ▼
Customer manually types in the amount themselves (since the QR doesn't know it)
        │
        ▼
Customer completes the payment on their end
        │
        ▼
Cashier checks the Bank's SMS alert or App notification to confirm payment arrived
        │
        ▼
Cashier manually marks the bill as Paid in the POS
```

**Real-world example:** A small food stall has one printed QR sticker taped near the register — the same one used for every single customer, every day. A customer scans it, types "Rs. 450" themselves into their banking app, and pays. The stall owner's phone buzzes with an SMS: "You have received Rs. 450." The owner glances at the phone, confirms it matches the bill, and manually taps "Paid" on the POS.

**Comparing the two QR methods:**

| | Dynamic QR | Static QR |
|---|---|---|
| Amount | Automatically set by the system | Customer types it manually |
| Confirmation | Automatic (via webhook) | Manual (cashier checks SMS/notification) |
| Setup cost | Needs bank gateway integration | Just a printed sticker — nearly free |
| Error risk | Very low | Higher (customer could type wrong amount) |

---

## 3. Webhook (API Callback) — How the Bank Tells the POS "Payment Received"

A **webhook** is simply a way for one system (the bank) to automatically notify another system (the POS) the moment something happens — in this case, a successful payment — without the POS having to constantly ask "did it happen yet?"

### The Flow

```
[Customer's Mobile Banking App]
        │
        │ (1) Customer pays via the Dynamic QR code
        ▼
[Bank's Gateway Server]
        │
        │ (2) Bank sends an HTTP POST request (the "Webhook")
        │     to a specific web address the POS exposes
        ▼
[POS Backend — Webhook API Endpoint]
        │
        │ (3) POS updates its database (marks bill as PAID)
        │     and triggers a WebSocket message
        ▼
[POS Frontend / Screen] ──> Auto-updates to show "Payment Success"
```

**Real-world analogy:** Think of a webhook like a courier company automatically texting you "Your package has been delivered" the moment it's dropped off — rather than you having to call the courier every 5 minutes asking "is it delivered yet?"

### Two Important Safety Rules for Webhooks

**1. Security (Signature Verification)**
When the bank sends this "payment successful" message, how does the POS know it's genuinely from the bank, and not from someone pretending to be the bank (a scammer trying to trick the system into marking an unpaid bill as paid)?

The answer: the bank attaches a **signature hash** — a special coded fingerprint — to the message. The POS backend checks this fingerprint against what it expects from the real bank before trusting the message. If the fingerprint doesn't match, the POS ignores the request.

**2. Idempotency (Avoiding Duplicate Records)**
Sometimes, due to unstable internet connections, the bank's server might accidentally send the *same* "payment successful" webhook twice (a network retry). If the POS isn't careful, it might record the payment twice — making it look like the customer paid double.

The fix: the POS checks the **Order ID** attached to the webhook. If a payment for that exact Order ID has already been recorded, the POS simply ignores the duplicate message instead of creating a second payment record.

---

## 4. Supporting Many Different Card Machines (Adapter Pattern Architecture)

### The Problem
Restaurants don't all use the same card machine. One restaurant might use an **Ingenico** machine, another a **Verifone**, another a **Pax** — and each of these can connect in different technical ways: through a **Serial Port** (a physical USB/cable connection), a **LAN IP address** (over the restaurant's network), or a **Local HTTP API** (a local web connection).

If the POS software were written to talk to only *one specific brand* of card machine, it would break the moment a restaurant switched to a different brand. So the system needs to be built in a flexible way that can plug in almost any card machine without rewriting the whole payment system each time.

### The Solution: Adapter Pattern
This is a common software design approach where you build one common "plug shape" (called an **interface**) that every card machine brand has to fit into — similar to how a universal travel adapter lets many different plug types work with the same wall socket.

### Architecture Diagram

```
[POS Billing Core Logic]
          │
          ▼
  ┌────────────────┐
  │  CardTerminal   │  <--- This is the "universal socket" —
  │   Interface     │       an Abstract Interface that every
  └───────┬─────────┘       card machine adapter must follow
          │
  ┌───────┼───────────────────┬───────────────────┐
  ▼       ▼                   ▼                   ▼
[HTTP REST Adapter]   [Serial Port Adapter]   [TCP Socket Adapter]
  │                     │                       │
  ▼                     ▼                       ▼
(Local HTTP/API)      (USB / RS-232 COM)      (LAN / Wi-Fi IP)
```

**In plain English:** The main billing logic of the POS doesn't need to know or care *how* a specific card machine communicates. It just calls a standard set of commands (like "connect" and "process this payment"), and then a specific **Adapter** — a small piece of code written just for that one brand/connection method — handles the messy technical details of actually talking to that particular machine.

### The Code Blueprint (Explained Simply)

```typescript
// 1. The Common Interface — every card machine "adapter" must promise
//    to have these two functions, no matter which brand it is
export interface CardTerminalAdapter {
  connect(config: Record<string, any>): Promise<boolean>;
  processPayment(request: { billId: string; amount: number }): Promise<{
    success: boolean;
    approvalCode?: string;
    errorMessage?: string;
  }>;
}

// 2. The Factory — a helper that picks the RIGHT adapter automatically,
//    based on what type of connection this restaurant's machine uses
export class CardTerminalFactory {
  static getAdapter(type: 'HTTP_REST' | 'SERIAL_PORT'): CardTerminalAdapter {
    if (type === 'HTTP_REST') return new GenericHttpTerminalAdapter();
    if (type === 'SERIAL_PORT') return new SerialPortTerminalAdapter();
    throw new Error('Unsupported Terminal Type');
  }
}
```

**Real-world example:** Imagine the same POS software is sold to 3 different restaurants:
- Restaurant A uses an Ingenico machine connected over Wi-Fi → the system automatically loads the **TCP Socket Adapter**
- Restaurant B uses a Verifone machine plugged in via USB cable → the system loads the **Serial Port Adapter**
- Restaurant C uses a Pax machine that talks over a local web API → the system loads the **HTTP REST Adapter**

All three restaurants use the exact same POS "core" software — only the small adapter piece changes behind the scenes. This means the company doesn't need to write 3 separate versions of their whole POS system.

---

## Key Best Practices for Developers

| Practice | What It Means | Why It Matters |
|---|---|---|
| **Audit Logging** | When using the Non-Integrated (Manual) method, always save the Approval Code / Auth Code the cashier types in, into the `payments.transaction_reference` field in the database | At the end of the day, the restaurant needs to cross-check every card payment against the bank's official settlement report. Without a saved reference number, there's no way to prove a specific transaction happened |
| **Fallback Mode** | If an Integrated Card Machine loses its connection, the system must NOT get stuck/blocked. It should offer a "Switch to Manual Entry" option in the UI | A broken connection shouldn't stop the restaurant from taking payments — cashiers need a backup way to record a card payment manually until the connection is fixed |
| **Asynchronous Waiting** | While waiting for a response from the card machine (which can take 30–60 seconds), the POS screen must not freeze. It should show a spinner and stay responsive | If the whole screen locks up while waiting, the cashier can't do anything else — including handling a different customer or cancelling a stuck transaction. A non-blocking wait keeps the system usable |

---

## Summary Cheat Sheet

| Method | Amount Entry | Confirmation | Best For |
|---|---|---|---|
| Integrated Card Terminal | Automatic | Automatic | Larger restaurants wanting speed & accuracy |
| Non-Integrated Card Terminal | Manual | Manual (reference code) | Smaller setups, low budget |
| Dynamic QR | Automatic | Automatic (via Webhook) | Restaurants with a customer-facing screen |
| Static QR | Manual (by customer) | Manual (cashier checks SMS) | Very small stalls/counters, near-zero setup cost |

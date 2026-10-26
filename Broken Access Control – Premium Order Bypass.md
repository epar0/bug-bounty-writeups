# From UI Restriction Bypass to Unauthorized Premium Order Issuance

- **Target:** `https://redacted.com/`
- **Vulnerable Endpoint:** `/endpoint/?productId={productId}` & `/endpoint/createTicket`
- **Vulnerable Parameter:** `productId` (Premium Order)
- **Severity:** High (Broken Access Control / Business Logic Flaw)
- **Outcome:** Bug Bounty Reward & Authorization Remediation

---

## Introduction

Hidden UI elements and client-side access controls provide a false sense of security. If authorization is enforced solely through the presentation layer while backend endpoints remain wide open, unprivileged users can easily bypass restrictions.

This writeup details a privilege escalation and business logic flaw discovered in an online banking portal, where basic tier users could directly invoke backend endpoints to initiate the order of an exclusive Premium Orders reserved strictly for Premium customers.

---

## The Discovery & The Flaw

### UI Hiding vs. Server-Side Authorization
While reviewing the application's card ordering functionality, I noticed that the option to order the Premium service was completely hidden from the user interface for basic account holders. However, security relies on server-side checks, not interface obscurity.

Analysis of the endpoints revealed that card products were referenced using simple sequential/predictable integer IDs (e.g., `productId=1`), making it trivial to enumerate restricted product tiers.

By directly navigating to requesting the specific product ID (`productId=1`) designated for the Premium tier, the application's lack of proper early-stage eligibility validation became apparent:

```text
https://redacted.com/endpoint?productId=1
```

### The Request & Backend Response
Submitting the subsequent ticket creation request for the card (`createTicket`) as a basic user did not trigger a permission error or rejection. Instead, the backend progressed straight into a regulated signing flow:

```http
POST /endpoint/createTicket HTTP/1.1
Host: redacted.com

HTTP/1.1 302 Found
Location: https://redacted.com/endpoint/signing
```

The server responded with an HTTP `302 Found` redirect pointing directly to the signing module, proving that the application was actively processing the order for an ineligible user rather than rejecting it.

---

## Note on Scope & Responsible Testing

The bug bounty program rules explicitly prohibited finalizing and completing physical product orders. To maintain strict compliance with program scope and avoid generating unauthorized backend fulfillment records, the testing was halted immediately upon receiving the `302 Found` response indicating progress into the signing module.

---

## Impact

* **Business Logic Risk:** Basic users could initiate orders for financial products restricted to high-net-worth / Premium segments.

---

## Remediation

To properly secure the card application lifecycle, the engineering team implemented strict server-side validation:

1. **Early Eligibility Enforcement:** Added server-side checks during the initial request phase to verify that the user's account segment matches the requested `productId`.
2. **Access Control Alignment:** Ensured that UI restrictions are fully mirrored by backend API guards, rejecting unauthorized requests immediately rather than forwarding them into the signing pipeline.

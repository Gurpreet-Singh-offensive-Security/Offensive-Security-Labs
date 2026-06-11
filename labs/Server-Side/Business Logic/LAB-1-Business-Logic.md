# Lab 1: Business Logic Vulnerabilities — Excessive Trust in Client-Side Controls

## Executive Summary

This lab demonstrates a critical business logic vulnerability where the application places unconditional trust in price data submitted by the client rather than enforcing pricing server-side. The application allows users to add products to their shopping cart via a POST request that includes the product price as a client-controlled parameter. Because the server accepts whatever price value is submitted without validating it against a server-side price database, an attacker can intercept the add-to-cart request and modify the price to any arbitrary value before forwarding it. A leather jacket priced at $1,337.00 — far exceeding the available $100.00 account credit — was purchased for $1.00 by modifying the price parameter in Burp Repeater. The order was confirmed and the item dispatched, with $99.99 credit remaining. This attack requires no authentication bypass, no injection, and no encoding — only the ability to modify an HTTP request parameter before it reaches the server. It is one of the most fundamentally broken trust models an e-commerce application can implement.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | Business Logic Vulnerabilities |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | Excessive Trust in Client-Side Controls — Price Manipulation via Parameter Tampering |
| **Risk**               | Critical - Arbitrary Price Manipulation, Financial Loss, Inventory Theft |
| **Completion**         | June 10, 2026 |

## Objective

Exploit a business logic vulnerability in the shopping cart functionality by intercepting the add-to-cart POST request and modifying the client-submitted product price to $1.00, bypassing the $100.00 account credit limitation to purchase a $1,337.00 leather jacket and complete the order.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and full application reconnaissance
- Burp Repeater for request modification and price parameter tampering

**Target:** `price` parameter in the POST /cart request — client-submitted product price accepted and trusted by the server without validation against a server-side price authority

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "Excessive trust in client-side controls" with Burp Suite Professional configured and actively intercepting traffic, with captured requests visible in the Proxy Intercept panel.

<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/f26f31c6-34e8-4ecd-a89d-2d23b271ae71" />

*Lab 1 Business Logic Vulnerabilities environment displaying the excessive trust in client-side controls challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with captured requests visible in the Proxy Intercept panel (left)*

### 2. Full Application Reconnaissance

Conducted complete application enumeration with Burp Proxy active, browsing all available sections — product catalog, product detail pages, shopping cart, checkout, and account management — to capture the full request inventory before attempting any manipulation. Logged into the provided account and confirmed the available credit balance.
<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/ff11984f-9ad0-419b-89f5-197a9924abbf" />

*Complete application reconnaissance captured in Burp HTTP history (left) — all product, cart, login, and account requests recorded | Account dashboard visible after login (right) — available credit confirmed at $100.00, far below the $1,337.00 price of the target leather jacket*

**Account State:**
- Available credit: **$100.00**
- Target product: **Lightweight "l33t" Leather Jacket — $1,337.00**
- Shortfall: **$1,237.00**

**Reconnaissance Observations:**
- Product prices displayed in the storefront are rendered client-side from values returned by the server
- Shopping cart operations use POST requests that include product identifiers and price data
- No visible client-side validation prevents adding high-value items to the cart
- The credit check appears to occur only at checkout — not when adding items to the cart

### 3. Purchase Attempt — Insufficient Funds Confirmed

Attempted to purchase the leather jacket through the normal application flow to confirm the credit check mechanism and observe where and how it is enforced.

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/742462e9-8d67-4322-b333-a84b913d5c0f" />

*Leather jacket added to cart and checkout attempted — "Not enough credit" error displayed in the application (right) | HTTP history showing the failed checkout request with insufficient funds response (left) — confirms the credit check is enforced at checkout, not at the add-to-cart stage*

**Response:** Checkout rejected — insufficient funds.

**Analysis:** The application enforces the credit check at the checkout stage. The product was added to the cart successfully — the restriction only triggers when attempting to complete the purchase. More importantly, the add-to-cart request was captured in HTTP history and reveals whether the price is submitted as a client-controlled parameter. Examining that request is the next step.

### 4. Add-to-Cart Request Analysis — Price Parameter Identified

Located the POST /cart request generated when the leather jacket was added to the cart. Forwarded it to Burp Repeater and sent it unmodified to confirm baseline behavior.

<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/d3aa11d7-50c5-4919-9d8b-505652d71315" />


*POST /cart request sent in Repeater with original parameters — 302 Found response confirms the item was added to the cart successfully. The request body is visible and reveals the price parameter submitted by the client alongside the product identifier*

**Request Captured:**
```http
POST /cart HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION]

productId=1&redir=PRODUCT&quantity=1&price=133700
```

**Response:** 302 Found — item added to cart.

**Critical Finding — Price Is a Client-Submitted Parameter:**

The `price=133700` parameter in the POST body is submitted by the client and represents the product price in cents ($1,337.00 = 133700 cents). The server accepts this value and uses it to calculate the cart total rather than looking up the authoritative price from its own database. This is the vulnerability: the client is telling the server what the product costs, and the server believes it.

**Why This Should Not Exist:**
In a correctly implemented e-commerce system, the client submits only a product identifier and quantity. The server looks up the current price from its own price database, calculates the total, and never accepts price data from the client. Price is a server-side concern — it should never be a parameter the client can supply or modify.

### 5. Price Parameter Manipulation

Modified the `price` parameter in Repeater from `133700` (representing $1,337.00) to `100` (representing $1.00) and sent the modified request.

**Modified Request:**
```http
POST /cart HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION]

productId=1&redir=PRODUCT&quantity=1&price=100
```

**Price Modification:**

| Field | Original Value | Modified Value | Represented Price |
|-------|---------------|----------------|-------------------|
| `price` | `133700` | `100` | $1,337.00 → $1.00 |


<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/8cc18bf6-7544-4622-84c0-5ae278b323c4" />

*Price parameter modified from `133700` to `100` in Repeater — 302 Found response confirms the server accepted the modified price and added the leather jacket to the cart at $1.00. The server performed no price validation against its own database*

**Response:** 302 Found — item added to cart at the manipulated price.

**Analysis:** The server accepted `price=100` without any validation. No error, no rejection, no comparison against the authoritative product price. The leather jacket is now in the cart at a price of $1.00 — within the $100.00 available credit. The checkout can now proceed.

### 6. Order Confirmed — Purchase Completed

Proceeded through the checkout flow using the normal application interface. The cart total reflected the manipulated price of $1.00. Submitted the order.

<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/f48e7d88-a351-49be-ae68-e11cdf85a246" />

*Checkout completed successfully — order confirmation showing "Congratulations! Your order is on its way!" displayed in the application (right) | HTTP history showing the checkout POST with `orderConfirmed=true` in the response (left) — $99.99 credit remaining after the $1.00 charge, confirming the $1,337.00 leather jacket was purchased for $1.00*

**Order Result:**
- **Item purchased:** Lightweight "l33t" Leather Jacket
- **Actual price:** $1,337.00
- **Price paid:** $1.00
- **Credit remaining:** $99.99
- **Order status:** Confirmed — item dispatched

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:H
```

**Primary Vulnerability:**

**CWE-602: Client-Side Enforcement of Server-Side Security**
- Product price submitted as a client-controlled HTTP parameter
- Server accepts and processes client-supplied price without validation against an authoritative server-side source
- Any authenticated user can purchase any product at any price by modifying the `price` parameter

**CWE-20: Improper Input Validation**
- No server-side validation that the submitted price matches the actual product price
- No range check, no sanity check, no cross-reference against the product catalog
- Negative prices, zero prices, and arbitrarily low prices all accepted

**Complete Attack Chain:**

```
Full Application Reconnaissance Completed
        ↓
Account Credit Confirmed: $100.00
        ↓
Leather Jacket ($1,337.00) Added to Cart — Checkout Fails (Insufficient Funds)
        ↓
POST /cart Request Captured: price=133700 Parameter Identified
        ↓
Request Forwarded to Repeater — Baseline 302 Found Confirmed
        ↓
price=133700 Modified to price=100 ($1.00)
        ↓
Modified Request Sent — Server Returns 302 Found
        ↓
Server Accepts Client-Supplied Price — No Validation Performed
        ↓
Cart Total Reflects $1.00 — Within Available Credit
        ↓
Checkout Submitted — Order Confirmed
        ↓
$1,337.00 Jacket Purchased for $1.00 — $99.99 Credit Remaining
```

**Business Impact in a Real-World Context:**

| Impact Category | Consequence |
|----------------|-------------|
| **Direct Financial Loss** | Products sold at attacker-controlled prices — potentially $0.01 or negative values |
| **Inventory Depletion** | High-value items obtained for negligible cost at scale |
| **Revenue Manipulation** | Automated scripts can drain inventory across all product categories |
| **Competitor Abuse** | Attackers purchase all stock of popular items at $0.01 to resell |
| **Fraud at Scale** | No technical barrier to purchasing thousands of items below cost |
| **Regulatory Exposure** | Financial integrity failures may trigger compliance and audit consequences |

**Variants of This Attack:**

Price parameter manipulation is one of several client-side trust vulnerabilities in e-commerce applications. The same trust model failure manifests in related forms:

| Variant | Manipulated Parameter | Effect |
|---------|----------------------|--------|
| Price manipulation (this lab) | `price` in add-to-cart | Purchase at arbitrary price |
| Quantity sign flip | `quantity=-1` | Receive refund instead of charge |
| Discount code stacking | Reuse of single-use codes | Unlimited discount application |
| Shipping cost removal | `shipping=0` | Free shipping on any order |
| Tax exemption | `tax=0` | Purchase without tax calculation |
| Currency confusion | `currency=JPY` price in USD | Pay ¥133700 (~$890) for $1,337.00 item |

All of these share the root cause: the server trusting client-supplied financial parameters.

**How a Developer Should Fix This Vulnerability:**

**Root Cause Fix — Never Accept Price from the Client:**

**Vulnerable Pattern:**
```python
# CRITICALLY UNSAFE — price comes from the client request
def add_to_cart(session, product_id, quantity, price):
    # price is whatever the client sent — completely untrusted
    cart_item = {
        "product_id": product_id,
        "quantity": quantity,
        "price": price          # Client controls this value
    }
    db.cart.insert(session["user_id"], cart_item)
```

**Secure Pattern — Server Looks Up Price Authoritatively:**
```python
# SECURE — price is always retrieved from the server-side database
def add_to_cart(session, product_id, quantity):
    # Client supplies only product_id and quantity — never price
    product = db.products.get(product_id)

    if not product:
        return 404

    if not product.in_stock or product.stock < quantity:
        return 400  # Out of stock

    # Price comes exclusively from the server-side product record
    cart_item = {
        "product_id": product_id,
        "quantity": quantity,
        "unit_price": product.price    # Server authority — client has no input here
    }
    db.cart.insert(session["user_id"], cart_item)
```

**Secure Checkout Validation:**
```python
def checkout(session):
    cart_items = db.cart.get_by_user(session["user_id"])
    total = 0

    for item in cart_items:
        # Re-fetch current price at checkout — never use stored cart price
        product = db.products.get(item["product_id"])
        line_total = product.price * item["quantity"]
        total += line_total

    user_credit = db.users.get_credit(session["user_id"])

    if total > user_credit:
        return {"error": "Insufficient funds"}

    # Process order using server-calculated total
    process_order(session["user_id"], total)
```

**Key Principles:**
- The client submits `productId` and `quantity` only — price is never a client parameter
- Price is looked up from the product database at add-to-cart time and re-validated at checkout
- The checkout total is always recalculated server-side from current product prices — stored cart prices are never trusted
- Any discrepancy between stored cart price and current product price is caught at checkout

**Defense-in-Depth Measures:**

| Control | What It Prevents |
|---------|-----------------|
| Server-side price lookup | Client price manipulation — root cause fix |
| Price re-validation at checkout | Cart tampering between add-to-cart and checkout |
| Order total audit logging | Detection of anomalous order values post-hoc |
| Rate limiting on cart operations | Automated price-scanning attacks |
| Anomaly detection on order values | Alert on orders significantly below catalog price |
| Integrity hash on cart contents | Detect tampered cart sessions |

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Complete application reconnaissance before targeting any specific parameter — the full request inventory reveals where price data appears in the HTTP flow
- Attempt the normal purchase flow first to understand where and how restrictions are enforced before attempting bypass
- Identify client-controlled parameters that carry business-critical values — price, quantity, discount, shipping — as primary manipulation targets
- Modify a single parameter at a time to isolate the effect and maintain clean evidence
- Confirm success through the application's own order confirmation flow, not just the HTTP response

**Critical Security Insights:**

**1. Anything the Client Can See, the Client Can Modify:**
Every value rendered in a web page, stored in a hidden form field, embedded in a JavaScript variable, or transmitted in an HTTP request parameter is under the client's control. Price data displayed in a product page is not protected simply because it was loaded from the server — once it reaches the browser, the user can modify it before it is sent back. Server-side systems must never trust values that have passed through the client.

**2. Business Logic Vulnerabilities Require No Technical Sophistication:**
This attack required no injection, no encoding, no timing, and no special tooling beyond the ability to modify an HTTP parameter. Burp Repeater is sufficient. The vulnerability is architectural — the entire attack is a single parameter change. Business logic vulnerabilities are often more impactful than technical vulnerabilities precisely because they are so simple to exploit and so easy to miss in security reviews that focus on injection and authentication.

**3. Client-Side Validation Is Not Validation:**
JavaScript price calculations, hidden form fields, and client-side cart total computations are user interface conveniences, not security controls. They can be bypassed trivially by any user with basic browser developer tools or a proxy. All financial calculations, price lookups, and business rule enforcement must be performed server-side using data the server controls.

**4. The Parameter Name Is Always a Signal:**
Seeing `price=133700` in a POST request body is an immediate red flag. Price is a business-critical value that should never be a client-supplied parameter. Any parameter name that carries financial meaning — `price`, `amount`, `total`, `cost`, `discount`, `tax`, `shipping` — in an HTTP request is a manipulation candidate and should be tested in every e-commerce assessment.

**5. Reconnaissance Precedes Exploitation:**
The complete application enumeration in SS2 is what revealed the POST /cart request structure and identified the `price` parameter as client-controlled. Jumping directly to the cart request without understanding the full application flow would risk missing context — for example, whether price validation occurs at a different endpoint, whether the cart uses session storage rather than POST parameters, or whether additional parameters affect pricing. Full reconnaissance before exploitation is methodology, not overhead.

## References

1. PortSwigger Web Security Academy - Business Logic Vulnerabilities
2. PortSwigger Web Security Academy - Excessive Trust in Client-Side Controls
3. OWASP Top 10 (2021) - A04: Insecure Design
4. CWE-602 - Client-Side Enforcement of Server-Side Security
5. CWE-20 - Improper Input Validation
6. OWASP Testing Guide - Testing for Business Logic (OTG-BUSLOGIC)
7. OWASP Business Logic Vulnerability Cheat Sheet

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Price manipulation, unauthorized purchases, or exploitation of business logic vulnerabilities against live e-commerce systems without explicit written authorization constitutes fraud and is illegal under applicable laws worldwide.

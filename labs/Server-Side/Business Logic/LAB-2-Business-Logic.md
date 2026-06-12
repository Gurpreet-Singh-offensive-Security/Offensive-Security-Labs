# Lab 2: Business Logic Vulnerabilities — High Level Logic Vulnerability

## Executive Summary

This lab demonstrates exploitation of a high-level business logic vulnerability where the application fails to validate that product quantities must be positive integers. The cart system accepts negative quantity values, which reduces the cart total rather than increasing it. By combining a legitimate item — the $1,337.00 leather jacket at quantity 1 — with a second product at a deeply negative quantity, the negative line total of the second product offsets the cost of the jacket, reducing the overall cart total to an amount within the available $100.00 credit. The application's only financial guard is a check preventing the cart total from going below zero, but it does not prevent the total from being artificially reduced to any value above zero. By calculating the precise negative quantity of a second product needed to bring the leather jacket's effective cost within the available credit, I purchased the $1,337.00 jacket for $16.72 — a reduction of $1,320.28 achieved entirely through quantity sign manipulation with no technical injection or encoding required.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | Business Logic Vulnerabilities |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | High-Level Logic Vulnerability — Negative Quantity Accepted in Cart |
| **Risk**               | Critical - Arbitrary Price Reduction via Quantity Sign Manipulation |
| **Completion**         | June 11, 2026 |

## Objective

Exploit a business logic vulnerability in the shopping cart by setting a second product's quantity to a calculated negative value, causing its negative line total to offset the cost of the target leather jacket. Reduce the overall cart total to within the available $100.00 credit and complete the purchase of the $1,337.00 leather jacket at a fraction of its actual price.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and full application reconnaissance
- Burp Repeater for quantity parameter manipulation across both cart items

**Target:** `quantity` parameter in the POST /cart request — negative integer values accepted by the server, allowing line totals to subtract from rather than add to the cart total

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "High level logic vulnerability" with Burp Suite Professional configured and actively intercepting traffic in the Proxy Intercept panel.

<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/f937c097-c6dd-4219-a4d5-96fd69a5cc2b" />

*Lab 2 Business Logic Vulnerabilities environment displaying the high level logic vulnerability challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with proxy active (left)*

### 2. Full Application Reconnaissance — Pattern Identification

Conducted complete application enumeration with Burp Proxy active, capturing all requests. Logged in with provided credentials, added the leather jacket to the cart, and attempted checkout to map the full purchase flow and identify where restrictions are enforced.

<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/5b72455d-d8a2-401b-9b77-326d5a1546e3" />

*HTTP history showing the complete purchase flow with key requests highlighted in green (left) — POST /login, POST /cart (leather jacket added), and the failed checkout request all captured and identifiable | Application showing "Not enough store credit" with the leather jacket at quantity 1, price $1,337.00 (right)*

**Captured Request Sequence:**
- `POST /login` — authentication with provided credentials
- `POST /cart` — leather jacket (productId=1) added at quantity 1
- `POST /cart/checkout` — rejected: insufficient funds

**Account State:**
- Available credit: **$100.00**
- Leather jacket price: **$1,337.00**
- Shortfall: **$1,237.00**

**Reconnaissance Observations:**
- Cart accepts a `quantity` parameter in the POST /cart request
- The checkout credit check is enforced at order submission — not at add-to-cart
- No visible client-side or server-side restriction on the quantity value range
- A `NEGATIVE_TOTAL` guard exists — confirmed during testing (detailed in step 5)
- The guard prevents total below zero but does not prevent total from being manipulated above zero

### 3. Quantity Parameter Baseline — Positive Modification Confirmed

Forwarded the POST /cart request for the leather jacket to Burp Repeater and sent it unmodified to confirm the baseline behavior. The cart reflected the quantity increment in the application.

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/56f2b930-8332-43ee-8827-8d42ececea6b" />

*POST /cart request sent in Repeater with original `quantity=1` — 302 Found response confirms the request was accepted | Application cart updated showing leather jacket quantity changed from 1 to 2, confirming the Repeater request directly modifies the live cart state*

**Request:**
```http
POST /cart HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION]

productId=1&quantity=1
```

**Response:** 302 Found — quantity incremented to 2 in the application.

**Analysis:** The cart request in Repeater directly modifies the live cart. Sending `quantity=1` incremented the jacket count by 1. This confirms the parameter is fully controllable and the server processes whatever value is submitted. The next test is whether negative values are accepted.

### 4. Negative Quantity Test — Server Accepts Negative Values

Modified the quantity to `-1` in Repeater and sent the request to test whether the server validates the sign of the quantity parameter.

**Modified Request:**
```http
POST /cart HTTP/1.1

productId=1&quantity=-1
```

<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/e7bb1d4f-1592-4b6b-9e55-2180124b7942" />

*`quantity=-1` submitted in Repeater — 302 Found response confirms the server accepted the negative value | Application cart updated — leather jacket quantity reduced from 2 back to 1, confirming negative quantities decrement the cart item count without error or rejection*

**Response:** 302 Found — quantity decremented by 1, jacket quantity returned to 1.

**Analysis:** The server accepted `quantity=-1` without error. No validation prevents negative quantity values. This is the core vulnerability — the server treats negative quantities as valid, causing line totals to become negative when a sufficiently large negative quantity is applied. A product with a negative quantity does not add to the cart total — it subtracts from it.

### 5. Cart Total Boundary — NEGATIVE_TOTAL Guard Identified

Continued testing by submitting `quantity=-2` for the leather jacket, pushing it to quantity -1. Observed the cart state and attempted checkout.

<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/217a2e1e-e60a-45ba-9cac-58199a066191" />

*Cart total driven to -$1,337.00 with leather jacket at quantity -1 — checkout rejected with "Cart total cannot be less than zero" error in the application | HTTP history showing `err=NEGATIVE_TOTAL` in the failed checkout response — confirming the application has a floor guard at zero but no guard preventing manipulation above zero*

**Cart State:**
- Leather jacket quantity: **-1**
- Cart total: **-$1,337.00**
- Checkout response: `err=NEGATIVE_TOTAL`

**Analysis — Boundary Mapping Complete:**

The application enforces exactly one financial guard: the cart total cannot go below zero. It does not enforce:
- That all quantities must be positive
- That the cart total must reflect full retail prices
- That combining positive and negative quantities to produce a reduced-but-positive total is prevented

This means the attack surface is well-defined: use a negative-quantity second product to offset the jacket's cost down to a value between $0.01 and $100.00 — above the zero floor, within the available credit. The cart total can be manipulated to any positive value as long as it does not cross zero.

**Mathematical Framework:**

```
Target:     Cart total > $0.00 AND cart total ≤ $100.00
Jacket:     productId=1, unit price $1,337.00, quantity=1
            Line total = $1,337.00

Second product: unit price P, quantity Q (negative)
                Line total = P × Q (negative value)

Required:   $1,337.00 + (P × Q) ≤ $100.00
            P × Q ≤ -$1,237.00
            Q ≤ -1237.00 / P
```

### 6. Second Product Added to Cart

Added a second product to the cart — "More Than Just Birdsong" (productId=16, unit price $50.78) — to serve as the negative-quantity offset item.

<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/27c08812-3fca-4228-b687-1ced20656622" />

*"More Than Just Birdsong" (productId=16) added to cart at quantity 1 — 302 Found response | Application cart showing total of $50.78 for the new item with productId=16 confirmed in the Repeater request body. This product's unit price ($50.78) is used to calculate the required negative quantity*

**Cart State After Addition:**
- Leather jacket (productId=1): quantity 1, $1,337.00
- More Than Just Birdsong (productId=16): quantity 1, $50.78
- Cart total: $1,387.78

**Required Negative Quantity Calculation:**

```
Unit price of productId=16: $50.78
Required offset:            at least -$1,237.00 (to bring total to ≤ $100.00)

Q ≤ -1237.00 / 50.78
Q ≤ -24.36

Minimum negative quantity: -25 (brings total to $1,337.00 - 25×$50.78 = $1,337.00 - $1,269.50 = $67.50)

Testing Q = -26:
  $1,337.00 + (-26 × $50.78) = $1,337.00 - $1,320.28 = $16.72
  $16.72 < $100.00 ✓ — within available credit
  $16.72 > $0.00  ✓ — above the zero floor guard
```

**Selected quantity: -26** — produces a cart total of $16.72, satisfying both constraints.

### 7. Negative Quantity Applied to Second Product

Modified the POST /cart request for productId=16 in Repeater, setting quantity to `-26`.

**Modified Request:**
```http
POST /cart HTTP/1.1

productId=16&quantity=-26
```
<img width="1920" height="982" alt="LAB1_ss7" src="https://github.com/user-attachments/assets/9359d23b-9a64-4555-921f-c1f72c8c2efe" />


*`quantity=-26` applied to productId=16 in Repeater — 302 Found response | Application cart showing: leather jacket quantity 1 ($1,337.00), More Than Just Birdsong quantity -26 (-$1,320.28), cart total $16.72 — within the $100.00 available credit and above the $0.00 floor*

**Response:** 302 Found — quantity accepted.

**Cart State:**

| Product | Quantity | Unit Price | Line Total |
|---------|----------|-----------|------------|
| Leather Jacket (productId=1) | 1 | $1,337.00 | $1,337.00 |
| More Than Just Birdsong (productId=16) | -26 | $50.78 | -$1,320.28 |
| **Cart Total** | | | **$16.72** |

Cart total of $16.72 is within the $100.00 available credit and above the $0.00 minimum. Order can now be placed.

### 8. Order Confirmed — Leather Jacket Purchased at $16.72

Proceeded through checkout with the manipulated cart.


<img width="1920" height="982" alt="LAB1_ss8" src="https://github.com/user-attachments/assets/0add8c2a-ff21-4aaf-aef4-2b647badeca4" />


*HTTP history showing the successful checkout with `orderConfirmed=true` in the response (left) — cart contents: leather jacket qty 1, More Than Just Birdsong qty -26, total $16.72 | Application showing order confirmation "Your order is on its way!" with store credit remaining $83.28 (right) — confirming the $1,337.00 leather jacket was purchased for $16.72*

**Order Result:**
- **Leather jacket purchased:** $1,337.00 item
- **Amount charged:** $16.72
- **Discount achieved:** $1,320.28 (98.75% reduction)
- **Credit remaining:** $83.28
- **Order status:** Confirmed — dispatched

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:H
```

**Primary Vulnerability:**

**CWE-20: Improper Input Validation — Missing Negative Value Constraint**
- `quantity` parameter accepts negative integer values without rejection
- No server-side validation enforcing quantity must be greater than zero
- Negative quantities produce negative line totals that reduce the overall cart total

**CWE-840: Business Logic Errors — Incomplete Constraint Enforcement**
- Application enforces only one financial constraint: total cannot go below zero
- Does not enforce that all quantities must be positive
- Does not enforce that cart total must reflect actual product costs
- Partial constraint enforcement leaves a wide exploitable range above the zero floor

**Complete Attack Chain:**

```
Full Application Reconnaissance — Purchase Flow Mapped
        ↓
Leather Jacket ($1,337.00) — Checkout Fails (Insufficient Funds)
        ↓
POST /cart Forwarded to Repeater — quantity=1 Baseline: 302 Found
        ↓
quantity=-1 Submitted — 302 Found — Negative Values Accepted
        ↓
quantity=-2 Applied to Jacket — Cart Total -$1,337.00
        ↓
Checkout Attempted — err=NEGATIVE_TOTAL — Floor Guard at $0.00 Identified
        ↓
Boundary Analysis: Must Keep Total > $0.00 AND ≤ $100.00
        ↓
Second Product Added: productId=16 at $50.78/unit
        ↓
Required Offset Calculated: Q ≤ -1237/50.78 → Q = -26
        ↓
quantity=-26 Applied to productId=16 — Cart Total = $16.72
        ↓
Checkout Submitted — orderConfirmed=true
        ↓
$1,337.00 Jacket Purchased for $16.72
```

**Why the NEGATIVE_TOTAL Guard Is Insufficient:**

The `err=NEGATIVE_TOTAL` response reveals that the development team was aware of the negative quantity vulnerability and implemented a partial fix — preventing the cart from going below zero. However, this guard addresses only the most obvious exploitation path (making items free by driving total to zero or below) while leaving the underlying vulnerability completely intact. An attacker who encounters this guard simply calculates how deeply negative the second product needs to go while keeping the total just above zero. The guard creates an obstacle but not a barrier.

This is a pattern common in business logic vulnerability remediation: developers fix the symptom they can observe (negative total) without addressing the root cause (negative quantities accepted). The correct fix eliminates negative quantities entirely — at which point the negative total is impossible regardless of any guard.

**Comparison — Lab 1 vs Lab 2 Attack Approach:**

| Attribute | Lab 1 (Price Manipulation) | Lab 2 (Quantity Manipulation) |
|-----------|---------------------------|------------------------------|
| Parameter targeted | `price` | `quantity` |
| Manipulation type | Replace with lower value | Set to negative integer |
| Second product required | No | Yes — for offset calculation |
| Mathematical calculation needed | No | Yes — offset calculation |
| Partial server guard present | No | Yes — NEGATIVE_TOTAL floor |
| Guard bypass required | No | Yes — keep total > $0.00 |
| Attack complexity | Minimal | Low-moderate |

**Scalability of This Attack:**

The technique is fully automatable. A script can:
1. Add the target high-value item at quantity 1
2. Enumerate available low-cost products to use as the offset item
3. Calculate the required negative quantity for each candidate
4. Select the candidate that produces a cart total closest to $0.01
5. Submit the order

At scale against a production system, an attacker could drain an entire product catalog at near-zero cost in minutes. The only constraint is the $0.00 floor — which can be satisfied to arbitrary precision by choosing the right offset product and quantity.

**How a Developer Should Fix This Vulnerability:**

**Root Cause Fix — Enforce Positive Quantity Constraint:**

**Vulnerable Pattern:**
```python
def add_to_cart(session, product_id, quantity):
    # No validation on quantity sign — accepts any integer
    product = db.products.get(product_id)
    cart_item = {
        "product_id": product_id,
        "quantity": quantity,           # -26 accepted silently
        "unit_price": product.price
    }
    db.cart.upsert(session["user_id"], cart_item)
```

**Secure Pattern — Quantity Must Be a Positive Integer:**
```python
def add_to_cart(session, product_id, quantity):
    # Enforce positive integer constraint at the earliest point
    if not isinstance(quantity, int) or quantity < 1:
        return {"error": "Quantity must be a positive integer"}, 400

    # Enforce reasonable upper bound to prevent cart stuffing
    if quantity > 99:
        return {"error": "Maximum quantity per item is 99"}, 400

    product = db.products.get(product_id)
    if not product or not product.in_stock:
        return {"error": "Product unavailable"}, 404

    cart_item = {
        "product_id": product_id,
        "quantity": quantity,
        "unit_price": product.price
    }
    db.cart.upsert(session["user_id"], cart_item)
```

**Secure Checkout Re-Validation:**
```python
def checkout(session):
    cart_items = db.cart.get_by_user(session["user_id"])

    # Re-validate all quantities and re-fetch all prices at checkout
    total = 0
    for item in cart_items:
        # Reject any cart item with a non-positive quantity
        if item["quantity"] < 1:
            return {"error": "Invalid cart state"}, 400

        product = db.products.get(item["product_id"])
        line_total = product.price * item["quantity"]

        # Sanity check — line total must be positive
        if line_total <= 0:
            return {"error": "Invalid cart state"}, 400

        total += line_total

    # Total must be positive before credit check
    if total <= 0:
        return {"error": "Invalid cart total"}, 400

    user_credit = db.users.get_credit(session["user_id"])
    if total > user_credit:
        return {"error": "Insufficient funds"}, 400

    process_order(session["user_id"], total)
```

**Defense-in-Depth Measures:**

| Control | What It Prevents |
|---------|-----------------|
| Quantity ≥ 1 validation at add-to-cart | Negative quantity submission — root cause fix |
| Quantity re-validation at checkout | Tampered cart quantities from add-to-cart to checkout |
| Line total positivity check at checkout | Any combination producing a negative line total |
| Cart total positivity check at checkout | Cart manipulation producing zero or negative totals |
| Maximum quantity cap (e.g. 99) | Quantity overflow attacks and cart stuffing |
| Order value anomaly monitoring | Detect orders significantly below expected price range |

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Full application reconnaissance to map the complete purchase flow before attempting any manipulation
- Systematic boundary testing: positive quantity first to confirm control, then negative to test sign validation, then zero-crossing to identify the floor guard
- Treat a partial server guard (`err=NEGATIVE_TOTAL`) as a signal that the developer was aware of the issue but implemented an incomplete fix — the root cause is still present
- Mathematical precision in offset calculation: knowing the unit price of the second product, calculate the exact negative quantity needed to satisfy both the credit ceiling ($100.00) and the floor guard ($0.00)
- Use the application's own cart display to verify the manipulated state before placing the order

**Critical Security Insights:**

**1. Partial Fixes Are More Dangerous Than No Fix:**
The `NEGATIVE_TOTAL` guard gave a false impression of security. An auditor seeing this guard might conclude the negative quantity vulnerability had been mitigated. In practice, the guard only narrows the attack parameter — it does not eliminate the vulnerability. Security fixes must address the root cause, not just the most obvious symptom. After any fix is implemented, the original attack vector and all variants should be retested to confirm the root cause is resolved.

**2. Business Logic Bugs Require Business Logic Understanding:**
This attack required understanding that cart totals are additive across line items, that line items with negative quantities produce negative totals, and that the overall total is the sum of all line items. The exploit is not a technical trick — it is arithmetic applied to the application's own data model. Finding and exploiting business logic vulnerabilities requires thinking like a user who understands the business rules, not like a technical attacker looking for injection points.

**3. Input Validation Must Include Range and Sign Constraints:**
Validating that a quantity is a number is not sufficient. A complete validation for a cart quantity field requires:
- Type check: must be an integer (not a float, string, or array)
- Sign check: must be greater than zero
- Range check: must not exceed a reasonable maximum
- Integrity check at checkout: re-validate all constraints against server-side data

Any single missing constraint is an exploitable gap.

**4. The Zero-Floor Guard Reveals the Developer's Mental Model:**
The existence of `err=NEGATIVE_TOTAL` reveals that the developer who implemented it was thinking about preventing the application from owing money to the user — which is the scenario where total goes negative. They were not thinking about the scenario where a manipulated-but-positive total allows a high-value item to be purchased far below cost. Understanding what a partial fix was designed to prevent helps identify what it was not designed to prevent.

**5. Two-Item Exploit Chains Are a Business Logic Pattern:**
Labs 1 and 2 together demonstrate a pattern: business logic vulnerabilities often require combining application features in ways the developer did not anticipate. Lab 1 needed only one parameter change. Lab 2 required adding a second product and calculating an offset. Real-world business logic exploitation frequently involves combining discount codes, referral credits, return mechanisms, and cart items in combinations the developer tested individually but never as a combined exploit chain.

## References

1. PortSwigger Web Security Academy - Business Logic Vulnerabilities
2. PortSwigger Web Security Academy - High-Level Logic Vulnerability
3. OWASP Top 10 (2021) - A04: Insecure Design
4. CWE-20 - Improper Input Validation
5. CWE-840 - Business Logic Errors
6. OWASP Testing Guide - Testing for Business Logic (OTG-BUSLOGIC)
7. OWASP Business Logic Vulnerability Cheat Sheet

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Quantity manipulation, negative value exploitation, or any form of price fraud against live e-commerce systems without explicit written authorization constitutes fraud and is illegal under applicable laws worldwide.

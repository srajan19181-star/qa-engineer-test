## Bug A: Payment Amount Data Type

**Location:** src/routes/payment.routes.js (lines 16, 52)  
**Severity:** High
**Component:** Payment API

**How to Reproduce:**
1. Start server: `npm start`
2. Login: `POST /api/auth/login` with `{"email":"merchant@test.com","password":"Test123!"}`
3. Send request: `GET /api/payments` with JWT token
4. Observe response JSON

**Expected Behavior:**
The `amount` field should be returned as a numeric type (e.g. `25.5`).

**Actual Behavior:**
The `amount` field is returned as a string type (e.g. `"25.5"`).

**Impact:**
Returning amount as a string can cause frontend calculation errors. String concatenation instead of addition can cause severe financial miscalculations.

**Root Cause:**
The server was converting amount to string by explicitly calling `.toString()`.

**Suggested Fix:**
Use `parseFloat(p.amount)` instead of `p.amount.toString()`.

---

## Bug B: Unrestricted Payment Amount

**Location:** src/routes/payment.routes.js (line 75)  
**Severity:** High  
**Component:** Payment API

**How to Reproduce:**
1. Start server: `npm start`
2. Login: `POST /api/auth/login`
3. Send request: `POST /api/payments` with `{"amount": 999999999, "currency": "EUR", "customer_email": "test@test.com"}`

**Expected Behavior:**
The API should reject amounts that are unrealistically high.

**Actual Behavior:**
The API accepts any positive amount, including billions.

**Impact:**
Malicious users or faulty clients could create massive, unrealistic payments, causing system instability or triggering fraud alerts.

**Root Cause:**
There was no upper limit validation on the `amount` field in the request body.

**Suggested Fix:**
Add a check `if (amount > 50000) { return res.status(400).json({ error: 'Amount exceeds maximum limit' }); }`

---

## Bug C: Over-permissive Email Validation

**Location:** src/routes/auth.routes.js (line 12)  
**Severity:** Medium  
**Component:** Auth API

**How to Reproduce:**
1. Send request: `POST /api/auth/register` with `{"email": "user@domain", "password": "TestPassword123!", "business_name": "Test"}`

**Expected Behavior:**
Registration should fail due to invalid email format (missing TLD).

**Actual Behavior:**
Registration succeeds.

**Impact:**
Users can register with non-deliverable email addresses, causing issues for account recovery and notifications.

**Root Cause:**
The regex `/^[^\s@]+@[^\s@]+$/` only checked for string structure around `@`, without validating the presence of a top-level domain.

**Suggested Fix:**
Update the regex to `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`.

---

## Bug D: Missing Webhook Signature Verification

**Location:** src/routes/webhook.routes.js (line 21)  
**Severity:** Critical  
**Component:** Webhooks

**How to Reproduce:**
1. Send request: `POST /api/webhooks/payment-status` with `{"payment_id": "1", "status": "completed", "event_id": "evt_123"}` and no valid signature header.

**Expected Behavior:**
The request should be rejected with 401 Unauthorized.

**Actual Behavior:**
The webhook is processed, and the payment status is updated.

**Impact:**
Any attacker can spoof a webhook payload and fraudulently mark unpaid payments as completed.

**Root Cause:**
The signature verification logic was commented out.

**Suggested Fix:**
Uncomment and implement HMAC-SHA256 signature verification comparing against `WEBHOOK_SECRET`.

---

## Bug E: Missing Webhook Idempotency Check

**Location:** src/routes/webhook.routes.js (line 38)  
**Severity:** High  
**Component:** Webhooks

**How to Reproduce:**
1. Send request: `POST /api/webhooks/payment-status` twice with the same `event_id`.

**Expected Behavior:**
The second request should acknowledge receipt without re-processing the logic.

**Actual Behavior:**
The webhook processes the payment state again every time.

**Impact:**
Multiple deliveries of the same webhook (common with payment providers) could lead to redundant processing, incorrect analytics, or duplicate state updates.

**Root Cause:**
Idempotency check logic was commented out.

**Suggested Fix:**
Uncomment the idempotency check to verify if `event_id` is in `db.processed_webhooks`, and store it after processing.

---

## Bug F: Duplicate Table Orders Allowed

**Location:** src/routes/table-order.routes.js (line 44)  
**Severity:** Medium  
**Component:** Table Orders

**How to Reproduce:**
1. Send request: `POST /api/table-orders` with valid payload for a specific `table_number`.
2. Immediately send the same request again.

**Expected Behavior:**
The second request should be rejected because a pending order already exists.

**Actual Behavior:**
Both orders are created as pending for the same table.

**Impact:**
Customers or staff could accidentally create duplicate orders, leading to over-serving and waste.

**Root Cause:**
Validation checking for existing pending orders was commented out.

**Suggested Fix:**
Uncomment the validation block that checks if `db.table_orders` already contains a pending order for the given table.

---

## Bug G: Duplicate E-Invoice Submission Allowed

**Location:** src/routes/einvoice.routes.js (line 50)  
**Severity:** Medium  
**Component:** E-Invoice

**How to Reproduce:**
1. Send request: `POST /api/einvoice/submit` for a specific `invoice_id`.
2. Send the exact same request again.

**Expected Behavior:**
The API should reject the second submission.

**Actual Behavior:**
Both requests succeed, returning a new submission ID and overwriting the previous state.

**Impact:**
Submitting the same invoice multiple times to the tax authority can result in duplicate tax liabilities or API rate-limiting penalties.

**Root Cause:**
Check for existing `einvoice_status` being `submitted` or `accepted` was commented out.

**Suggested Fix:**
Uncomment the duplicate validation check.

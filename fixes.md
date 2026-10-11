## Fix 1: Payment Amount Data Type

**Bug A:** Amount returned as string instead of number

**Files Changed:**
- src/routes/payment.routes.js (lines 16, 52)

**Code Changes:**
```diff
- amount: p.amount.toString(),
+ amount: parseFloat(p.amount),
```

**Why This Fix:**
Strings can cause NaN errors and incorrect mathematical operations in frontend applications. Using parseFloat ensures numerical integrity.

---

## Fix 2: Unrestricted Payment Amount

**Bug B:** No maximum limit check on payment amount

**Files Changed:**
- src/routes/payment.routes.js (lines 75)

**Code Changes:**
```diff
+ if (amount > 50000) {
+   return res.status(400).json({ error: 'Amount exceeds maximum limit of 50000' });
+ }
```

**Why This Fix:**
Prevents massive erroneous transactions which could crash the system, skew data, or cause payment provider failures.

---

## Fix 3: Over-permissive Email Validation

**Bug C:** Regex allows emails without TLD

**Files Changed:**
- src/routes/auth.routes.js (line 12)

**Code Changes:**
```diff
- const emailRegex = /^[^\s@]+@[^\s@]+$/;
+ const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
```

**Why This Fix:**
Enforces the existence of a domain structure, increasing the likelihood that the email is deliverable.

---

## Fix 4: Missing Webhook Signature Verification

**Bug D:** Webhooks processed without verifying sender

**Files Changed:**
- src/routes/webhook.routes.js (lines 21-32)

**Code Changes:**
```diff
+ const expectedSignature = crypto
+   .createHmac('sha256', WEBHOOK_SECRET)
+   .update(JSON.stringify(req.body))
+   .digest('hex');
+ 
+ if (signature !== expectedSignature) {
+   return res.status(401).json({ error: 'Invalid signature' });
+ }
```

**Why This Fix:**
Critically required to verify that the webhook request authentically originated from the trusted payment provider.

---

## Fix 5: Missing Webhook Idempotency Check

**Bug E:** Webhooks can be processed redundantly

**Files Changed:**
- src/routes/webhook.routes.js (lines 38-44, 55-56)

**Code Changes:**
```diff
+ if (db.processed_webhooks.includes(event_id)) {
+   return res.status(200).json({ message: 'Already processed' });
+ }
...
+ db.processed_webhooks.push(event_id);
```

**Why This Fix:**
Protects against network retries or misconfigured providers sending identical webhook events, preserving correct system state.

---

## Fix 6: Duplicate Table Orders Allowed

**Bug F:** Allowing multiple pending orders for the same table

**Files Changed:**
- src/routes/table-order.routes.js (lines 44-55)

**Code Changes:**
```diff
+ const existingOrder = (db.table_orders || []).find(
+   o => o.merchant_id === merchant_id && 
+        o.table_number === table_number && 
+        o.status === 'pending'
+ );
+ if (existingOrder) {
+   return res.status(409).json({ error: 'Table already has a pending order' });
+ }
```

**Why This Fix:**
Ensures clear business logic state in point of sale systems, preventing double billing or confusion.

---

## Fix 7: Duplicate E-Invoice Submission Allowed

**Bug G:** Allowing multiple submissions for the same e-invoice

**Files Changed:**
- src/routes/einvoice.routes.js (lines 50-58)

**Code Changes:**
```diff
+ if (invoice.einvoice_status === 'submitted' || invoice.einvoice_status === 'accepted') {
+   return res.status(409).json({ 
+     error: 'Invoice already submitted to tax authority' 
+   });
+ }
```

**Why This Fix:**
Avoids issues with tax authorities related to duplicate invoice references and saves unnecessary API calls.

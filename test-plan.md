# Webhook Payment Flow Test Plan

## 1. Test Scope
- **What will be tested:** End-to-end payment webhook flow (receiving, validating, updating status, and idempotency).
- **What won't be tested:** UI representation, performance/load testing, other unrelated webhook types.

## 2. Test Environment
- API: `http://localhost:3000`
- Webhook simulator: Jest/Supertest or Postman
- Database: In-memory (resets on restart)

## 3. Test Scenarios

- ✅ Valid webhook with correct signature updates payment status successfully.
- ✅ Webhook with invalid signature should reject with 401 Unauthorized.
- ✅ Duplicate webhook (same `event_id` twice) should return 200 but process only once.
- ✅ Webhook for non-existent payment should return 404 Not Found.
- ✅ Webhook arrives before payment created (if async creation is possible, returns 404).
- ✅ Multiple webhooks for same payment (different statuses) updates successfully in order.
- ✅ Webhook with missing fields (`payment_id`, `status`) should return 400 Bad Request.
- ✅ Webhook with malformed JSON should return 400 Bad Request.
- ✅ Concurrent webhooks (race condition) shouldn't violate idempotency rules.
- ✅ Webhook with SQL injection attempt in `payment_id` shouldn't impact backend.
- ✅ Webhook signature timing attack (ensuring `crypto.timingSafeEqual` or equivalent is used).
- ✅ Webhook replay attack (old `event_id`) should be caught by idempotency checks.

## 4. Security Tests
- **Signature verification (HMAC-SHA256):** Test with valid, invalid, missing, and incorrectly typed signatures.
- **Replay attack prevention:** Repeated `event_id`s must be caught and ignored safely.
- **SQL injection:** Attempt injection in payload parameters to verify ORM safety.
- **XSS:** Attempt script tags in the `status` field to ensure safe storage/rendering.

## 5. Edge Cases
- Very large `payment_id` string or number causing overflow.
- Unicode/Emojis in `status` or `event_id` field.
- Null/empty `event_id` and whether it circumvents idempotency.
- Future timestamps or unusual timing discrepancies.

## 6. Integration Tests
- **Database Persistence:** Webhook → Payment status update → Database correctly shows updated status and time.
- **Socket.IO:** Webhook → Socket.IO event emission triggers for connected clients.
- **Retry Logic:** Failed webhook (e.g. 500 error simulated) responds properly to initiate external provider retry.

## 7. Risk Assessment
- **Risk:** Signature verification flaw allowing false payments.
- **Detection:** Discrepancy between internal payment records and actual gateway settlement records.
- **Mitigation:** Implement strict HMAC validation and reconcile internal records periodically.

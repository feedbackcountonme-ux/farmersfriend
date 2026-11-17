# Security Specification — Farmers Friend Firestore

## 1. Data Invariants
1. **Authentication & Verification**: All non-public writes require `request.auth != null && request.auth.token.email_verified == true`.
2. **PII Isolation**: User emails, phone numbers, and residential localities live strictly in `/users/{userId}/private/{infoId}` and are readable/writable only by the owning user (`request.auth.uid == userId`) or verified admin (`isAdmin()`). Blanket `isSignedIn()` reads and `list` queries on `/users` or `/users/{userId}/private` are forbidden.
3. **Self-Assigned Role Prevention**: Creating `/users/{userId}` restricts `role == 'customer'`; users can never self-elevate to `'admin'` or write `/admins/{userId}`.
4. **Public Catalogue Integrity**: `/specimens/{specimenId}`, `/banner_config/{configId}`, and `/taxonomy_config/{configId}` contain zero PII and allow public read/query of valid botanical records while enforcing strict admin-only writes with full field validation.
5. **Enquiry Ownership & Terminal Locking**: Every `/enquiries/{enquiryId}` record binds `userId == request.auth.uid` on creation (`status == 'pending'`), restricts `list` queries via `resource.data.userId == request.auth.uid || isAdmin()`, keeps `userId` and `createdAt` immutable, and locks updates once `status == 'fulfilled'` unless performed by `isAdmin()`.
6. **Master Gate & Atomic Chat Threads**: Every `/concierge_threads/{userId}/messages/{messageId}` requires the parent `/concierge_threads/{userId}` document to exist (via `exists(...)` or `existsAfter(...)` inside the same batch write), restricts `senderRole` to `'customer'` for non-admins, binds `senderUid == request.auth.uid`, and forbids message tampering or deletion by customers.
7. **Temporal & ID Integrity**: Document IDs match `^[a-zA-Z0-9_\-]+$` (`<= 128` chars), and all `createdAt`/`updatedAt` timestamps strictly equal `request.time`.

## 2. The "Dirty Dozen" Adversarial Payloads
1. **Unverified Email Spoof**: Admin email `info.pncom@gmail.com` with `email_verified: false` writing to `/specimens/spec-1` -> **DENIED**.
2. **Role Escalation Create**: User `u1` creating `/users/u1` with `{ role: "admin" }` -> **DENIED**.
3. **Shadow Ghost Field**: User `u1` creating `/users/u1` with extra `{ isVerified: true }` -> **DENIED** by `hasOnly`.
4. **PII Cross-Read**: Authenticated user `u2` calling `get(/users/u1/private/info)` -> **DENIED**.
5. **Unbounded List Scraping**: User `u2` calling `list(/enquiries)` without matching `userId` -> **DENIED** by `resource.data.userId == request.auth.uid`.
6. **Identity Spoofing on Enquiry**: User `u2` creating `/enquiries/inq-1` with `{ userId: "u1" }` -> **DENIED**.
7. **Terminal State Bypass**: User `u1` updating `/enquiries/inq-1` when existing `status == "fulfilled"` -> **DENIED**.
8. **Admin Role Fabrication**: User `u1` writing `/admins/u1` -> **DENIED**.
9. **Client Timestamp Forgery**: User `u1` creating `/enquiries/inq-1` with backdated `createdAt` (`!= request.time`) -> **DENIED**.
10. **ID Poisoning Attack**: Writing to `/specimens/` with a 500-char junk document ID containing `/` or spaces -> **DENIED** by `isValidId`.
11. **Customer Impersonating Staff in Chat**: User `u1` creating `/concierge_threads/u1/messages/m1` with `senderRole: "admin"` -> **DENIED**.
12. **Orphaned Chat Message / Unbounded Payload**: User `u1` writing a 50KB `text` string or writing to `/concierge_threads/u2/messages/m1` -> **DENIED**.

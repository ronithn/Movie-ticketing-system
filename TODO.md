# To-do

## Done
- [x] **Per-seat booking storage** — bookings keyed by show + seat; collision-proof,
  atomic double-booking protection.
- [x] **Cancellation flow** — admin cancel deletes the seats (and the week lock) and
  frees the seat live. Required publishing `allow update, delete: if request.auth != null`
  on `bookings`.
- [x] **One booking per mobile number per week** — atomic `weekLocks` guard +
  client pre-check. Requires the `weekLocks` rule to be published.
- [x] **Screening change-log** — every movie set by admin is recorded in `movieLog`.
- [x] **Mobile seat-map clipping** — wide rows (JCOs/ORs) now scroll cleanly from
  the left edge instead of clipping the first seats.

## Backlog
- [ ] **PII split** — bookings are world-readable, exposing customer names + mobile
  numbers to anyone with the (public) Firebase config. Split into a public, PII-free
  `occupancy` collection (powers the seat map) and a private `bookings` collection
  with contact details, readable only by admins. Bookings/cancels become atomic
  writes across both. Also hash the mobile number in the `weekLocks` id (it currently
  contains the raw number and is publicly readable).
- [ ] (optional) **Sequential ticket numbers** — replace the random `KA-YYYY-XXXXX`
  code with a real running number via a transactional counter.
- [ ] (optional) **Firebase Hosting** — add `firebase.json` so the app can be
  deployed to a public URL.

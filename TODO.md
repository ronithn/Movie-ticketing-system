# To-do

## Priority
- [ ] **Fix the cancellation flow** — clicking *Cancel* in the admin panel does not
  free the seat on the seating page. Investigation so far:
  - The seat-map refresh logic is correct (removing a reservation re-renders the
    seat as available).
  - `fbCancelBooking()` builds the right delete batch.
  - Reproduced `permission-denied` on delete when **unauthenticated** (expected).
  - Root cause is almost certainly that the published Firestore rules for
    `bookings` don't allow `delete` for admins, i.e. they are missing
    `allow update, delete: if request.auth != null;`. Verify the published rules,
    or test with an admin login.

## Backlog
- [ ] **PII split** — bookings are currently world-readable, exposing customer
  names + mobile numbers to anyone with the (public) Firebase config. Split into:
  - a public, PII-free `occupancy` collection (powers the seat map), and
  - a private `bookings` collection with contact details, readable only by admins.
  Bookings/cancels become atomic writes across both collections.
- [ ] **Tighten security rules** — unauthenticated clients should not be able to
  delete bookings. Ensure `delete` requires `request.auth != null`.
- [ ] (optional) **Sequential ticket numbers** — replace the random
  `KA-YYYY-XXXXX` code with a real running number via a transactional counter.
- [ ] (optional) **Firebase Hosting** — add `firebase.json` so the app can be
  deployed to a public URL.

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
- [x] **Read-cost fix (occupancy doc per show)** — the public seat map used to
  subscribe to the *entire* `bookings` collection (one read per booked seat, per
  client, continuously → ~73M reads/week). It now reads a single
  `occupancy/{date}_{slot}` document — a `{ seatId: ticketNo }` map — so cost is
  **one read per show viewed**. Booking writes seats into that doc in the same
  atomic batch; cancel removes them with `deleteField()`. The full `bookings`
  collection is subscribed only while an admin is signed in. **Requires the new
  `occupancy` rule to be published (see below).**

## Security note
- User self-cancel (by ticket number) requires **public deletes** on `bookings`
  and `weekLocks`. This means anyone who knows a ticket number can cancel that
  booking (and, via direct DB access, delete arbitrary bookings). Accepted trade-off
  for a backend-less app. Revisit if a backend / Cloud Function becomes available
  (move cancel behind a verified function).

## Firestore rules (publish these)

The read-cost fix adds an `occupancy` collection. Publish rules that let the
public create/update it (mirroring seats) while reads stay cheap. Example:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Public seat map reads this; booking/cancel writes to it. No PII here —
    // just { seats: { seatId: ticketNo } }.
    match /occupancy/{showId} {
      allow read: if true;
      allow create, update: if true;   // guarded by the atomic booking batch
      allow delete: if request.auth != null;
    }

    match /bookings/{seatId} {
      allow read: if true;                       // needed for cancel-by-ticket lookup
      allow create: if true;                     // create-only ⇒ no double-booking
      allow update, delete: if true;             // public self-cancel (accepted trade-off)
    }

    match /weekLocks/{lockId} {
      allow read: if true;
      allow create: if true;                     // create-only ⇒ one booking/number/week
      allow update: if false;
      allow delete: if true;                     // freed on cancel
    }

    match /config/{doc} {
      allow read: if true;
      allow write: if request.auth != null;      // admin only
    }

    match /movieLog/{doc} {
      allow read: if true;
      allow create: if request.auth != null;
      allow update, delete: if false;
    }
  }
}
```

> The public `update/delete` on `bookings`/`weekLocks`/`occupancy` is the same
> backend-less self-cancel trade-off noted below — anyone who knows a ticket
> number (or has direct DB access) can cancel bookings. Lock this down behind a
> Cloud Function if a backend becomes available.

## Backlog
- [ ] **PII split (finish it)** — the public, PII-free `occupancy` collection now
  exists and powers the seat map, but `bookings` is still world-*readable* (the
  cancel-by-ticket lookup queries it), so customer names + mobile numbers remain
  exposed to anyone with the (public) Firebase config. To close this: restrict
  `bookings` reads to admins and move cancel behind a Cloud Function (or a lookup
  keyed by a hashed ticket token), so the public never reads contact details. Also
  hash the mobile number in the `weekLocks` id (it currently contains the raw
  number and is publicly readable).
- [ ] (optional) **Sequential ticket numbers** — replace the random `KA-YYYY-XXXXX`
  code with a real running number via a transactional counter.
- [ ] (optional) **Firebase Hosting** — add `firebase.json` so the app can be
  deployed to a public URL.

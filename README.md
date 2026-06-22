# Kerketta Auditorium — Movie Ticket Booking

A single-page web app for booking seats at **Kerketta Auditorium**. The entire
application lives in [`index.html`](index.html) — open it in any modern browser
to run it. No build step is required.

## Features

- **Weekly schedule** — a Friday → Thursday week strip with show times
  (18:30 on Fri/Sat/Mon/Tue/Wed, an extra 14:30 show on Sunday, closed on
  Thursday for maintenance). Seat availability is keyed by category + date + slot.
- **Seating layout** mirroring the physical auditorium chart:
  - **OFFRs** (Officers) — 58 seats, rows M–R centre section.
  - **JCOs** (Junior Commissioned Officers) — 116 seats, left/right sections
    with an aisle and continuous numbering.
  - **ORs** (Other Ranks) — 366 seats across left / tapering centre / right
    blocks, rows A–K, with continuous numbering.
  - **VIP** — 11 sofa seats shown above the map, display-only (not bookable).
- **Booking flow** — pick a day and show time, choose a category, select seats,
  enter name + 10-digit mobile number, and confirm.
- **One booking per number per week** — a mobile number may hold a single booking
  (which can include multiple seats) per Friday→Thursday week.
- **No double-booking** — a seat can never be sold twice for the same show; this
  is enforced atomically by the data model, not just the UI.
- **Tickets** — a styled on-screen ticket plus a client-side **PDF** generated
  with [jsPDF](https://github.com/parallax/jsPDF); the PDF auto-downloads on
  confirmation and can be re-downloaded or printed.
- **Admin panel** — configure the current movie (title, poster, genre, cast,
  synopsis, etc.), view seat availability and the weekly schedule, browse and
  cancel bookings by date, and see a **screening log** of every movie set.
- **Real-time sync** — bookings and movie config sync live across devices via
  **Firebase Firestore**, with admin access gated by **Firebase Authentication**.

## How it works

The whole app is one `index.html`: inline HTML/CSS and vanilla JS. A small
`render()` function rebuilds the page from in-memory state, and Firestore
real-time listeners call `render()` whenever data changes — so every open client
stays in sync.

**Identity of a booking.** A booking is really *"this seat, for this show
(date + time)."* So instead of one document per ticket, the app stores **one
document per booked seat**, with the document id derived from the show and seat:

```
bookings/2026-06-23_18:30_OFFRS-M3
```

This makes two guarantees fall out for free:

- **Different shows/seats never collide** — each maps to a distinct id.
- **The same seat can't be sold twice** — two people booking it map to the *same*
  id, and the security rules permit a `create` only when the document does not
  already exist, so the second write is rejected. A multi-seat booking is written
  as a single atomic batch (all seats succeed or none do).

**One booking per number per week.** Each booking also writes a lock document
`weekLocks/{weekStart}_{mobile}` in the same atomic batch. The rules allow it to
be created only if absent (never updated), so a second booking by the same number
in the same week fails. Cancelling a booking deletes the lock, freeing the number.

**Ticket number.** `KA-<year>-<random>` (e.g. `KA-2026-87B07`) is generated at
booking time. It is a human-facing reference and the key used to group a booking's
seats together and to target cancellation — it is *not* the storage key.

**Movie & history.** The current movie lives in a single `config/movie` document
shown on the home hero. Every time an admin saves a movie, an entry is also
appended to `movieLog` so there is a permanent record of what was screened.

## Running

The app loads Firebase as an ES module, so it must be served over **http://** —
opening `index.html` directly from the filesystem (`file://`) is blocked by the
browser's module CORS rules.

```sh
# from the folder containing index.html
python3 -m http.server 8000
# then open http://localhost:8000
```

Any static server works (`npx http-server`, VS Code Live Server, Netlify, etc.).

## Configuration

### Firebase

Real-time sync and admin login use Firebase. The `firebaseConfig` block and the
Firestore/Auth listeners live inline in the `<script type="module">` near the top
of `index.html`. To point the app at your own project:

1. Create a Firebase project and enable **Firestore** and **Email/Password
   Authentication**.
2. Create an admin user under Authentication.
3. Replace the `firebaseConfig` object in `index.html` with your project's config.

Firestore uses four collections:

- `config/movie` — a single document holding the current movie details.
- `bookings/{showDate_slot_seat}` — one document per booked seat.
- `weekLocks/{weekStart_mobile}` — one-booking-per-number-per-week locks.
- `movieLog/{autoId}` — append-only history of movies set by the admin.

**Security rules.** The app depends on rules that let the public create bookings
and locks (validated) while restricting overwrites, cancels, and movie edits to
authenticated admins. A booking will fail with `permission-denied` until the rules
for **all four** collections are published. See the rules block in the project
notes / `TODO.md` for the exact ruleset to paste into the Firestore console.

> Note: a Firebase web `apiKey` is a public client identifier, not a secret.
> Protect data with Firestore security rules and Authentication, not by hiding
> the config.
>
> Known limitation: booking documents (and the `weekLocks` ids) are currently
> world-readable, which exposes customer names/mobile numbers to anyone with the
> public config. Splitting contact details into an admin-only collection is
> tracked in `TODO.md`.

## Admin access

Open the **Admin** button in the header and sign in with the Firebase
Authentication email/password you configured. From the admin panel you can edit
the movie, review availability, and cancel bookings (which releases the seats).

## Notes

- No online payments — tickets are confirmation-only and presented at the gate.
- There are no end-user accounts; only admins log in.

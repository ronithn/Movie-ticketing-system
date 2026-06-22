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
- **Tickets** — a styled on-screen ticket plus a client-side **PDF** generated
  with [jsPDF](https://github.com/parallax/jsPDF); the PDF auto-downloads on
  confirmation and can be re-downloaded or printed.
- **Admin panel** — configure the current movie (title, poster, genre, cast,
  synopsis, etc.), view seat availability and the weekly schedule, browse and
  cancel bookings by date.
- **Real-time sync** — bookings and movie config sync live across devices via
  **Firebase Firestore**, with admin access gated by **Firebase Authentication**.

## Running

```sh
# just open the file
open index.html        # macOS
xdg-open index.html    # Linux
```

Or serve it statically (e.g. `python3 -m http.server`) and visit the page.

## Configuration

### Firebase

Real-time sync and admin login use Firebase. The `firebaseConfig` block and the
Firestore/Auth listeners live inline in the `<script type="module">` near the top
of `index.html`. To point the app at your own project:

1. Create a Firebase project and enable **Firestore** and **Email/Password
   Authentication**.
2. Create an admin user under Authentication.
3. Replace the `firebaseConfig` object in `index.html` with your project's config.

Firestore is used with two collections:

- `config/movie` — a single document holding the current movie details.
- `bookings/{ticketNo}` — one document per booking.

> Note: a Firebase web `apiKey` is a public client identifier, not a secret.
> Protect data with Firestore security rules and Authentication, not by hiding
> the config.

## Admin access

Open the **Admin** button in the header and sign in with the Firebase
Authentication email/password you configured. From the admin panel you can edit
the movie, review availability, and cancel bookings (which releases the seats).

## Notes

- No online payments — tickets are confirmation-only and presented at the gate.
- There are no end-user accounts; only admins log in.

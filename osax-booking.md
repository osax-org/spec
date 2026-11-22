# OSAX — Booking interface — v0.3 draft

Part of OSAX, the Open Sports Activity Exchange. The overview — access rules, memberships, player IDs,
base URL — is in
[osax.md](osax.md)

This interface lets a client book players into a publisher's activity — a free court (`play`), an
event, a lesson, a place in an open match — and take the payment itself, instead of sending players
to the publisher's page. It builds on the [Activities
interface](osax-activities.md), which
is how the client finds the activities.

**Optional, per activity.** A publisher can share its activities without offering booking. It offers
this interface only for activities it marks `directBooking: true` (Activities §3); a publisher
without it answers `501` (Overview §4). Everything else is booked at the activity's `url`.

**The client handles the money.** The client takes the payment and pays out refunds, following the
activity's cancellation terms (§3) and the publisher's callbacks (§2.4); the publisher never takes
money in this flow.

**One booking per thing paid for.** A booking is what one payment covers, so each can be cancelled
and refunded on its own:

- **Priced per participant** (events, lessons, a place in an open match, tournaments): one booking
  per player. To sign up two players, the client books twice and gets two booking IDs.
- **Priced per booking** (a court, `play`): one booking for the whole thing, with or without its
  players (§2.2). A player dropping out changes nothing; the booking stands.

## 1. Identifying players

Players are given by player ID (Overview §3.6), of a kind in the publisher's `playerIds`
(Activities §4.2): a federation (`knltb.nl:1234567`), the publisher's own accounts
(`bookingsystem.nl:5521`), or an email address with the player's name (`jan@example.nl`). `guest`
stands for a player who is not identified. The player ID decides the price: a player the publisher
recognises as a member gets the member price.

## 2. API

Paths are relative to the publisher's base URL; JSON; API key and status codes as in Overview §4.

| Endpoint | Does |
|---|---|
| `POST /activities/{id}` | Personal prices for these players (§2.1) |
| `POST /bookings` | Book the activity (§2.2) |
| `DELETE /bookings/{bookingId}` | Client cancels a booking (§2.3) |
| `POST {callbackUrl}` | Publisher tells the client about a cancellation or resale (§2.4) |

### 2.1 Personal prices

`POST /activities/{id}` with the players in the body, given as for `POST /bookings` (§2.2), returns
the activity with its current details plus a `quote`: what each player pays, the total, and the
activity's refund window (`cancelBeforeHours`, §3). It reserves nothing. Players go in the body,
never in the URL: player IDs and email addresses are personal data, and URLs end up in logs.

```
POST /activities/event-77
{ "players": [
    { "player": "knltb.nl:1234567" },
    { "player": "knltb.nl:7654321" },
    { "player": "guest" }
] }
```

```json
"quote": {
  "players": [
    { "player": "knltb.nl:1234567", "amount": 7.50, "label": "Member" },
    { "player": "knltb.nl:7654321", "amount": 8.50, "label": "KNLTB member" },
    { "player": "guest", "amount": 9.50, "label": "Non-member" }
  ],
  "total": 25.50,
  "currency": "EUR",
  "cancelBeforeHours": 24
}
```

A player not allowed in (members only, level, full) gets `"allowed": false` and a `reason` instead
of an amount.

For a price `per: "booking"` (typical for `play`):

- **Without players** (`GET /activities/{id}`), the quote has only `total`: the booking price.
- **With players**, the publisher prices each player as it does for an activity priced per
  participant, and `total` is the sum of their amounts. List everyone who will play; use `guest` for
  a player you don't know.
- A publisher that can't price players separately leaves out `players` and gives the booking price
  as `total`. The client then splits it as it likes.

```
POST /activities/play-4521-p1-2026-11-20T19:00-90
{ "players": [
    { "player": "bookingsystem.nl:5521" },
    { "player": "knltb.nl:1234567" },
    { "player": "guest" },
    { "player": "guest" }
] }

"quote": {
  "players": [
    { "player": "bookingsystem.nl:5521", "amount": 0, "label": "Member, court included" },
    { "player": "knltb.nl:1234567", "amount": 8.00, "label": "Non-member" },
    { "player": "guest", "amount": 8.00, "label": "Non-member" },
    { "player": "guest", "amount": 8.00, "label": "Non-member" }
  ],
  "total": 24.00,
  "currency": "EUR"
}
```

### 2.2 Book

Priced per participant — exactly one player:

```
POST /bookings
{ "activityId": "event-77",
  "players": [ { "player": "knltb.nl:1234567" } ],
  "callbackUrl": "https://clubmatch.org/osax/callbacks/bookings" }
```

Priced per booking — with the players on the court, or without them:

```
POST /bookings
{ "activityId": "play-4521-p1-2026-11-20T19:00-90",
  "players": [
    { "player": "knltb.nl:1234567" },
    { "player": "jan@example.nl", "name": "Jan de Vries" }
  ],
  "callbackUrl": "https://clubmatch.org/osax/callbacks/bookings" }

POST /bookings
{ "activityId": "play-4521-p1-2026-11-20T19:00-90",
  "callbackUrl": "https://clubmatch.org/osax/callbacks/bookings" }
```

With players, the booking is made at their own prices and the `quote` in the response has one amount
per player, as in §2.1. Without players, it is made at the booking price.

`name` is required with `email`, and only then. `callbackUrl` is required: where the publisher
reports cancellations and resales (§2.4). Returns `201` with `{ "bookingId": "b-88213", "quote": { …
} }`: the booking is made at that price. `400` when more than one player is given for an activity
priced per participant; `409` when it is no longer available (court taken, activity full); `403`
when a player is not allowed in. There is no separate confirm step.

### 2.3 Pay or cancel

After booking, the client takes the payment. If payment fails or is abandoned, the client cancels
with `DELETE /bookings/{bookingId}`; the client is responsible for cancelling, for example with its
own timer. Returns `204`.

A client also cancels this way when its player cancels, and refunds by the activity's terms (§3): in
full before the deadline; after it, nothing — unless the activity has `refundIfResold` and a
`resold` callback arrives later.

### 2.4 Callbacks

The publisher posts to the booking's `callbackUrl`, one call per booking. The client answers `2xx`;
otherwise the publisher retries for at least 24 hours. Each call is signed with a secret agreed with
the API key, so the client can trust it. The client finds its own payment by `bookingId`.

| `type` | When | Client does |
|---|---|---|
| `cancelled` | The organizer cancelled the activity or this booking. | Refund the booking in full; tell the player(s). An organizer's cancellation is always a full refund. |
| `resold` | The player cancelled after the deadline, and the place or court was sold again (only for `refundIfResold`). | Refund `refund.amount`. |

```
POST https://clubmatch.org/osax/callbacks/bookings
{ "type": "cancelled",
  "bookingId": "b-88213",
  "activityId": "event-77",
  "reasonCode": "rescheduled",
  "reason": "Moved to Sunday because of the club championship",
  "replacementActivityId": "event-81" }

{ "type": "resold",
  "bookingId": "b-88214",
  "activityId": "play-4521-p1-2026-11-20T19:00-90",
  "refund": { "amount": 32, "currency": "EUR" } }
```

`reasonCode` (with `cancelled`, required) is one of:

| `reasonCode` | Meaning |
|---|---|
| `too-few-players` | Not enough players signed up. |
| `courts-unavailable` | Courts cannot be used: weather, maintenance, closure. |
| `rescheduled` | Moved to another time or place. `replacementActivityId` gives the new activity; the client can offer its players to sign up there. |
| `other` | Anything else; `reason` says what. |

`reason` is optional free text for players. A cancelled activity also shows `status: "cancelled"` in
the Activities feed; the callback is what the client acts on.

## 3. Cancellation terms

Set per activity in the Activities interface, and `cancelBeforeHours` is repeated in the quote, so
the client can show them before the player pays:

| Field | Meaning |
|---|---|
| `cancelBeforeHours` | A player may cancel for a full refund up to this many hours before the start (e.g. `24`). Leave out = the player cannot cancel for a refund. |
| `refundIfResold` | `true`: after the deadline, the player still gets a refund if the place or court is sold again, reported by a `resold` callback. |

## 4. Scenarios

- **Two friends join an americano at EUR 10 each.** Two `POST /bookings`, two booking IDs. One drops
  out: the client deletes that booking and refunds EUR 10; the other stays booked.
- **Two players join an open match.** Same: one booking per player, each cancelled on its own.
- **Jan books a padel court for four at EUR 32.** One booking with four players. A friend drops out:
  nothing changes. Jan cancels the court: one `DELETE`, refund by the terms.
- **A member books a squash court with a guest.** The quote gives the member 0 (court included in
  their membership) and the guest 8.00. One booking with both players; the client charges the guest
  only.
- **A club books a court without naming players.** One booking at the booking price.
- **The organizer cancels the americano for too few players.** One `cancelled` callback per booking;
  the client refunds each player in full.

## 5. Flow

1. The client has the activity from the Activities interface, with `directBooking: true`, and shows
   its cancellation terms.
2. `POST /activities/{id}` with the players — show each player their price. For a court, leave out
   `players` to get the booking price only.
3. `POST /bookings` with a `callbackUrl` — one per player, or one for a court — and get a
   `bookingId` each.
4. Take the payment; on failure, `DELETE /bookings/{bookingId}`.
5. If a player cancels: `DELETE` their booking, and refund by the terms.
6. If a callback arrives: refund as it says; for `rescheduled`, offer the replacement activity.

# OSAX — Activities interface — v0.3 draft

Part of OSAX, the Open Sports Activity Exchange. The overview — access rules, sports, levels,
categories, age groups, memberships, player ID issuers, discovery — is in
[osax.md](osax.md)

This interface lets a system publish its activities (free court times to play, matches, events,
tournaments, leagues, lessons, courses) with their venues and courts, and lets others retrieve them.
Booking players in directly is a separate, optional interface:
[Booking](osax-booking.md).

**Retrieve, cache, filter at the client.** The API is for retrieving activities and keeping a copy
in sync, not for searching. A client fetches the activities for the countries and sports it serves,
caches them, and filters for its users in its own system: by place and distance, level, category,
age group, price, places left. The server offers no filters for those. A client that needs the
current details of one activity, for example places left when a user opens it, asks `GET
/activities/{id}`. The exception is type `play` (§1.1): free court times are retrieved live, per
venue and per day, and never cached.

Every request carries an API key; who gets a key, and which activities it can see, is up to the
publisher (Overview §2).

## 1. Activity types

`type` is one of:

| `type` | Meaning |
|---|---|
| `play` | A free time on one resource (court, lane, table, tee) that can be booked to play (§1.1). |
| `match` | One game between players or teams. |
| `event` | Social play, americano, clinic, open session. |
| `tournament` | A competition over one or a few days, with matches inside it. |
| `league` | A competition over a season (league, ladder, competition), with rounds or matches inside it. |
| `lesson` | One session given by a coach. |
| `course` | A series of sessions given by a coach, signed up for and paid for as a whole. |

A match inside a league or tournament points to it with `parentId`; so does a lesson that is one
session of a course.

### 1.1 Rules for type play

A `play` activity is an ordinary activity (§3), with these differences:

- **Published only by systems that hold availability** (booking systems, venues), never republished
  from another system.
- **Retrieved live, one venue, one day:** `GET /activities?type=play&venueId=…&date=…`. `date` at
  most 14 days ahead; optional `sport`, `fromTime`, `toTime` (local times).
- **Not cached, not synced.** Never returned in country-wide retrieval or with `updatedSince`. The
  response carries `checked`; clients show them as "free when checked".
- **One entry per resource, start and length:** exactly one `resourceId`; a start time with several
  lengths gives one entry per length.
- **The ID is made up by the publisher** from venue, resource, start and length (e.g.
  `play-4521-r7-2026-11-20T19:00-90`); nothing is stored until it is booked.
- **Not passed on:** a client shows them to its own users only (Overview §2).

Example: §6.2.

## 2. Venues

A venue is where an activity is played: a club, a commercial centre, public courts, a sports hall.
Every system keeps venues under its own IDs, which mean nothing to another system. So a publisher
shares the bare minimum a client needs to map each venue **once** to its own record, and every
activity refers to its venue by the publisher's `venueId`. How the client matches (exact Google
Place ID, website, or distance plus name) is up to the client. A system serving one club simply
returns one venue.

```json
"venues": [
  { "id": "4521", "name": "Padel Club Example",
    "latitude": 52.0907, "longitude": 5.1214, "country": "NL",
    "city": "Utrecht", "googlePlaceId": "ChIJ…", "website": "padelclubexample.nl",
    "sports": ["padel", "tennis"],
    "resources": [
      { "id": "r7", "name": "Court 7", "sports": ["padel"], "roofType": "indoor", "surface": "artificial-grass", "walls": "glass" },
      { "id": "r12", "name": "Court 12", "sports": ["tennis"], "roofType": "outdoor", "surface": "clay" }
    ],
    "updated": "2026-11-01T09:00:00Z" }
]
```

| Field | R | Notes |
|---|---|---|
| `id` | R | The publisher's own ID; stable. |
| `name` | R |  |
| `latitude`, `longitude` | R | WGS 84, at least 4 decimals (~10 m). |
| `country` | R | ISO 3166-1 alpha-2. |
| `city` |  |  |
| `googlePlaceId` |  | Exact match for clients that use Google. |
| `website` |  | Domain only (`padelclubexample.nl`). Exact match without Google. |
| `sports` |  | Sports (Overview §3.1) this publisher offers activities in at this venue — not everything the venue has. |
| `resources` |  | The venue's courts, lanes, tables, tees (§2.1). Required for a publisher of `play` activities. |
| `updated` | R | UTC; changes when any field above changes (resources included), so a client re-maps only changed venues. |

Nothing else about a venue is shared — no opening hours: a closed venue simply has no free `play`
times. A client wanting more finds it on the venue's own site.

A response carries the venues its activities refer to, before the activities. `GET /venues` (§4)
returns the full list for keeping a mapping in sync.

### 2.1 Resources

A resource is one bookable place to play at a venue: a court, lane, table, tee or pitch. Resources
rarely change, so they are shared with the venue and synced with it; activities refer to them by
`resourceIds`. The values for `roofType`, `surface` and `walls` are kept in the list of [sports,
rating systems and court
properties](osax-lists.md), so they can
grow per sport without a new version.

| Field | R | Notes |
|---|---|---|
| `id` | R | The publisher's own ID; stable, unique within the venue. |
| `name` | R | As players know it: `Court 7`, `Centre court`. |
| `sports` | R | Sports played on it (Overview §3.1); a list, since a hall court can carry lines for several. |
| `roofType` |  | From the list, e.g. `indoor`, `covered`, `outdoor`. |
| `surface` |  | From the list, e.g. `clay`, `artificial-grass`, `wood`. |
| `walls` |  | From the list, e.g. `glass`, `concrete`, `plaster`. Leave out for sports without walls in play. |

## 3. The activity

```json
{
  "id": "match-123456",
  "type": "match",
  "name": "Doubles match — 1 place left",
  "description": "Friendly doubles, all welcome at this level.",
  "sport": "tennis",
  "category": "P2xP2",
  "tags": ["competitive"],

  "startDate": "2026-11-20T19:00:00+01:00",
  "endDate": "2026-11-20T20:30:00+01:00",
  "durationMinutes": 90,
  "timeZone": "Europe/Amsterdam",
  "recurrence": null,
  "status": "open",
  "registrationCloses": "2026-11-20T12:00:00+01:00",

  "venueId": "4521",
  "resourceIds": ["r12"],
  "organizer": { "name": "Padel Club Example", "type": "club", "url": "https://padelclubexample.nl" },
  "federation": { "name": "KNLTB", "url": "https://www.knltb.nl" },

  "minLevel": ["clubmatch:3.0", "wtn:18"],
  "maxLevel": ["clubmatch:4.0", "wtn:24"],
  "ratedIn": ["clubmatch", "wtn"],
  "participants": { "registration": "individual", "min": 4, "max": 4, "current": 3 },

  "requires": null,
  "prices": [
    { "label": "Member", "amount": 7.50, "currency": "EUR", "per": "participant",
      "requires": [{ "venueId": "4521" }] },
    { "label": "KNLTB member", "amount": 8.50, "currency": "EUR", "per": "participant",
      "requires": [{ "federation": "knltb.nl" }] },
    { "label": "Non-member", "amount": 9.50, "currency": "EUR", "per": "participant" }
  ],
  "directBooking": true,
  "cancelBeforeHours": 24,
  "refundIfResold": true,

  "url": "https://clubmatch.org/…",
  "joinUrl": "https://clubmatch.org/…/join",
  "imageUrl": null,
  "language": "nl",
  "parentId": "league-789",

  "publisher": "ClubMatch",
  "created": "2026-11-10T08:00:00Z",
  "updated": "2026-11-13T10:12:00Z"
}
```

### Fields

**R** = required. Everything else may be left out (or `null`) when unknown.

| Field | R | Notes |
|---|---|---|
| `id` | R | Unique and stable within the publisher. For `play`: made up from its parts (§1.1). |
| `type` | R | §1. |
| `name` | R |  |
| `description` |  | Plain text. |
| `sport` | R | Short name from the sport list (Overview §3.1). |
| `category` |  | Who plays against whom: `PxP`, `M2xM2`, `MFxMF`, `P11xP11`… (Overview §3.3). Leave out when it does not apply (play, lessons, courses). |
| `tags` |  | Free tags, lowercase: `competitive`, `social`, `beginners`… |
| `startDate` | R | ISO 8601 with offset. For a league, tournament or course: its first day. |
| `endDate` |  | ISO 8601 with offset. For a league, tournament or course: its last day. |
| `durationMinutes` |  | Required when `endDate` is missing. |
| `timeZone` | R | IANA, e.g. `Europe/Amsterdam`. |
| `recurrence` |  | RRULE, e.g. `FREQ=WEEKLY;BYDAY=TU;UNTIL=20270630T000000Z`. `startDate` is the first occurrence. |
| `status` | R | `announced`, `open`, `closed`, `live`, `paused`, `finished`, `cancelled`, `deleted` (§3.1). |
| `registrationOpens`, `registrationCloses` |  | ISO 8601 with offset. When `announced` becomes `open`, and `open` becomes `closed`. |
| `venueId` | R | The `id` of a venue in the same response (§2). |
| `resourceIds` |  | The venue's resources it is played on (§2.1). Exactly one for `play`; for a tournament or americano the courts it uses, or left out when not known. Roof type, surface, walls and court name come from the resource. |
| `organizer` | R | `name`, `type` (`club`, `federation`, `coach`, `platform`, `person`), `url`. A person only by the public name they agreed to. Optional for `play`. |
| `federation` |  | The body that sanctions it: `name`, `url`. |
| `minLevel`, `maxLevel` |  | Level range, one entry per rating system: `"wtn:13"` (Overview §3.2). Leave out = all levels welcome. |
| `ratedIn` |  | The rating systems the results count for, by short name from the list (Overview §3.2): `["wtn", "clubmatch"]`. Leave out = the results count for no rating. Separate from `minLevel`/`maxLevel`, which only say who is welcome. |
| `participants` |  | How players sign up and how many: `registration` (`individual` or `team`); `min` (needed for it to go ahead), `max` (leave out = unlimited) and `current` (signed up now). Places left = `max − current`. With `team`, `teamSize` gives the players per team as `min` and `max`; this is separate from `category`. A doubles tournament for 8 to 16 pairs with 11 entered: `{ "registration": "team", "teamSize": { "min": 2, "max": 2 }, "min": 8, "max": 16, "current": 11 }`. For `play`: `max` = players allowed on the booking. |
| `ageGroup` |  | Youth age group, `U5` to `U21` (Overview §3.4). Leave out = no age limit. |
| `requires` |  | Members only: the memberships that give access, any one of them is enough (Overview §3.5). Leave out = open to everyone. |
| `prices` | R | List price. One entry per tier: `label`, `amount`, `currency`, `per` (`participant` default, `team`, `booking` for the whole booking), and `requires` for a member price (Overview §3.5). An entry without `requires` is for everyone. A player pays the lowest price whose `requires` they meet. Free = one entry with `amount: 0`. |
| `directBooking` |  | `true`: a client may book this activity through the Booking interface and take the payment itself. Leave out or `false`: players sign up at `url`. Set per activity. |
| `cancelBeforeHours` |  | Cancellation terms: a player may cancel for a full refund up to this many hours before the start, e.g. `24`. Leave out = no refund when the player cancels. An organizer's cancellation is always a full refund (Booking §2.4). |
| `refundIfResold` |  | `true`: a player who cancels after the deadline still gets a refund if the place or court is sold again (Booking §2.4). |
| `url` | R | Public page where a person reads more and joins. |
| `joinUrl` |  | Direct link to sign up, if different. |
| `imageUrl`, `language` |  | `language` as `nl`, `en`, `pt`… |
| `parentId` |  | The league, tournament or recurring activity this belongs to. |
| `publisher` | R | Name of the system that published it. |
| `created`, `updated` | R (`updated`) | UTC. `updated` changes on every change. |

**Members only and member prices.** A client uses `requires` to show the right price and to tell a
player when an activity is for members only; with the player's consent it can check their
memberships first (Identity and membership interface). The publisher checks again when the player
signs up or is booked.

**What to publish** is up to the publisher: which activities (for example only public matches) and
which optional fields.

**Never published:** names, contact details or ratings of participants. By default an activity shows
how many take part and the level range, not who.

**Participants by player ID.** A caller may ask for the participants of activities, identified by
the player IDs of one issuer (Overview §3.6). With `participants=idrettsforbundet.no` (§4.1), each
activity gets `sides`, and a finished match gets `result`:

```json
"sides": [
  { "players": [{ "issuer": "idrettsforbundet.no", "id": "1234567" }] },
  { "players": [{ "issuer": "idrettsforbundet.no", "id": "7654321" }, {}] }
],
"result": { "winner": 1, "score": "6-4 3-6 10-7" }
```

| Field | Notes |
|---|---|
| `sides` | One entry per side in a match, or per registered player or team in other activities. Each player is their ID from the requested issuer; a player without one appears as `{}`. |
| `result` | Finished matches only: `winner` (`1` or `2`, `0` for a draw) and `score` in the sport's usual notation. |

- **Only the issuer, or with permission.** The publisher returns participants only when the caller
  is that issuer, or has the publisher's permission for that issuer's IDs (set on its API key). Any
  other caller gets `403`.
- **Only IDs, never names.** A player ID means something only to its issuer.
- **The publisher's terms with its players** decide whether a player's ID may be shared at all.

**Recurring activities.** Give one activity a `recurrence`. When occurrences differ (their own
places left, a cancelled week), publish each occurrence as its own activity with `parentId` pointing
to the recurring one.

### 3.1 Status

| `status` | Meaning |
|---|---|
| `announced` | Known, but registration not open yet. |
| `open` | Registration open. |
| `closed` | Registration closed, not started yet (matches drawn, or full). |
| `live` | In progress. |
| `paused` | Break between rounds, e.g. a league's summer break. |
| `finished` | Over. |
| `cancelled` | Will not take place. Clients that booked players in directly are also told by callback (Booking §2.4). |
| `deleted` | Removed; only appears when syncing with `updatedSince`. |

- **Full is not a status.** Places left follow from `participants`; a full activity stays `open`
  until registration ends.
- **A new date is an update**, not a status: change `startDate` and `updated`.
- Not every type uses every status: a match is typically `open` → `closed` → `live` → `finished`; a
  `play` activity is always `open`.

## 4. API

Base: `https://{host}/osax/v1`. Read-only, JSON. Every request carries an API key in the `X-Api-Key`
header; a request without a valid key gets `401`.

| Endpoint | Returns |
|---|---|
| `GET /activities?…` | Activities to retrieve and keep in sync (§4.1); free court times with `type=play` (§1.1) |
| `GET /activities/{id}` | One activity, with its current details |
| `GET /activities.ics?…` | The same activities, as a calendar |
| `GET /venues?…` | Venues with their resources, for building and syncing a mapping; takes `country`, `sport`, `updatedSince`, `limit`; pages with `next` |

### 4.1 Retrieving activities

| Parameter | Example | Meaning |
|---|---|---|
| `country` | `NL` or `NL,BE` | ISO 3166-1, of the activity's venue. Required, except with `type=play`. |
| `sport` | `padel` or `padel,tennis` | Overview §3.1; exact names only |
| `type` | `play` or `match,tournament` | §1. `play` cannot be combined with other types and needs `venueId` and `date` (§1.1). |
| `venueId`, `date`, `fromTime`, `toTime` | `4521`, `2026-11-20`, `18:00` | Only with `type=play`. |
| `status` | `finished` or `announced,open` | §3.1. |
| `from`, `to` | `2026-11-14`, `2026-11-30` | Activities that take place in this range |
| `updatedSince` | `2026-11-13T00:00:00Z` | Only changed activities, including `deleted` ones. Not with `type=play`. |
| `limit` | `50` | Max 200 |
| `participants` | `idrettsforbundet.no` | Adds `sides` and `result` (§3, Participants by player ID). |

Without `status` or `updatedSince`, every status except `deleted` is returned. Without `type`, every
type except `play`.

### 4.2 Response

```json
{
  "acceptedIssuers": ["knltb.nl", "padelclubexample.nl", "email"],
  "venues": [ { "id": "4521", "…": "…" } ],
  "activities": [ { "venueId": "4521", "…": "…" } ],
  "next": "https://clubmatch.org/osax/v1/activities?country=NL&sport=padel&page=2",
  "checked": "2026-11-20T16:42:00Z"
}
```

- `acceptedIssuers`: the player ID issuers (Overview §3.6) this publisher accepts to identify
  players, including `email` if it accepts players by email address — usually the same for all
  activities, at least per sport. Used by the Booking interface and the membership check.
- `next` is absent on the last page. `checked` only with `type=play`.
- Errors are an HTTP status with `{ "error": "missing-country", "message": "Give a country." }`.

**Keeping a copy in sync:** fetch everything once, then poll with `updatedSince` = the highest
`updated` you have seen. Venues (with their resources) sync the same way through `GET
/venues?updatedSince=…`.

## 5. Typical flow

1. `GET /venues?country=NL&sport=padel` — the venues with their resources; map each once to your own
   and keep them in sync.
2. `GET /activities?country=NL&sport=padel` — events, lessons, tournaments and more, with
   `acceptedIssuers`; cache them and poll with `updatedSince`.
3. When a player looks at a venue: `GET /activities?type=play&venueId=…&date=…` — free court times,
   live.
4. When a player picks one: `GET /activities/{id}` for current details. If `directBooking`, book
   through the Booking interface; otherwise send the player to `url`.

## 6. Examples

### 6.1 A club's weekly social night

```json
{
  "acceptedIssuers": ["knltb.nl"],
  "venues": [
    { "id": "1", "name": "Padel Club Example", "latitude": 52.0907, "longitude": 5.1214,
      "country": "NL", "city": "Utrecht", "website": "padelclubexample.nl",
      "resources": [
        { "id": "c1", "name": "Court 1", "sports": ["padel"], "roofType": "indoor", "surface": "artificial-grass", "walls": "glass" },
        { "id": "c2", "name": "Court 2", "sports": ["padel"], "roofType": "indoor", "surface": "artificial-grass", "walls": "concrete" }
      ],
      "updated": "2026-09-01T08:00:00Z" }
  ],
  "activities": [
    { "id": "social-night", "type": "event", "name": "Tuesday social night",
      "description": "Mix-in doubles, new partners every round. Rackets for hire at the bar.",
      "sport": "padel", "category": "P2xP2", "tags": ["social"],
      "startDate": "2026-09-01T19:30:00+02:00", "endDate": "2026-09-01T22:00:00+02:00",
      "timeZone": "Europe/Amsterdam", "recurrence": "FREQ=WEEKLY;BYDAY=TU",
      "status": "open", "venueId": "1", "resourceIds": ["c1", "c2"],
      "organizer": { "name": "Padel Club Example", "type": "club", "url": "https://padelclubexample.nl" },
      "participants": { "registration": "individual", "min": 8, "max": 24, "current": 17 },
      "prices": [
        { "label": "Member", "amount": 0, "currency": "EUR", "requires": [{ "venueId": "1" }] },
        { "label": "Guest", "amount": 7.50, "currency": "EUR" }
      ],
      "url": "https://padelclubexample.nl/social-night",
      "publisher": "Example Booking System",
      "updated": "2026-09-01T08:00:00Z" }
  ]
}
```

No `minLevel` or `maxLevel`: all levels are welcome. No `ratedIn`: social play counts for no rating.
No `directBooking`: players sign up at the club's page.

### 6.2 A free court time

`GET /activities?type=play&venueId=1&date=2026-11-20&fromTime=19:00&toTime=20:00`:

```json
{
  "acceptedIssuers": ["knltb.nl", "email"],
  "venues": [ { "id": "1", "…": "…" } ],
  "activities": [
    { "id": "play-1-c2-2026-11-20T19:00-90", "type": "play", "name": "Court 2",
      "sport": "padel",
      "startDate": "2026-11-20T19:00:00+01:00", "durationMinutes": 90,
      "timeZone": "Europe/Amsterdam", "status": "open",
      "venueId": "1", "resourceIds": ["c2"],
      "participants": { "registration": "individual", "max": 4 },
      "prices": [{ "amount": 32, "currency": "EUR", "per": "booking" }],
      "directBooking": true,
      "cancelBeforeHours": 24,
      "refundIfResold": true,
      "url": "https://padelclubexample.nl/book?court=2&start=1900",
      "publisher": "Example Booking System",
      "updated": "2026-11-20T16:42:00Z" }
  ],
  "checked": "2026-11-20T16:42:00Z"
}
```

The court can be cancelled free of charge until 24 hours before; after that, the player is refunded
only if the court is booked again.

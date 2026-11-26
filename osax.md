# OSAX — Open Sports Activity Exchange — v0.3 draft

Working name; home: **osax.org**.

A standard for sharing sport activities (free courts to play, matches, events, tournaments, leagues,
lessons, courses) between clubs, platforms, federations and aggregators.

A player looks for two things: an activity **nearby**, and one at their **play level**. So OSAX ties
every activity to a venue (Activities interface §2) and can give it a level range in the rating
systems players already have (§3.2). A search asks for both.

### Typical applications

- **"What's on near me, at my level"** — a player app or website shows matches, events and
  tournaments from many clubs and platforms in one list: "tennis tonight within 10 km, for a WTN
  18".
- **Filling open matches** — a platform shows another platform's open matches with places left at
  the player's level, so fewer matches go unplayed.
- **Federation calendar** — a federation shows every tournament and league it sanctions, whichever
  system the organizing club uses, and players see which ones match their rating.
- **Federation member benefits** — members of a national federation see the activities open only to
  members, and their member prices, on any platform that shows them. The player proves membership by
  signing in with the federation.
- **National rating and ranking** — a federation retrieves the results of club matches played by its
  members, identified by its own player IDs, and rates or ranks all its players, not only those who
  enter official tournaments.
- **Club website** — a club shows its own activities from its booking or league system on its own
  site, without copying them by hand.
- **Free courts and tee times** — a booking system offers its free courts as `play` activities to
  player apps, which book them directly and take the payment, or send players back to book.
- **Coach listings** — a coach publishes lessons and courses once, with the levels they suit, and
  appears in every app that reads OSAX.
- **Tourism** — a holiday or travel site shows where visitors can play during their stay, at their
  own level, using the rating they have at home.
- **Calendar subscription** — a player subscribes to a club's activities in their phone calendar via
  the `.ics` export.

### Standards used

**Design goal: little work to implement.** Plain JSON, flat fields, short names. Where a well-known
standard covers something, OSAX uses it — and nothing more:

| For | Standard |
|---|---|
| Dates and times | ISO 8601 with offset; IANA time zone names |
| Recurrence | iCalendar RRULE (RFC 5545) — one string, libraries in every language |
| Country | ISO 3166-1 alpha-2 (`NL`, `GB`) |
| Currency | ISO 4217 (`EUR`) |
| Field names | Borrowed from schema.org where one exists (`name`, `startDate`, `organizer`…) |
| Calendar export | iCalendar `.ics` |
| Sign-in | OpenID Connect, with its discovery at `/.well-known/openid-configuration` |

OSAX is not tied to OpenActive. A publisher that also wants OpenActive compatibility maps the sport
names to OpenActive's activity IDs in its own export.

## 1. The interfaces

OSAX consists of three technical interfaces. A system implements the ones that fit what it holds and
what it wants to allow.

| Interface | What it does | Who offers it |
|---|---|---|
| [Activities](osax-activities.md) | Publish free court times, matches, events, tournaments, leagues, lessons and courses, with their venues and courts; others retrieve, cache and filter them | Booking systems, league and tournament platforms, federations |
| [Booking](osax-booking.md) | Optional: let a client get personal prices, book players into an activity and take the payment itself, and cancel | Booking systems and other publishers that let others book their venues' courts and activities, per activity |
| [Identity and membership](osax-identity.md) | Let a player sign in with an account they already have (OpenID Connect) and, with their consent, check their club and federation memberships | Systems that hold player accounts and memberships |

Many clubs will share activities but not allow booking through another platform: they implement
Activities only.

A club takes part by using a system that supports these interfaces.

## 2. Access

OSAX standardizes **how** data is exchanged, not **who** gets it. Every request needs an API key
(for identity: a registered client), and each publisher decides who receives one and what it can
see. A booking system may share its open matches with one platform and not with another, or not
share them at all. Implementing OSAX does not oblige a publisher to open its data to anyone.

A consumer may show what it receives to its own users, but may not pass it on to another system,
through its own OSAX API or otherwise, without permission from the source. Rebroadcast data would
reach parties the source never gave a key to.

## 3. Shared definitions

### 3.1 Sports

Lowercase English names, words joined by `-`, for example `tennis`, `padel`, `table-tennis`. The
list of sports is kept by osax.org in a separate document, so it can grow without a new version of
this specification: [osax-lists.md](osax-lists.md)

A sport is listed only when it has an international federation and **at least one listed rating
system** (§3.2). To add a sport, open an issue at [github.com/osax-org/spec](https://github.com/osax-org/spec/issues)
with its proposed name, its international federation and the rating system that covers it. Names are fixed once published.

### 3.2 Levels

**Why OSAX requires rating systems and percentile tables.**

1. **A rating makes an activity relevant.** An activity is only worth showing a player when it is
   nearby *and* at their level. Without a rating, systems can exchange activities but not tell which
   ones fit the player. That is why a sport is only listed with at least one rating system.
2. **A percentile table makes a rating convertible.** Players and activities are rated in different
   systems, and a WTN 18 means nothing to a ClubMatch player. A percentile table says what share of
   players sits at or below each level, so a level in one system can be converted to the level with
   the same percentile in another. Without the table there is no conversion, which is why a rating
   system is only listed with one.

OSAX itself publishes no conversions; each implementer converts with the tables.

**Writing a level.** A level is the rating system's short name and a value, joined by `:`, for
example `wtn:13`. An activity gives its range as `minLevel` and `maxLevel`, one entry per rating
system the publisher uses:

```json
"minLevel": ["wtn:13"],
"maxLevel": ["wtn:20"]
```

`minLevel` and `maxLevel` are the numeric lower and upper bound, whatever the direction: on WTN,
where lower is better, `wtn:13` to `wtn:20` still means 13 to 20. Filtering works the same for every
system; direction only matters for display. The activity's `sport` says which sport the level is in;
a rating system covering several sports rates each one separately. An activity without `minLevel`
and `maxLevel` welcomes all levels.

**Short names.** Lowercase, for example `clubmatch`, `wtn`, `knltb`, `squashlevels`, `handicap`. The
rating systems are kept in the same separate document as the sports.

**Adding a rating system.** Open an issue at
[github.com/osax-org/spec](https://github.com/osax-org/spec/issues). A rating system is listed only when:

1. **It has a percentile table:** at least 5 levels, and for each level the percentile of players
   with that rating or lower. A system running from 0 to 10 gives the percentile up to 1, up to 2,
   up to 3 and so on, up to 10. Without the table there is no conversion, so no listing.
2. **At least 1,000 players have been assigned a rating** in it.
3. **Its provider offers an OSAX-compliant API** (Activities interface) that lists activities with
   levels in it.

The issue also states the short name, the sports it covers, its range and its direction
(whether higher or lower is better).

### 3.3 Categories

`category` says who plays against whom: one side, `x`, the other side. Each side is a letter for who
may play, followed by the number of players per side: `P` any player, `M` men, `F` women. The number
is left out for singles.

| `category` | Meaning |
|---|---|
| `PxP` | Singles, open to all |
| `MxM` | Men's singles |
| `FxF` | Women's singles |
| `P2xP2` | Doubles, open to all |
| `M2xM2` | Men's doubles |
| `F2xF2` | Women's doubles |
| `MFxMF` | Mixed doubles: one man and one woman per side |
| `P11xP11` | Teams of 11, open to all |
| `M11xM11` | Men's teams of 11 |
| `F11xF11` | Women's teams of 11 |

Any team size follows the same pattern: `M5xM5` for men's five-a-side, `P6xP6` for open volleyball.

### 3.4 Age groups

`ageGroup` is optional and marks a youth activity: `U5`, `U6` and so on up to `U21`, meaning under
that age. The cut-off date is the one the sport's federation uses. An activity without `ageGroup`
has no age limit.

### 3.5 Memberships

A membership can give a player access or a better price: some activities are open to members only,
some give members a discount. Club memberships and federation memberships work the same way. A
membership is named as:

| Membership | Written as | Notes |
|---|---|---|
| Club | `{ "venues": ["padelclubexample.nl"] }` | Venue handles (§3.7, Activities interface §2); a member of any listed venue qualifies. |
| Federation | `{ "federation": "knltb.nl" }` | The national federation's handle (§3.7), where the player signs in to show the membership (§3.6). |

The national federation issues the membership and, optionally, a federation player ID. It can be one
sport's federation (KNLTB for tennis and padel in the Netherlands) or one body for all sports (the
Norwegian Olympic and Paralympic Committee and Confederation of Sports, `idrettsforbundet.no`). The
membership check says which sports it covers.

These appear as `requires` on an activity (who may join) and on a price (who pays that price).
Several entries mean any one of them is enough. Federation membership is optional for players; an
activity that requires it says so, and the player can check their membership with the federation
(Identity and membership interface).

### 3.6 Player IDs

A player is identified across systems by a **full player ID**: the handle (§3.7) of the system that
vouches for the player, a colon, and the player's ID in that system. That system hands the ID over
when the player signs in with it (Identity and membership interface §2): its own player ID, such as
a federation's player number, as the `osax_player_id` claim, or else the OpenID Connect `sub`.
Alone, the ID means something only to that system; with the handle in front, it means the same in
every system.

| Kind | Example | Rule |
|---|---|---|
| Full player ID | `knltb.nl:1234567` | `handle:id`, split at the last colon. |
| Email address | `jan@example.nl` | Written bare; a string containing `@` is an email address. |
| Not identified | `guest` | A player nobody identifies. |

Wherever this specification says "player ID", it means one of these three; where a player is
identified by a system, that is the full player ID.

- **The handle names the system that vouches for the player.** A federation (`knltb.nl`,
  `idrettsforbundet.no`), or a booking system or club system that issues its own player accounts
  (`bookingsystem.nl`). There is no central list: the handle is the lookup.
- **Finding where a player signs in.** The handle is its OpenID Connect issuer:
  `https://{handle}/.well-known/openid-configuration` says where to send the player to sign in. It
  may redirect to another host.
- **An email address is a player ID nobody vouches for.** A publisher that lists `email` in its
  `playerIds` (Activities interface §4.2) accepts players by email address, without identity
  management: the caller sends `jan@example.nl` with the player's name. Because nobody vouches for an
  email address, the player normally gets the non-member price unless the publisher recognises the
  address as one of its own members. The caller shares a player's name and email only with the
  player's consent.
- A publisher lists the player IDs it accepts in `playerIds`: the handles, its own written out, and
  `email` if it accepts email addresses — for example `["bookingsystem.nl", "knltb.nl", "email"]`.
  A caller uses one of those to get personal prices and to book (Booking interface).

### 3.7 Entities and handles

An **entity** is anything OSAX refers to by name: a federation, a platform, a club, an operator, a
venue, an organizer, a system that vouches for players. Each system keeps entities under its own
internal IDs, which mean nothing to another system. So OSAX identifies every entity by its
**handle**: the shortest official URL where information about the entity can be found, typically
its website. A handle is not unique or permanent: two systems may know the same entity under
different handles, and a handle changes when the entity's website does. But a handle is a URL, so a
system can follow it and find out which entity is behind it (Resolving a handle, below), and map another
system's handle to its own record once.

Handles are used for venues (`venues`, Activities interface §2), organizers and federations
(`organizer`, `federation`), memberships (§3.5), player IDs (§3.6) and a club's discovery file
(§4.1).

#### Examples

| Entity | Kind | Handle |
|---|---|---|
| Deutscher Tennis Bund (Germany) | National federation | `dtb-tennis.de` |
| Lawn Tennis Association (UK) | National federation | `lta.org.uk` |
| USTA (US) | National federation | `usta.com` |
| Tennis Vlaanderen (Belgium) | Regional federation | `tennisvlaanderen.be` |
| Norwegian Olympic and Paralympic Committee and Confederation of Sports | National federation for all sports | `idrettsforbundet.no` |
| ClubMatch | League and event platform | `clubmatch.org` |
| Example Booking System (Germany) | Booking system that issues its own player accounts | `buchungssystem.de` |
| Frans Otten Stadion (Netherlands) | Single venue with its own website | `fransotten.nl` |
| Oslo Padel Example (Norway) | Club and its single venue, under one website | `oslopadelexample.no` |
| Racket Gyms Example (UK) | Operator of many venues | `racketgymsexample.co.uk` |
| Racket Gyms Manchester, Racket Gyms Leeds (UK) | Two venues of one operator, each with a page on the operator's website | `racketgymsexample.co.uk/manchester`, `racketgymsexample.co.uk/leeds` |
| Sam, a coach at Austin Pickleball Example (US) | Organizer with a page on the club's website | `austinpickleballexample.com/coaches/sam` |

A club and its venue share one handle when the club has one venue and one website. An operator and
each of its venues have separate handles: the operator's domain, and a page per venue on it.

#### Writing a handle

- **Only what is needed:** the URL keeps every part needed to reach the entity's page, and drops
  every part at the start or end that is not: the protocol, a leading `www.`, a trailing `/`, and
  any query (`?…`), fragment (`#…`) or page name that the page does not need.
  `https://www.fransotten.nl/` is written `fransotten.nl`; `https://www.racketgymsexample.co.uk/manchester?_uid=284678`
  is written `racketgymsexample.co.uk/manchester`. A site that shows a venue only through its query keeps it:
  `clubsite.nl/venue?id=12`. The domain is in lowercase. A client opening a handle prefixes
  `https://`.
- **Shortest:** only the domain when the entity has a website of its own. A path only when it does
  not (a venue of a multi-venue operator, a coach), and then the path of the entity's own real,
  working page, never a made-up tag.
- **No handle, no address.** An entity without an official web page of its own, such as a small
  venue without a website, has no handle and cannot be addressed in OSAX.
- **Comparing:** two handles that match exactly, as written above, are the same entity. Two that
  differ may still be the same entity; only resolving tells. For a multi-venue operator, a match on
  the domain alone is not a match: `racketgymsexample.co.uk` is the operator, not its venue in Manchester.
- **A handle changes** when the entity's website changes. The publisher then uses the new handle and
  keeps resolving the old one.

#### Resolving a handle

A system maps a handle it does not know to its own record by following the URL behind it: its
redirects, the page's canonical URL, or the page itself. It keeps the mapping, so it resolves each
handle once.

When a request names an entity by a handle the server does not know, the server MAY resolve it this
way. If it cannot map it to a handle it knows, it answers `404`:

```json
{ "error": "unknown-handle", "message": "No venue known as fransottenpadel.nl." }
```

If it can, it answers the request and includes the mapping in the response, so the client can store
the resolved handle:

```json
"resolved": { "fransottenpadel.nl": "fransotten.nl" }
```

## 4. Base URL and responses

A publisher's interfaces live under one base URL, for example
`https://squashcity.bookingsystem.nl/osax/v1`. All paths in this specification are relative to it:
`/venues`, `/activities`, `/bookings`, `/me/memberships`. How a client gets a publisher's base URL
and API key is part of setting up the integration, not of OSAX: a booking system with a subdomain per
club, for example, builds the base URL from the club's subdomain.

Every request carries the API key in the `X-Api-Key` header. A publisher answers:

| Status | Meaning |
|---|---|
| `200` (or as the interface says) | An OSAX response, as the interface specifies. |
| `401` | No key, or the key is not valid. |
| `403` | The key is valid but gives no access to this interface or this data. |
| `501` | The publisher does not offer this interface. |

`404` keeps its usual meaning: the venue, activity or booking asked for does not exist.

**Errors.** Every error is an HTTP status with `{ "error": "<code>", "message": "<text>" }`. `error` is
a short code a client can act on; `message` explains it for the developer. A publisher may use its
own codes. `400` is the catch-all for a request the publisher will not serve; `500` for its own
failure. For example, free courts asked for over several days:

```
400
{ "error": "range-too-large", "message": "Free courts are returned for one day at a time." }
```

### 4.1 A club's discovery file

A club can say where its activities are published with a file on its own website, at
`https://{club handle}/.well-known/osax.json`:

```json
{
  "baseUrl": "https://squashcity.bookingsystem.nl/osax/v1",
  "publisher": "bookingsystem.nl",
  "venues": ["squashcity.nl"]
}
```

`baseUrl` is required. `publisher` is the publishing system's handle (§3.7). `venues` gives the
handles of this club's venues at that base URL; leave it out when the base URL serves only this club. The file is
optional: many club websites cannot serve files under `/.well-known/`, and a booking system may set
it up for its clubs. It carries no key; access is still arranged with the publisher (§2).

## 5. Members

osax.org keeps a public list of members: organisations that implement OSAX, with each one's name,
website, role (publisher, client or both) and base URL.

- **Implementing is free.** Using the specification needs no fee or registration at osax.org. Access
  to a publisher's data is between you and that publisher (API key, §2).
- **Membership costs a one-off fee of EUR 1,000 or USD 1,000.** The fee pays for developing and
  maintaining the specification, the sport list and the rating system list.
- **Membership gives no extra say in the specification.** Decisions are open to every contributor,
  member or not (§6).
- **To become a member, email info@osax.org.** A publisher's interfaces (§4) must be reachable.

## 6. Governance

The specification is developed in the open at [github.com/osax-org/spec](https://github.com/osax-org/spec),
under the Community Specification Governance Policy 1.0, which the
[Contributor License Agreement](CLA.md) adopts.

- **Anyone can contribute,** free of charge: questions, gaps and proposals as issues, changes to the
  text as pull requests. A contributor is a participant in the sense of that policy.
- **Decisions are made by consensus of the participants,** including the content and releases of
  the specification. Full agreement is preferred but not required.
- **The maintainers** review and merge pull requests and determine when consensus is reached. New
  maintainers are chosen with the participants' approval. A decision can be appealed through an issue.

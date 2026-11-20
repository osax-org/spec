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
| [Activities](https://sportconnexions.com/doc/osax-activities-interface-v0-3-draft-5w5xdrzjs4) | Publish free court times, matches, events, tournaments, leagues, lessons and courses, with their venues and courts; others retrieve, cache and filter them | Booking systems, league and tournament platforms, federations |
| [Booking](https://sportconnexions.com/doc/osax-booking-interface-v0-3-draft-5mdcnfdkrz) | Optional: let a client get personal prices, book players into an activity and take the payment itself, and cancel | Publishers willing to let others book, per activity |
| [Identity and membership](https://sportconnexions.com/doc/osax-identity-and-membership-interface-v-qxjzpxrphp) | Let a player sign in with an account they already have (OpenID Connect) and, with their consent, check their club and federation memberships | Systems that hold player accounts and memberships |

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
this specification: https://sportconnexions.com/doc/osax-sports-and-rating-systems-pnc34r7q24

A sport is listed only when it has an international federation and **at least one listed rating
system** (§3.2). To add a sport, email info@osax.org with its proposed name, its international
federation and the rating system that covers it. Names are fixed once published.

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

**Adding a rating system.** Email info@osax.org. A rating system is listed only when:

1. **It has a percentile table:** at least 5 levels, and for each level the percentile of players
   with that rating or lower. A system running from 0 to 10 gives the percentile up to 1, up to 2,
   up to 3 and so on, up to 10. Without the table there is no conversion, so no listing.
2. **At least 1,000 players have been assigned a rating** in it.
3. **Its provider offers an OSAX-compliant API** (Activities interface) that lists activities with
   levels in it.

The submission also states the short name, the sports it covers, its range and its direction
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
| Club | `{ "venueId": "4521" }` | The publisher's venue id (Activities interface, §2). |
| Federation | `{ "federation": "knltb.nl" }` | The national federation's web domain. Its discovery file (§4) is at that domain, so a client knows where the player signs in to show the membership. |

The national federation issues the membership and, optionally, a federation player ID. It can be one
sport's federation (KNLTB for tennis and padel in the Netherlands) or one body for all sports (the
Norwegian Olympic and Paralympic Committee and Confederation of Sports, `idrettsforbundet.no`). The
membership check says which sports it covers.

These appear as `requires` on an activity (who may join) and on a price (who pays that price).
Several entries mean any one of them is enough. Federation membership is optional for players; an
activity that requires it says so, and the player can check their membership with the federation
(Identity and membership interface).

### 3.6 Player ID issuers

A player is identified across systems by an ID from an **issuer**: the system that vouches for that
player. A player is written as `issuer:id`, for example `knltb.nl:1234567`.

- **An issuer is named by its web domain.** A federation (`knltb.nl`, `idrettsforbundet.no`), or a
  booking system or club system that issues its own player accounts (`bookingsystem.nl`). There is
  no central list: the domain is the lookup.
- **Finding where a player signs in.** The issuer's discovery file at
  `https://{domain}/.well-known/osax.json` (§4) gives in `identity` its OpenID Connect issuer; that
  issuer's `/.well-known/openid-configuration` says where to send the player to sign in.
- **`email` is a special issuer.** A publisher that lists `email` in its `acceptedIssuers`
  (Activities interface §4.2) accepts players by email address, without identity management: the
  caller sends `email:jan@example.nl` with the player's name. Nobody vouches for an email address,
  so the player normally gets the non-member price unless the publisher recognises the address as
  one of its own members. The caller shares a player's name and email only with the player's
  consent.
- A publisher lists the issuers it accepts in `acceptedIssuers`; a caller uses one of those to get
  personal prices and to book (Booking interface).

## 4. Discovery

A publisher or issuer puts a small file at `https://{host}/.well-known/osax.json`:

```json
{
  "publisher": "ClubMatch",
  "url": "https://clubmatch.org",
  "version": "0.3",
  "activities": "https://clubmatch.org/osax/v1/activities",
  "venues": "https://clubmatch.org/osax/v1/venues",
  "bookings": "https://clubmatch.org/osax/v1/bookings",
  "identity": "https://clubmatch.org",
  "memberships": "https://clubmatch.org/osax/v1/me/memberships",
  "license": "CC-BY-4.0",
  "terms": "https://clubmatch.org/osax/terms",
  "apiKeys": "mailto:info@clubmatch.org",
  "contact": "https://clubmatch.org/contact"
}
```

Each interface a publisher does not offer is left out: `activities` and `venues` (Activities),
`bookings` (Booking), `identity` and `memberships` (Identity and membership; `identity` is the
OpenID Connect issuer). A federation that only issues player IDs and memberships publishes just
`identity` and `memberships`.

`license` is the publisher's choice (an SPDX identifier or a URL); CC BY 4.0 is the suggested
default. `apiKeys` says where to request a key.

## 5. Implementers

osax.org keeps a public list of organisations that implement OSAX, with each one's name, website,
role (publisher, client or both) and discovery file.

- **Implementing is free.** Using the specification needs no fee or registration at osax.org. Access
  to a publisher's data is between you and that publisher (API key, §2).
- **Being listed costs a one-off fee of EUR 1,000 or USD 1,000.** The fee pays for developing and
  maintaining the specification, the sport list and the rating system list.
- **To join, email info@osax.org.** A publisher's discovery file (§4) must be reachable.

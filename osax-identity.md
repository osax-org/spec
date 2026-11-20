# OSAX — Identity and membership interface — v0.3 draft

Part of OSAX, the Open Sports Activity Exchange. The overview — access rules, sports, levels,
discovery — is in [osax.md](osax.md)

This interface lets a player use the account they already have in one system to sign in to
another, and lets that other system check, with the player's consent, whether the player is a
member of a club or a national federation. A platform can then show member prices, open
members-only activities, or skip asking for a membership number.

It is built on **OpenID Connect** (OIDC), the identity layer on OAuth 2.0. OSAX adds one scope and
one endpoint; everything else is standard OIDC, with libraries in every language.

Identity and membership are kept apart. Signing in says who the player is. Membership is not a
claim in the token: it is a separate call from the client's server to the provider (§3).

## 1. Roles

| Role | Who | Does |
|---|---|---|
| Identity provider | A system where the player has an account and memberships are known: a federation, a club or booking system, a platform | Signs the player in, issues the ID token, answers membership checks |
| Client | A platform or competition organizer that wants to know who the player is, or whether they are a member | Sends the player to the provider to sign in, then asks the provider about that player's memberships, server to server |
| Player | The person | Signs in at the provider and decides what the client may see |

## 2. Signing in

Standard OIDC authorization code flow with PKCE.

**Set up once per provider:**

1. The client registers with the provider and gets a client ID and a secret. Registration is on
   the provider's terms, like an API key (Overview §2).
2. The client reads the provider's OIDC settings at `{issuer}/.well-known/openid-configuration`.
   The `issuer` is the provider's base address and is in its OSAX discovery file (Overview §4).

**At each sign-in:**

1. The client sends the player to the provider's authorization endpoint with its client ID, its
   return URL, the scopes it needs and a `state` value.
2. The player signs in at the provider and sees what the client asks for; they allow or refuse.
3. The player comes back to the return URL with the same `state` and an authorization code that
   works once.
4. The client's server exchanges the code and its secret for the ID token. No token passes
   through the browser.

**Player identity.** A player is identified by the pair `iss` + `sub` from the ID token: the
provider's issuer and its stable id for that player. A client stores that pair against its own
account as the key that links the two, not the email address, which can change. `sub` means
nothing outside that provider.

### 2.1 Scopes

| Scope | Gives the client |
|---|---|
| `openid` | Required. The player's `sub`. |
| `profile` | The player's name. |
| `email` | The player's email address. |
| `osax.levels` | The player's own levels, as `osax_levels` in the ID token (§2.2). |

A client asks only for what it needs. Membership needs no scope of its own: signing in to a client
is what lets that client check the player's memberships (§3.1). The player can withdraw the
client's access at the provider at any time.

### 2.2 Levels claim

With `osax.levels`, the ID token carries the player's levels, one per sport and rating system,
written as in Overview §3.2:

```json
"osax_levels": [
  { "sport": "padel", "level": "clubmatch:4.2" },
  { "sport": "tennis", "level": "wtn:18.5" }
]
```

These are the player's own levels, shared by the player. They are never published with activities.

## 3. Checking membership

A client's server asks the provider for a player's memberships, by the player's `sub`, with the
client's own ID and secret. The player is not redirected and no player token or refresh token is
involved:

```
GET {host}/osax/v1/memberships?sub=8f3a61
Authorization: Basic {client ID:secret}
```

A federation answers with one entry per club the player is a member through:

```json
{
  "memberships": [
    { "federation": "knltb.nl", "federationName": "KNLTB",
      "clubNumber": "12345", "clubName": "TV Example", "playerId": "12345678",
      "status": "active", "type": "senior", "sports": ["tennis", "padel"],
      "validUntil": "2026-12-31" }
  ],
  "checked": "2026-11-20T16:42:00Z"
}
```

A club or booking system answers the same call for its own members, with its venue in place of
the club number:

```json
{
  "memberships": [
    { "venueId": "4521", "venueName": "Padel Club Example",
      "status": "active", "type": "Adult", "sports": ["padel"],
      "validFrom": "2026-01-01", "validUntil": "2026-12-31" }
  ],
  "checked": "2026-11-20T16:42:00Z"
}
```

| Field | R | Notes |
|---|---|---|
| `federation`, `federationName` | R from a federation | The national federation's web domain, as in `requires` (Overview §3.5). |
| `clubNumber`, `clubName` | | From a federation: its own code and name for the club the membership runs through. Left out for a licence the federation issued to the player directly. |
| `venueId`, `venueName` | R from a club or booking system | The provider's venue id, as in the Activities interface (§2), so the client can match it to the venue it already mapped. |
| `playerId` | | The ID the federation issued to the player, if it issues one. Optional for the federation and for the player. |
| `status` | R | `active`, `pending` (applied, not yet accepted), `suspended` (still a member but not allowed to play, for example unpaid dues or a ban), `expired`. |
| `type` | | The club's or federation's own name for the membership: `senior`, `junior`, `social`, `Off-peak`… |
| `sports` | | Sports (Overview §3.1) the membership covers; leave out = all the club or federation covers. |
| `validFrom`, `validUntil` | | Dates, local to the club or venue. |
| `checked` | R | UTC; when the answer was produced. |

To check one club only, add `clubNumber=12345` at a federation or `venueId=4521` at a club or
booking system.

| Answer | Means |
|---|---|
| `200` with entries | The player's memberships, one entry per club or venue. |
| `200` with an empty list | The player is known and has no membership (at that club, when one was asked for). |
| `404` | Player not found: the `sub` is unknown, or has not signed in to this client (§3.1). |

**Federation memberships.** Membership of a federation usually runs through a club, so the
federation's answer names the club or clubs. Some federations also issue a licence to the player
directly; that is an entry without `clubNumber`. A national federation that offers this interface
answers for its own members: the player signs in at the federation (found through the discovery
file at its domain) and the client checks there. A club or booking system may also return the
federation membership it knows, but the federation is the source.

### 3.1 Rules

- **Only for a player who signed in.** A provider answers only for a `sub` that has signed in to
  this client, never by name, email or membership number. A provider offers no lookup of other
  players. When the player withdraws the client's access at the provider, the provider stops
  answering for that `sub`.
- **Check when it matters.** Membership changes; check it at the moment it counts (showing a
  member price, joining a members-only activity) rather than storing it. The check needs no
  player present, so a client can repeat it whenever it counts.
- **Not passed on.** A client uses a membership answer for that player only and passes it to no
  one else (Overview §2).
- **The provider is the source.** If memberships are held in more than one system, the club
  decides which one answers; a client trusts the provider the player signed in with.

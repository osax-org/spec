# OSAX — Sports, rating systems and court properties

The lists that the OSAX specification refers to. Kept by osax.org; changes are made here, without a
new version of the specification.

## Why these lists exist

1. **A rating makes an activity relevant.** Systems exchange activities so players find ones nearby
   *and* at their level. Without a rating, there is no telling which activities fit a player. So a
   sport is only listed with at least one rating system.
2. **A percentile table makes a rating convertible.** A level in one rating system means nothing in
   another until both say what share of players sits at or below each level. Matching those
   percentiles converts one system's level into the other's. So a rating system is only listed with
   a percentile table.
3. **Court properties tell a player what they will play on.** Indoor or outdoor, the surface and the
   walls matter to players choosing a court or event, and they differ per sport. So the values are
   listed here and can grow per sport.

## Sports

A sport is listed only when it has an international federation and **at least one listed rating
system**. To submit one, open an issue at
[github.com/osax-org/spec](https://github.com/osax-org/spec/issues) with its proposed name, its
international federation and the rating system that covers it.

| `sport` | International federation |
|---|---|
| `padel` | International Padel Federation (FIP) |
| `tennis` | International Tennis Federation (ITF) |
| `squash` | World Squash Federation (WSF) |
| `badminton` | Badminton World Federation (BWF) |
| `table-tennis` | International Table Tennis Federation (ITTF) |
| `racquetball` | International Racquetball Federation (IRF) |
| `pickleball` | No single body yet: several federations claim the role |
| `golf` | International Golf Federation (IGF) |
| `pool` | World Pool-Billiard Association (WPA) |
| `snooker` | World Professional Billiards and Snooker Association (WPBSA) |
| `billiards` | Union Mondiale de Billard (UMB), carom billiards |
| `darts` | World Darts Federation (WDF) |

Names are fixed once published. Where a name could mean two sports, the federation settles it:
`padel` = padel (FIP), not paddle tennis; `billiards` = carom billiards (UMB).

## Rating systems

A rating system is listed only when its organisation provides a **percentile table**: for at least 5
levels, the share of rated players at that level or lower. The system also needs at least 1,000
rated players, and an OSAX-compliant API from its provider that lists activities with levels in it.
To submit one, open an issue at [github.com/osax-org/spec](https://github.com/osax-org/spec/issues).

### ClubMatch

- **Short name:** `clubmatch`
- **Provider:** Sportconnexions
- **Sports:** padel, tennis, squash, badminton, table-tennis, racquetball, pickleball, golf, pool,
  snooker, billiards, darts
- **Range:** 0.1–10.0
- **Better is:** higher

| Level | Players at this level or lower |
|---|---|
| up to 1 | 1.2% |
| up to 2 | 6.7% |
| up to 3 | 22.7% |
| up to 4 | 50.0% |
| up to 5 | 77.3% |
| up to 6 | 93.3% |
| up to 7 | 98.8% |
| up to 8 | 99.87% |
| up to 9 | 99.99% |
| up to 10 | 100% |

Example: a ClubMatch 4 is at the 50th percentile; in another listed system, the level at its 50th
percentile is the equivalent.

## Court properties

Values for a resource's `roof`, `surface` and `walls` (Activities interface §2.1). Each is
optional on a resource. To add a value, open an issue at
[github.com/osax-org/spec](https://github.com/osax-org/spec/issues) with the sport and a short
description.
Values are lowercase, words joined by `-`, and fixed once published.

### Roof types (all sports)

| `roof` | Meaning |
|---|---|
| `indoor` | In a closed building. |
| `covered` | Roof overhead, open sides. |
| `outdoor` | No roof. |

### Surfaces

| `surface` | Sports |
|---|---|
| `clay` | tennis |
| `artificial-clay` | tennis |
| `grass` | tennis, golf |
| `artificial-grass` | tennis, padel |
| `hard` | tennis, pickleball |
| `carpet` | tennis, badminton |
| `wood` | squash, badminton, racquetball, pickleball |
| `synthetic` | badminton, pickleball, table-tennis |
| `glass` | squash |
| `other` | any |

### Walls

| `walls` | Sports |
|---|---|
| `glass` | padel, squash (all walls glass) |
| `concrete` | padel |
| `plaster` | squash, racquetball |
| `glass-back` | squash, racquetball (glass back wall, others solid) |
| `panel` | squash, racquetball (prefabricated panels) |
| `other` | any |

Leave `walls` out for sports without walls in play.

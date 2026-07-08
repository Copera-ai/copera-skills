# Bookings & Meetings

Scheduling data: bookings (individual scheduled events), booking types (the event templates), and meeting notes (recorded/summarized meetings). All read-only.

**Auth caveat:** these are PAT-scoped (`cp_pat_…`) and resolve against the token user. See `auth-and-profiles.md`.

## List bookings

```bash
copera bookings list --json \
  [--status PENDING|CONFIRMED|DECLINED|CANCELLED|COMPLETED] \
  [--type <bookingTypeId>] \
  [--from <ISO-8601>] [--to <ISO-8601>] \
  [--limit 1-100] [--cursor <keyset>]
```

Newest first. Filters:

- `--status` — one of `PENDING`, `CONFIRMED`, `DECLINED`, `CANCELLED`, `COMPLETED`.
- `--type` — a booking type ID (get one from `booking-types list`).
- `--from` / `--to` — bound the booking **start** instant (ISO-8601), at/after and at/before respectively.
- `--limit` — page size, 1–100.

**Pagination is keyset-based, not offset.** The response carries a `nextCursor`; pass its value back via `--cursor` to fetch the next page. When `nextCursor` is empty/absent, you've reached the end.

## Get a booking

```bash
copera bookings get <booking-id> --json
# → single booking object
```

`<booking-id>` is a required positional. Human output surfaces `ID / Status / Type / Start / End / Duration / Timezone / Booker / Location / Hosts / Guests`.

## List booking types

```bash
copera booking-types list --json
# → the workspace's booking / event types
```

No flags. Human output: `ID / Title / Slug / Kind / Duration / Hidden / Active`. Use these IDs with `bookings list --type`.

## List meeting notes

```bash
copera meeting-notes list --json [--query <text>] [--limit 1-100] [--before <ISO date>]
```

Newest first, scoped to notes the token user owns or is a shared participant of. `--query` searches note titles and typed notes. `--limit` is 1–100. `--before` (an ISO date) pages **back in time** — pass the oldest note's start date to get the previous page.

## Get a meeting note

```bash
copera meeting-notes get <note-id> --json
copera meeting-notes get <note-id> --transcript --json
```

`<note-id>` is a required positional. Returns the note's summary. Add `--transcript` to include the full transcript segments; human output renders each segment as `[Speaker] text`. Transcripts can be long — omit `--transcript` when you only need the summary.

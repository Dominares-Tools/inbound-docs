# Changelog

## 1.1.3 (2026-10-06)

Tests only. No code change.

- The tests no longer import what they don't use, and your copy's README test file drops a helper nothing in it calls. Version number only otherwise.

## 1.1.2 (2026-10-05)

Tests only. No code change.

- Buyer test suite restored; no code change. More of the README-checking tests now ship and run in your copy. Version number only otherwise.

## 1.1.1

Fixes from review of 1.1.0. The receiving side is unchanged.

- **Erasure told the truth only in part.** `ReadState.forget_keys` (and so `Shelf.forget_threads`)
  rewrote a damaged `read_state.json` as blank, deleting every other thread's marks and reporting
  `read: 0`. It now raises `StoreUnreadable` and leaves the file as it was. The follow-up log is
  kept by default (it is what stops a second follow-up going out) and is erased only with
  `forget_threads(keys, followups=True)`, backed by the new `FollowupLog.forget_keys`. The docs say so.
- **The board counted a thread twice.** The `waiting` source is now "needs a reply and has no
  draft", so a thread with a draft is counted once, under `drafts`. `total` is the number of
  distinct threads from the two built-in sources (it was `waiting + drafts` with overlap).
- **Thread keys.** A row with no `thread_id` now uses the first of `references`, else `in_reply_to`,
  so a reply you stored without a `thread_id` lands on the thread it answers instead of a phantom
  `subj:` thread beside it. The README states that every message of a thread must reach the same key tier.
- **`inbound thread tag`** exits 2 and names tags it does not know instead of printing "tagged"
  (nothing is stored when none is known; known ones are kept when only some are not).
- **`inbound thread rows` / `rollup`** name an unreadable state file on stderr before showing it as empty.
- **`inbound thread key`** and **`draft --file`** on a missing file print `inbound: ...` and exit 2
  instead of a traceback; `show` cleans its key like every store.
- README: the custom-store contract names the exception a strict read must raise; the `.lock`
  files are mentioned. The brand test now skips any virtual environment inside the repo.

## 1.1.0

Shared-inbox thread state. The receiving side (Worker, signing, parsing, handler) is unchanged.

New, all standard library:

- `inbound.Shelf`: folder, tags, one draft, read/unread and done, and a follow-up log for every
  thread, behind one object. `shelf.rows(mail_log)` returns one row per thread, newest first, with
  **needs_reply derived from the log** (never stored), each store read once per call.
- `inbound.thread_key` / `thread_key_for`: one key per thread (`cid:`, `msg:` or `subj:`), used by
  every store.
- `Folders` / `FolderSet`: one folder per thread; retired names fold to the inbox; a classifier's
  `only_if_auto` write never overwrites a person's filing; an unreadable store reads as the inbox
  and refuses writes.
- `Tags` / `TagSet`: many tags per thread in declared order, residual tags ("general", "ignore")
  never beside a substantive one, "no tags" stored as an answer.
- `Drafts`: at most one unsent reply per thread, kept apart from the mail log; `save_error` keeps
  the reply that was waiting; `restore` for undo.
- `ReadState`: read/unread, done (who and when; covers a thread only until new mail), a banner
  dismissed by stamp. An unreadable or evicted mark reads as unread.
- `needs_reply`, `thread_facts`, `rollup`: the derivation (messages ordered by instant, so a reply
  written in another UTC offset is not placed before the message it answers) and a counts-only
  "needs a human" board whose sources fail safe to 0.
- `decide_followup`, `FollowupLog`, `render_transcript`, `strip_internal`: decide whether to email a
  conversation to the person who had it (no address, one-sided chat, repeat sends and internal
  `[bracketed]` text are all refused or stripped), and remember a send.
- `render_chat`, `render_call`, `render_email_thread`, `reference_plain`, `reference_html`,
  `rich_text_html`: a conversation as plain text and email-safe HTML, for quoting to staff.
- `clean_addr_list`, `offered_recipients`, `default_body_for`: recipient hygiene that never adds a
  recipient, and prefilled bodies only for purposes you list.
- `JsonFileStore` (locked, atomic read-modify-write, safe across threads and processes on POSIX)
  and the store shape any other store can implement. The statepath tool's `JsonStore` fits.
- `inbound thread` command: `rows`, `rollup`, `file`, `tag`, `draft`, `read`, `unread`, `done`,
  `undone`, `show`, `key`. `examples/maillog.json` is a sample mail log.

Changed:

- The README opens with a one-sentence description and a three-line example you can paste.
- The 1.0.1 changelog entry no longer names the seller.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show inbound`, `inbound.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated to version 1.3.
- One public contact address in the licence and the README.

## 1.0.0 (2026-09-27)

First release.

- `worker/`: a Cloudflare Email Worker with no npm dependencies. Signs each message for an
  inbound address (`<slug>-<token>[+tag]@domain`) with HMAC-SHA256 over
  `timestamp.envelope_to.sha256(raw)` and POSTs it to your app; caps the size; bounces
  with a fixed sentence per status; never follows a redirect (a 3xx is a bounce with a
  log line naming the new location); forwards every other address to a fallback
  mailbox; never ends in an uncaught exception. `wrangler.jsonc` template and a setup runbook.
- `inbound.addresses`: mint, parse, tag and rotate addresses, always within the
  64-character local-part limit; `slugify`; `is_inbound_shaped` to check a human
  mailbox name.
- `inbound.signing`: `sign`, constant-time `verify` with a comma-separated rotation list
  and a ±5 minute window, failing closed when no secret is set; `signed_headers`;
  `rotation_warning`.
- `inbound.parse`: `parse_email` into an `InboundEmail` dataclass (text and HTML bodies,
  addresses, message ids and thread root, attachments, forwards, SPF / DKIM / DMARC
  verdicts read only from a trusted server's header, passing DKIM domains, Gmail
  forwarding confirmation). Never raises.
- `inbound.handler`: `InboundHandler` with `resolve` / `deliver` callbacks, `+tag` routes,
  `Reject(status)` for clean bounces, sync and async entry points (plain callbacks run in
  a worker thread under `handle_async`). A reply that cannot be encoded, or a secret
  that cannot be read, is handled without a crash.
- `inbound.fastapi_router`: optional FastAPI router (extra `fastapi`), cap enforced while
  reading.
- `inbound` command: `secret`, `address`, `parse`, `send`.
- Examples: a standard-library receiver and a FastAPI app.

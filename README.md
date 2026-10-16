# inbound — An email address per account, on your domain

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/inbound**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

Your users forward or BCC mail to an address that belongs to their account. Each message reaches your code verified and parsed. You run no mail server and set nothing up per account. A shared-inbox layer keeps each thread's folder, tags, draft and read state and counts what still needs a reply.

## Get it

Buy inbound at https://dominares.org/tools/inbound: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Cloudflare Email Worker and the Python receiver.

Requirements: The core is the Python standard library. The [fastapi] extra adds the FastAPI router. The Email Worker runs on your own Cloudflare account.

This page documents version 1.1.3. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

Give every account in your app its own email address on your domain, delivered as a signed webhook, and keep each thread's shared-inbox state (folder, tags, draft, unread, needs a reply) beside it.

```bash
inbound thread rows examples/maillog.json
```

The last line reads a small sample mail log and prints one line per thread: which folder it is in,
whether it is unread, whether a draft is waiting, and whether someone still has to answer it.

### What it does

**Receive.** Every account gets its own address on your domain
(`acme-k3v7q2m9x4ab@example.com`, or `acme-k3v7q2m9x4ab+billing@example.com` for a tagged one).
Your users forward, auto-forward, BCC or filter mail to that address from any mailbox
(Gmail, Outlook and Microsoft 365, iCloud, a phone's mail app). Each message reaches your
code as a verified, parsed Python object, with no mail server to run and no per-account
setup anywhere.

**Keep thread state.** Once a message has arrived, a help desk needs more than the message:
which folder the thread is in, what it is about, the reply nobody has sent yet, whether anyone
has looked at it, and whether it is still waiting on an answer. `inbound.Shelf` keeps those five
things in small JSON files and turns your mail log into inbox rows (see
[Thread state](#thread-state-folders-tags-drafts-unread-needs-a-reply)). It is standard library
only and works on any mail log, not just mail received through the Worker.

**Who it is for:** developers of help desks, CRMs, ticketing and ops tools, expense and
invoice inboxes, or any product where "just forward it to us" beats an integration.

### How it works

    sender -> your domain's MX (Cloudflare Email Routing, free)
           -> catch-all rule -> the Worker in worker/ (no npm dependencies)
           -> POST https://your-app/inbound-email   raw MIME + HMAC signature
           -> inbound.InboundHandler: verify -> resolve(address) -> parse -> deliver(account, message)

- **One Worker, no per-account config.** Accounts are identified by the random token in
  the address, so adding or rotating an account never touches Cloudflare.
- **Signed and fresh.** The Worker signs the timestamp, the recipient and a hash of the
  raw message with a shared secret. The receiver checks it in constant time, refuses
  anything older or newer than 5 minutes, and refuses everything while no secret is set.
- **Nothing is silently lost.** If your app answers anything but 2xx, the sender gets a
  bounce with a plain sentence for that status. Mail to your own mailboxes on the same
  domain (`hello@`, `support@`) is forwarded to your inbox untouched.
- **A parsed message, not a blob.** Subject, sender, recipients, thread root, text and
  HTML bodies, attachment metadata (or bytes, if you ask), the original sender of a
  forward, the receiving server's SPF / DKIM / DMARC verdicts, and Gmail's "confirm
  forwarding" email recognised so you can show the user its code.

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

### 60-second quickstart (local, no Cloudflare yet)

    export INBOUND_SECRET="$(inbound secret)"
    python3 examples/stdlib_server.py

It prints `Demo account address: acme-...@example.com`. In a second terminal, use
the same environment, export the same `INBOUND_SECRET`
value, and send a sample message exactly as the Worker would:

    inbound send tests/fixtures/forwarded_gmail.eml \
        --url http://127.0.0.1:8000/inbound-email --to <the demo address>

The server prints one line per delivered message and `inbound send` prints
`200 {"ok": true, ...}`. Try a wrong address (404), or a different `INBOUND_SECRET` (401).
The FastAPI version works the same way:
`python3 -m pip install ".[fastapi]" uvicorn`, then
`uvicorn examples.fastapi_app:app --port 8000`.

When that works, follow worker/README.md to point your domain at the
Worker (about 15 minutes, all in the Cloudflare dashboard plus two `wrangler` commands).

### Use it in your app

```python
from inbound import InboundHandler, Reject, mint, rotate, slugify

# 1. Give an account an address, and store the token (a unique column).
addr = mint(slugify("Acme Widgets"), "example.com")   # Address(slug="acme-widgets", token="k3v7q2m9x4ab", ...)
db.save_inbound_token(account_id, addr.token)
show_to_user(str(addr))                   # acme-widgets-k3v7q2m9x4ab@example.com

# 2. Receive.
def resolve(address):                     # -> account, or None for "unknown recipient"
    account = db.account_by_inbound_token(address.token)
    if account is None or account.slug != address.slug:
        return None
    if account.suspended:
        raise Reject(403, "suspended")    # sender sees "not accepting mail right now"
    return account

def deliver(account, message):            # message is an inbound.InboundEmail
    if db.seen_message(account.id, message.message_id):
        return {"duplicate": True}        # redelivery: accept, store once
    db.store_email(account.id, message.subject, message.from_addr, message.as_text())
    return {"stored": True}               # merged into the JSON reply (a UUID or date becomes a string)

handler = InboundHandler(resolve=resolve, deliver=deliver)
```

Then wire `handler` to a route:

- **FastAPI:** `app.include_router(inbound_router(handler))` with
  `from inbound.fastapi_router import inbound_router`. Callbacks may be `async`; plain
  (non-async) ones run in a worker thread, so a blocking database call does not stall
  other requests. See `examples/fastapi_app.py`.
- **Anything else:** call `handler.handle(raw_body_bytes, request_headers)` (or
  `await handler.handle_async(...)`) and answer with `response.status` and
  `response.json_bytes()`. Headers can be a dict, a list of pairs, a WSGI `environ`, or
  any object with `.get()`. Refuse bodies over `handler.max_bytes` before reading them.
  `examples/stdlib_server.py` does this with `http.server`.

**Rotate an account's address** (for example after it leaks): `new = rotate(old)`, store
`new.token` in place of the old one. The old address bounces as "unknown recipient" at
once.

**Address length.** `<slug>-<token>+<tag>` must fit in 64 characters (the limit on the
part before the `@`); `mint()` raises `AddressError` otherwise. `slugify()` caps a slug
at 51 characters, so a tagless address always fits; leave room if you add tags.

#### Recipes

**Route by tag.** `+tag` needs no setup, so one account can have many addresses:

```python
handler = InboundHandler(resolve=resolve, deliver=deliver,
                         tag_routes={"billing": deliver_billing, "support": deliver_support})
```

Any other tag (and no tag) goes to `deliver`; `message.address.tag` is always available.

**Only trusted senders on a privileged tag.** If `+billing` mail does something powerful,
check the sender's domain and refuse a DMARC failure:

```python
def deliver_billing(account, message):
    if not message.sender_domain_in(account.billing_domains) or message.auth.dmarc_failed:
        raise Reject(403, "untrusted_sender")
    ...
```

**Gmail's forwarding confirmation.** When a user adds your address as a Gmail forwarding
destination, Gmail sends a confirmation email to it. Handle it separately so the user
can finish setup from your UI:

```python
def on_confirmation(account, message):
    c = message.forwarding_confirmation   # .code, .confirm_url (Google hosts only), .forwarding_from
    notify_account(account, f"Gmail asks you to confirm forwarding: code {c.code}")

handler = InboundHandler(resolve=resolve, deliver=deliver, on_forwarding_confirmation=on_confirmation)
```

**Rate limit.** Count deliveries per account in your store and `raise Reject(429, ...)`
when an account is over its limit; the sender is asked to try again later.

**Parse without the webhook.** `inbound.parse_email(raw_bytes)` works on any `.eml`;
`inbound parse message.eml` prints the result as JSON.

### Thread state: folders, tags, drafts, unread, needs a reply

```python
from inbound import Shelf

shelf = Shelf("state-dir")                      # JSON files in this directory
rows = shelf.rows(mail_log)                     # one row per thread, newest first
waiting = [r for r in rows if r["needs_reply"]]

shelf.folders.set(row["key"], "newsletter", actor="dana")      # one folder per thread
shelf.tags.set(row["key"], ["question", "event"], why="asks about Friday")   # many tags
shelf.drafts.save(row["key"], text="Friday at ten works.")     # at most one unsent reply
shelf.read.mark_read(row["key"], actor="dana")                 # unread again: mark_unread
shelf.read.mark_done(row["key"], actor="dana")                 # "dealt with"; new mail re-opens it
print(shelf.rollup(shelf.rows(mail_log)))                      # the "needs a human" board
```

`mail_log` is a list of dicts you already have: `direction` (`inbound` or `outbound`, default
inbound), `received_at` (or `ts`), `from_addr`, `from_name`, `to_addr`, `to_name`, `subject`,
`body`, and optionally `conversation_id` / `thread_id`. `examples/maillog.json` is a sample.

**Every message of a thread, the replies you sent included, must reach the same key tier.** Give
all of them the same `conversation_id`, or the same `thread_id`. A reply you stored with only
`in_reply_to` (or `references`) is placed on the thread whose root it names, but a reply stored
with nothing but a subject is keyed `subj:...` while the incoming mail it answers is keyed
`msg:...`: that shows as two threads, and the incoming one keeps "needing a reply".

The design is argued from mistakes, and each rule is tested:

- **Derived facts are not stored places.** A folder costs the operator a decision on every message
  forever; a fact costs nothing. So *needs a reply* is computed from the log every time: the
  thread's face is the newest incoming message, it is answered when somebody on your side wrote
  after it, and it needs a reply when it is unanswered, not filed away, the correspondent has not
  opted out, and no done mark covers it. Messages are ordered by the instant, not the text, so a
  reply written with another UTC offset is not placed before the message it answers.
- **Folders** are the inbox plus the places mail is filed away into (newsletters, removal
  requests, do not contact, junk by default; `FolderSet` takes your own). Retired names fold to
  the inbox on read and never raise. `folders.set(..., only_if_auto=True)` is for a classifier: it
  never overwrites a person's filing. An unreadable store reads as the inbox (mail is put back in
  front of a person, never hidden) and refuses writes.
- **Tags** are many per thread, in declared order, and a tag set is an argument (`TagSet`). Unknown
  tags are dropped, never stored. "No tags" is a stored answer, so `classified_keys()` tells
  "looked, nothing" from "never asked".
- **Drafts live apart from the mail log**, one per thread, keyed by the thread (never a message
  id): storing a draft on an append-only log means rewriting the log per message. A failed
  re-draft records its error and keeps the good reply that was waiting.
- **Read state fails toward unread.** A corrupt or evicted mark reads as unread. `is_read(key,
  newest_at=...)` and `rows(..., reopen_read_on_new_mail=True)` re-open a thread when mail
  arrives after it was read; by default a read mark stays read.
- **One thread key** for every store (`inbound.thread_key`): `cid:<conversation id>`, else
  `msg:<root message id>` (`thread_key_for(parsed_message)`; a row with no `thread_id` uses the
  first of its `references`, else its `in_reply_to`), else `subj:<subject>` with one reply prefix
  removed. Keys are cut to 256 characters on write and on read.
- **Writes never lose an update.** Every change is a locked read-modify-write with an atomic
  replace, across threads and processes. Pass `Shelf(lambda filename: your_store)` to keep the
  documents anywhere that has `read(default, strict=)` and `mutate(default, strict=)`, where a
  `strict=True` read of a document that exists but cannot be parsed raises `inbound.store.StoreUnreadable`
  or an exception class named `StateStoreUnreadable` (any other class makes the folder, tag and
  draft writers raise instead of refusing). The `JsonStore` from the statepath tool fits.
- **Erasure:** `shelf.forget_threads(keys)` removes the threads' folder, tags, draft and read/done
  marks. It does NOT touch the follow-up log by default, on purpose: that log only says "this
  conversation was already emailed", and forgetting it would let a second follow-up go out to
  someone you were erasing. Pass `followups=True` to erase it too (the conversation can then be
  emailed again). An unreadable store raises and is left as it was, rather than reporting the
  person gone or rewriting the file blank.
- **The board counts each thread once.** `shelf.rollup` has two built-in sources that do not
  overlap: threads waiting with no draft yet, and drafts waiting to be checked. `total` is their
  sum plus any sources of your own.

Beside the stores, pure helpers you can use on their own:

| Function | Does |
|---|---|
| `needs_reply(thread_facts(messages), filed_away=, opted_out=, done_mark=)` | the derivation above |
| `rollup([(key, label, count_fn)])` | counts-only "needs a human" board; a source that raises counts 0 |
| `decide_followup(...)` | should this conversation be emailed to the person who had it, and what exactly? Refuses with a reason: no address, one-sided chat, already sent, internal `[bracketed]` text stripped |
| `FollowupLog` | remembers which conversations were already emailed (record a send only after the provider accepted it) |
| `render_chat`, `render_call`, `render_email_thread` | a conversation as plain text, to quote into an email to staff |
| `reference_plain`, `reference_html`, `rich_text_html` | the quoted block, and plain text to email-safe HTML (bold labels, bullets, no CSS an email client drops) |
| `clean_addr_list`, `offered_recipients`, `default_body_for` | a clean recipient list; one-click copy suggestions that never add a recipient; a prefilled body only for purposes you list |

**Command line** (state under `--state`, default `$INBOUND_STATE` or `./inbound-state`):

    inbound thread rows examples/maillog.json         # one line per thread (add --json)
    inbound thread rollup examples/maillog.json       # the board
    inbound thread file cid:t-invoice junk --actor dana
    inbound thread tag cid:t-invoice question --why "asks how"   # an unknown tag is named on stderr, exit 2
    inbound thread draft cid:t-invoice --text "Friday at ten works."   # no text: show it; --clear
    inbound thread read cid:t-invoice                 # also: unread, done, undone
    inbound thread show cid:t-invoice                 # everything held, as JSON
    inbound thread key message.eml                    # the thread key of a raw message

### Configuration reference

**Environment (your app)**

| Name | Default | Meaning |
|---|---|---|
| `INBOUND_SECRET` | unset: every delivery is refused with 503 | Shared secret. A comma list (`new,old`) while rotating; the Worker signs with the first. |
| `INBOUND_DOMAINS` | unset: any domain | Comma list of domains addresses may arrive on. The first is where `mint()` puts new addresses. |
| `INBOUND_MAX_BYTES` | `10485760` (10 MiB) | Largest raw message accepted (413 above it). Keep equal to the Worker's. |

`INBOUND_SECRET` may also be read from your secrets manager: pass `secret=` a function.
If that function raises, the delivery is answered 503 and the sender is asked to try
again later.

**Worker secrets and vars** (details in worker/README.md):
`INBOUND_URL`, `INBOUND_SECRET`, `INBOUND_FALLBACK_FORWARD` (secrets), `INBOUND_MAX_BYTES`
(var). Set `INBOUND_URL` to the final URL: the Worker does not follow redirects, and
bounces (with a log line naming the new location) when your app answers 3xx.

**Headers the Worker sends:** `X-Inbound-Envelope-From`, `X-Inbound-Envelope-To`,
`X-Inbound-Timestamp`, `X-Inbound-Signature` =
hex HMAC-SHA256(secret, `"<timestamp>.<envelope to>.<sha256 hex of the raw message>"`).

**`InboundHandler(...)` options:** `resolve`, `deliver` (required); `secret` (string or
callable; default `INBOUND_SECRET`); `domains`; `max_bytes`; `window` (seconds, default
300); `tag_routes`; `on_forwarding_confirmation`; `keep_attachment_data` (default False);
`trusted_auth_servers` (default `("mx.cloudflare.net",)`: whose authentication headers
count); `clock` (for tests). `parse_email(raw, keep_attachment_data=..., trusted_auth_servers=...)`
takes the same two.

**Status codes** your receiver returns, and what the sender sees:

| Status | When | Bounce text |
|---|---|---|
| 200 | delivered | none |
| 3xx | your app redirected (the Worker never follows) | could not accept, try again later |
| 401 | missing, forged, re-addressed or stale signature | could not accept, try again later |
| 404 | not an inbound address, wrong domain, or `resolve` returned None | Unknown recipient |
| 413 | over `INBOUND_MAX_BYTES` | Message too large |
| 402, 403 | you raised `Reject(402/403)` | not accepting mail right now |
| 429 | you raised `Reject(429)` | try again later |
| 500 | `deliver` raised (logged to the `inbound` logger) | could not accept, try again later |
| 503 | no secret configured, the `secret` function raised, or `resolve` raised | could not accept, try again later |

**Main types:** `Address(slug, token, domain, tag)`; `InboundEmail` with `subject`,
`from_name`, `from_addr`, `to`, `cc`, `reply_to`, `date`, `message_id`, `in_reply_to`,
`references`, `thread_id`, `text`, `html`, `attachments` (`filename`, `content_type`,
`size`, `content_id`, `inline`, `data`), `forwarded`, `forwarded_for`, `auth`
(`spf`, `dkim`, `dmarc`, `dkim_pass_domains`, `server`, `spf_failed`, `dkim_failed`,
`dmarc_failed`),
`forwarding_confirmation`, `envelope_from`, `envelope_to`, `address`, `secret_slot`,
`raw`, plus `as_text()` and `to_dict()`.

**Command line:** `inbound secret`, `inbound address <slug> [--domain D] [--tag T] [--json]`,
`inbound parse <file|->`, `inbound send <file|-> --url URL --to ADDRESS`.

### Honest limits

**Thread state**

- **It does not keep your messages.** You pass the mail log in; `Shelf` keeps only the five small
  documents. Persisting inbound messages stays your database's job.
- **Each store is one JSON document**, rewritten whole on every change. That suits a help desk or
  a shared mailbox (thousands of threads), not millions; for more, pass your own store through the
  factory. Read and done marks are capped at 20,000 each, oldest evicted first (an evicted mark
  reads as unread). A damaged `read_state.json` or follow-up log is replaced by the next ordinary
  write rather than refused (unread is the cheap direction to be wrong in; erasure is the
  exception and refuses); folders, tags and drafts refuse instead.
- **Each state file has a `<name>.json.lock` file beside it.** The lock is taken on that file, so
  keep the pairs together when you copy or back up a state directory.
- **Cross-process locking uses `flock`** (macOS, Linux). On Windows only threads in one process
  exclude each other; run one writer process or bring your own store.
- **No classifier and no drafting model are included.** Tags and drafts are what you or your model
  put there; `only_if_auto` and `model` / `why` fields exist so you can tell the two apart.
- **"Needs a reply" is a rule, not a judgement.** It does not know a message was an automatic
  receipt or a thank-you. Tag it `ignore`, file it away, or mark it done.
- **`decide_followup` and `FollowupLog` do not send.** They decide, compose and remember; sending
  and the "only after the provider accepted it" call are yours.
- **The assistant's half of a stored chat** is read from `reply` (or `assistant`, `rc`) beside
  `user`, or from flat `{role, content}` turns. Another shape contributes nothing; seed your tests
  through the writer your product really uses.
- **`rich_text_html` is a small renderer** for labelled lines, bullets and paragraphs, not a
  Markdown engine, and it does not build a full email shell.

**Receiving**

- **The Worker is Cloudflare-only.** The Python side verifies any sender that signs the
  same way, but the included Worker needs your domain's DNS on Cloudflare and Email
  Routing enabled. Email Routing replaces the domain's MX records: if your domain
  already receives mail elsewhere, use a subdomain for inbound addresses.
- **Human mailboxes must not look like inbound addresses.** On a shared domain, a
  mailbox named `<something>-<12 of a-z2-7>` would be handed to your app instead of your
  inbox. `inbound.is_inbound_shaped("name")` checks a name before you create it.
- **A refusal is a bounce, not a retry.** When your app answers non-2xx, the Worker
  rejects the message and the sender's server reports it to the sender; Cloudflare does
  not queue it for later. Keep the receiver up, and use 5xx only for real failures.
- **No storage, dedupe or rate limiting built in.** Those belong in your database: use
  `message.message_id` as an idempotency key, and `Reject(429)` for limits.
- **Size:** 10 MiB by default (Cloudflare's own ceiling is 25 MiB). Attachments are
  listed by name, type and size; their bytes are kept only with `keep_attachment_data=True`.
- **SPF / DKIM / DMARC flags are read from headers** written by the receiving server.
  Only the top-most `Authentication-Results` or `ARC-Authentication-Results` whose
  server name is in `trusted_auth_servers` counts (default `mx.cloudflare.net`, the name
  Cloudflare Email Routing writes); headers naming any other server are ignored. The
  flags are only as good as that header: if the receiver adds none, they stay empty.
  Use a failure to lower trust; never treat a `pass` as proof on its own.
- **An attached message is one part.** Files inside a forwarded-as-attachment message
  belong to that message and are not listed in `attachments`.
- **Forwarding confirmation detection covers Gmail**, the provider that sends one.
  Outlook and Microsoft 365 forwarding needs no confirmation email.
- **HTML to text is a regex flattener**, good for message bodies, not a renderer.
- **Tokens are fixed at 12 characters** (60 random bits), the length the Worker matches.

### Tests

    python -m pytest

The Worker tests run the real `worker/worker.js` under Node 18+ (including one that
posts to a real Python receiver over HTTP). Without Node, deselect them:
`pytest -m "not worker"`.

---

Docs mirror of the private repository Dominares-Tools/inbound; updated with each release.

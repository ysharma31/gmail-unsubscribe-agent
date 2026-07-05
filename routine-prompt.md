# Monthly routine prompt

This is the exact instruction the scheduled (monthly) trigger runs. It uses the
Gmail connector's read + label tools. It is **report-and-label only** — it never
unsubscribes or sends mail. Cadence: monthly.

---

## Task

Scan Gmail for subscription/notification senders the user has stopped engaging
with, and flag the most recent thread from each with the `Unsubscribe Candidate`
label (labelId resolved via `list_labels`; create it if missing — color
background `#fb4c2f`, text `#ffffff`). Then report the list. Do not unsubscribe;
do not send email.

## Definitions

- **Candidate pool**: threads matching
  `(category:promotions OR category:updates) -label:"Unsubscribe Candidate" newer_than:365d`.
- **Interaction** (evidence the user engaged with a sender), any of:
  - a thread contains a reply from the user (`in:sent to:<sender>`), or
  - a message from the sender is `is:read` or `is:starred`.
  Do **not** count `is:important` — Gmail applies it algorithmically, not by user
  action, so it is not reliable evidence of engagement.
- **Stale**: the sender has **no** interaction in the last **90 days**, AND the
  sender has sent **≥ 2** messages in the 365-day window.

## Exclusions (never flag these)

`category:updates` sweeps in transactional / security / financial / government
mail, which must never be flagged (you cannot and should not unsubscribe from a
fraud alert or a receipt). Exclude senders that are clearly:

- **Bank / card / brokerage alerts**: e.g. `*.chase.com`, `*adcb*`,
  `*capitalone*`, `*.fidelity.com`, `interactivebrokers.com`, `*sofi*`,
  `fundrise.com`, `stripe.com`, `uphold.com`.
- **Account security / login / receipts**: e.g. `noreply-accounts@google.com`,
  `*@mail.anthropic.com` (login links, receipts), any "verify your email" /
  "secure link to log in" / "your receipt" sender.
- **Government / official services**: e.g. `*.tamm.abudhabi`, `*.abudhabi`.
- Any sender the user has an existing Gmail **label** for (they clearly care):
  check `list_labels`; e.g. `events@thedigitaleconomist.com` is covered by the
  "The Digital Economist" label.

When unsure whether a sender is marketing vs transactional, **do not flag it** —
under-flagging is the safe error; the report is reviewed by the user.

## Algorithm

1. Page through the candidate pool (metadata view) and group messages by sender
   address. Record, per sender: message count, most-recent message date, and the
   threadId of that most-recent message.
2. Drop senders with < 2 messages, and drop excluded senders (above).
3. For the remaining senders, batch-verify interaction with OR-queries
   (~12 senders per query) to keep tool calls low:
   - `from:(a OR b OR …) (is:read OR is:starred) newer_than:90d`
   - `in:sent to:(a OR b OR …) newer_than:120d`
   Any sender that appears in either result has interacted recently → **not stale**.
4. The remaining senders are **stale**. Apply `Unsubscribe Candidate` to each
   one's most-recent threadId.
   - **Apply labels sequentially (one `label_thread` call at a time), not in a
     large parallel batch** — parallel write bursts have caused the connector's
     permission stream to drop. One-at-a-time is reliable.
5. Report a table: sender, type, message count, most-recent date. Note any
   senders excluded as transactional so the user sees why.

## Notes / known caveats

- **Read-signal caveat**: `is:read` is a proxy for engagement. If the user ever
  bulk-"mark as read", a genuinely-stale sender would look engaged and be
  skipped. Worth mentioning in the report if the pattern looks off.
- Senders already carrying the label are excluded by the pool query, so the user
  won't be re-notified about the same sender each month. If they decide to keep a
  labeled sender, they remove the label; if they unsubscribe, the label stays as
  a record.

# Gmail Unsubscribe Agent

A Claude Code routine (not a standalone app) that scans your Gmail inbox on a
schedule, finds recurring subscription/notification senders you've stopped
engaging with, and flags them for you to review. **It never unsubscribes or
sends anything on your behalf** — it only labels candidate threads and sends
you a summary. You decide what to actually unsubscribe from.

## Why a routine, not a script

The scan relies on the Gmail connector already authorized for this Claude
session (search, read, label). There's no separate OAuth app, no credentials
to manage, and no server to host — a scheduled trigger just re-invokes Claude
with the instructions below and lets it use the same Gmail tools.

## Definitions

- **Subscription/notification email**: a message in Gmail's `category:promotions`
  or `category:updates` categories. These are the categories Gmail itself
  applies to bulk/automated senders (newsletters, product updates, digests,
  receipts-with-marketing, etc.).
- **Interaction**: any signal that you engaged with a sender's email —
  - you replied to one of their threads (a message in the thread from your own address), or
  - the message is marked read (`is:read`), or
  - the message is starred or marked important.
  Gmail's API doesn't expose open/click tracking, so read-status is the closest
  available proxy for "you looked at it."
- **Stale**: the sender's most recent interaction (as defined above) is more
  than the staleness threshold ago, or there has never been any interaction,
  **and** the sender has sent at least 2 emails historically (so a single
  one-off email doesn't get flagged).
- **Staleness threshold**: 90 days (default; edit below if you want to change it).

## Gmail label used

`Unsubscribe Candidate` — applied to the most recent thread from a sender
that meets the "stale" definition above. Once labeled, a sender is excluded
from future scans (`-label:Unsubscribe Candidate` in the search query) so you
don't get repeat notifications about the same sender. If you decide to keep a
sender after reviewing, just remove the label from its thread; if you decide
to unsubscribe, do so manually (via the email's unsubscribe link) and the
label stays as a record.

## Algorithm (what each run does)

1. Search: `(category:promotions OR category:updates) -label:"Unsubscribe Candidate" newer_than:365d`
2. Group the resulting threads by sender email address.
3. Drop senders with fewer than 2 total messages in the window.
4. For each remaining sender, compute:
   - `last_email_date`: date of their most recent message.
   - `last_interaction_date`: most recent date among messages that are read,
     starred, important, or replied-to; `null` if none qualify.
5. Flag the sender as a candidate if `last_interaction_date` is `null` or more
   than 90 days before today.
6. Apply the `Unsubscribe Candidate` label to that sender's most recent thread.
7. Send a summary report: sender, message count, last email date, last
   interaction date (or "never"), and thread link/id for quick access.

## Schedule

Runs weekly. Each run starts a fresh session (no memory of prior runs beyond
what Gmail labels encode), scans, labels, and reports — then ends.

## Changing the threshold or cadence

- Threshold: edit "90 days" in this file and in the routine's trigger prompt
  (`Unsubscribe Candidate` scan logic) to match.
- Cadence: update the routine's cron schedule.

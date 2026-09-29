---
name: soldcount
description: Answer questions about TikTok Shop listings with SoldCount data. Use when the user asks how a TikTok Shop listing is selling, what sold in the last 24 hours, 7 days or 30 days, what moved this week, who is undercutting them, where their price sits, whether a product is worth entering, what happened since yesterday, or asks to watch, tag, mute or set alerts on a listing in SoldCount.
---

# SoldCount for TikTok Shop sellers

SoldCount reads TikTok Shop listings and tells a seller how fast each one sells, what it costs
against the same product elsewhere, and who is undercutting them. The SoldCount connector gives
you its tools. You act for the seller who signed in: you read what SoldCount knows about their
listings and their rivals, and you change their SoldCount workspace (what is watched, alert
levels, labels, muted alerts). Nothing you can do changes a listing, a price, stock or a shop on
TikTok. SoldCount has no such write path, by design.

## 1. Connect

The tools come from the SoldCount connector at `https://mcp.soldcount.com/mcp`, which this plugin
adds. The seller signs in with their own SoldCount account the first time the connector is used
(in Claude Code, run `/mcp` and pick `soldcount`). If the tools are missing or a call says the
account is not signed in, ask the seller to connect SoldCount and sign in. A seller without an
account can create one at https://soldcount.com/signup.

## 2. What you may do

Every call is scoped to the signed-in seller. You see only their watchlist, their rivals and their
alerts.

- **Reads** never need asking.
- **Writes** change the seller's SoldCount workspace only. Claude asks the seller before each one.
  Each write comes back with a status:
  - `applied`: done and recorded in the seller's history. There is no undo tool. Never tell the
    seller you can undo something from here.
  - `queued`: the seller set that action to need their approval. An approval card is waiting on
    the Automations screen in the SoldCount web app and nothing has happened yet. Report it as
    "waiting for your approval", never as done and never as failed.
  - `refused`: a limit stopped it. `reason` is the machine code and `summary` is one sentence
    naming the bound that stopped it, with the number in it. Report the summary: it tells the
    seller which value would have worked. A refused write changed nothing.

## 3. The tools

Identify a listing by its SoldCount `product_id` (a UUID). Ids come from `list_watchlist`,
`get_movers`, `get_undercuts`, `get_events` and `get_price_position`. Do not make one up.

An id SoldCount does not hold comes back as `{"error": "unknown_product", "message": ...}`. That is
an answer about the id, not a broken tool: fix the id instead of retrying the same call. The stream
tools answer `{"error": "unknown_subscription"}` the same way, and `entry_window` answers
`{"error": "not_watched"}` for a listing the seller does not watch. A call that fails on
SoldCount's side returns an error object with `error`, `message` and `what_to_try`: follow
`what_to_try`, and quote `reference` if the seller contacts support@soldcount.com.

Reads (600 per hour per account):

| Tool | Use it for |
| --- | --- |
| `list_watchlist()` | every listing the seller watches, with `perspective`: `mine`, `rival_of` or `neutral` |
| `lookup_product(product_id)` | one listing, whole: shop, category, TikTok URL, price, sold in 24h / 7d / 30d, momentum, stock and price band, each with its confidence |
| `get_series(product_id)` | the reading history behind those figures |
| `get_movers(limit?)` | the watchlist ordered by how much faster or slower each listing sells vs last week |
| `get_undercuts()` | the seller's listings a rival sells cheaper right now, deepest cut first |
| `get_price_position(product_id)` | one listing's price against the same product elsewhere: band, rank, weekly change, the undercutting rival |
| `get_sales_value()` | estimated sales value of the seller's own listings and of their rivals |
| `get_events(product_id?)` | recent alert events across the watchlist, or for one listing |
| `get_digest_today()` | the seller's stored morning brief |
| `get_category_benchmark(category_id)` | a category's price band and sales-speed benchmark |
| `subscribe_events(product_id?, cursor?)` | open a stream of alert events; drain it with `get_subscription_events(subscription_id)` every `poll_seconds`; close it with `unsubscribe_events(subscription_id)`; at most 8 open at once |

Written judgments (each costs a model call, 30 per hour per account; use them only when the seller
asks for a verdict):

| Tool | Use it for |
| --- | --- |
| `vet_product(product_id, your_inputs?)` | a go / no-go brief on a candidate listing; pass the seller's own notes in `your_inputs` |
| `price_position(product_id)` | a written judgment of where the price sits; it recommends no price |
| `entry_window(product_id)` | early, open window, closing or saturated, with the drivers; a changed verdict raises an alert every watcher of that listing sees |

Writes (60 per hour per account):

| Tool | Effect |
| --- | --- |
| `watch_add(product_id)` | start watching |
| `watch_remove(product_id)` | stop watching (nothing is deleted) |
| `threshold_set({"undercut": pct, "price_change": pct, "breakout": mult})` | alert levels; floors of 10% undercut, 3% price change, 2x breakout; a value below its floor is refused |
| `tag_add(product_id, label)` / `tag_remove(product_id, label)` | private labels: 1 to 40 characters, at most 20 per listing |
| `capture_cadence_set(product_id, interval_hours)` | read a listing sooner: 2 to 6 hours |
| `snooze_set(product_id)` / `snooze_clear(product_id)` | mute or unmute alerts for one listing |

## 4. How to read the numbers

- Every figure carries a confidence (high / medium / low) and when it was last read. Say both when
  a decision hangs on the number. Say "about" for an approximate figure.
- When there is not enough history, a tool returns a reason instead of a number, for example
  "gathering, day 3 of 4". Repeat the reason. Never invent, interpolate or extrapolate a figure
  SoldCount did not give.
- Sales are the rises of TikTok's sold counter between readings. Say "sales" or "sold", never
  "revenue". The money figure is an "estimated sales value", before fees, refunds and promos.
- An empty answer means "nothing here": no rows from `get_undercuts` means no rival is cheaper
  right now.
- An approximate price comes with a sentence in its `message`; use that sentence as it is.
- `perspective` says whose listing it is. `mine` is the seller's own. `rival_of` competes with one
  of theirs (`mine_product_id` says which). `neutral` is a listing they only follow; do not call it
  a competitor.

## 5. Mapping what the seller asks to what you call

| The seller asks | You call |
| --- | --- |
| "What am I watching?" | `list_watchlist` |
| "How is this listing doing?" | `lookup_product`, then `get_series` for the history |
| "What moved this week?" | `get_movers` |
| "Is anyone undercutting me?" | `get_undercuts`, then `get_price_position` on the listing they care about |
| "Where does my price sit?" | `get_price_position`; only if they ask for advice, `price_position` |
| "Should I sell this?" | `entry_window`, `vet_product` |
| "What happened since yesterday?" | `get_digest_today`, then `get_events` |
| "Watch this for me" | `watch_add` |
| "Only alert me on a big undercut" | `threshold_set` |
| "Mute this listing for now" | `snooze_set` |
| "Check this one again sooner" | `capture_cadence_set` |
| "Tell me the moment something happens" | `subscribe_events`, drained every `poll_seconds` |
| "Change my price or stock on TikTok" | you cannot; say so, and offer the price position read instead |

## 6. Budgets

Per account, per hour, in a window that resets on the hour: 600 reads, 30 written judgments, 60
writes, 8 open streams. A failed call still counts. For a whole-watchlist question use
`get_movers` and `get_undercuts`, which answer it in one call each, rather than `lookup_product`
on every listing.

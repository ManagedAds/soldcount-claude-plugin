# SoldCount for Claude

![SoldCount](assets/soldcount-mark.svg)

SoldCount tells TikTok Shop sellers how fast listings sell. This plugin lets you ask Claude about
the listings you watch in SoldCount and get the same honest figures you see in the SoldCount web
app and browser extension.

## What it does

The plugin adds two things to Claude:

- **The SoldCount connector**, a remote MCP server at `https://mcp.soldcount.com/mcp`. It gives
  Claude tools to read your SoldCount data (the listings you watch, how many sold in the last 24
  hours, 7 days and 30 days, prices against the same product sold elsewhere, rivals undercutting
  you, alerts and your morning brief) and to change your own SoldCount workspace (your watchlist,
  tags, alert levels and silenced alerts).
- **The `soldcount` skill**, which explains to Claude what each tool returns, how to read a
  confidence level, and what to say when SoldCount does not have enough data yet.

Ask things like "what moved on my watchlist this week?", "is anyone undercutting my pillow
listing?" or "silence alerts for this listing".

## What it never does

It never writes to TikTok. No tool can change a listing, a price, stock or anything else in a
TikTok shop. Changes are limited to your SoldCount workspace, Claude asks you before each one, and
every change is recorded in your SoldCount activity.

## Sign in

You need a SoldCount account (https://soldcount.com/signup). The first time Claude uses the
connector, it opens a SoldCount page where you sign in and allow access. To disconnect, remove the
connector in Claude's settings, or revoke the connected app in SoldCount under Automations, then
Connect.

## Data and privacy

The plugin contains only instructions and the connector address. It runs no code on your computer
and stores nothing itself. When Claude calls a SoldCount tool, it sends SoldCount the tool name and
its arguments (for example a SoldCount listing id or a tag label) over HTTPS to
`mcp.soldcount.com`, which is operated by Managed Ads Inc., the company behind SoldCount.
SoldCount never receives your conversation with Claude. SoldCount keeps a record of the changes
made through the connector and of the sign-in grant until you revoke it. The full policy is at
https://soldcount.com/privacy.

## Limits

Per account and per hour: 600 reads, 30 written verdicts and 60 changes. Up to 8 alert streams can
be open at once, and an account watches up to 100 listings by default.

## Your own agent

The same tools work outside Claude too. An agent on your own computer connects with an access key
or by signing in, and reads its instructions from https://soldcount.com/soldcount-agent.md. Use one
SoldCount connection per client, or every tool shows twice.

## Support

Documentation: https://soldcount.com/claude. Questions or problems: support@soldcount.com.

## License

MIT, see `LICENSE`.

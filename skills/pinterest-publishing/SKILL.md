---
name: pinterest-publishing
description: Publish, schedule, bulk-post, fix and measure Pinterest pins through the PinBridge MCP server. Use when the user wants to post a pin, schedule pins for later, spread a batch of pins over days, pick or create a board, check how pins performed, or fix a pin that failed or went out wrong.
---

# Pinterest publishing with PinBridge

The `pinbridge` MCP server does the Pinterest work: account tokens, queueing, retries and status tracking. This skill is the order to call it in and the checks that keep a pin from going out wrong. Anything published lands on the user's public Pinterest profile, so nothing is published or scheduled before the user has seen the final payload and said yes.

## Before anything else

- No `pinbridge` tools available: tell the user to connect PinBridge (the plugin adds the server; Claude asks them to sign in with their PinBridge account the first time). Setup guide: https://www.pinbridge.io/docs/mcp/setup/
- `list_pinterest_accounts` returns nothing: the user has to connect a Pinterest account in the dashboard at https://app.pinbridge.io. Connecting and disconnecting accounts is dashboard-only on purpose; don't look for a tool that does it.

## Publish one pin now

1. **Account.** `list_pinterest_accounts`. One account: use it. Several: ask which.
2. **Board.** `list_boards(account_id)`. Suggest the board that fits the pin and confirm it. If the board is new to this session or a publish to it failed before, run `check_board_access` and follow its remediation. Only `create_board` when the user asks for a new board.
3. **Image.** A public URL goes in as `image_url`. An image the user attached, one you generated, or a URL behind a login goes through `upload_asset` first, then `asset_id`. Pass one or the other, never both.
4. **Copy.** Draft it, following "Writing the pin" below.
5. **Dry run.** `create_pin(..., dry_run=true)`. Show the user the resolved title, description, link, board and every check. Stop if a check failed and fix the input.
6. **Publish.** After the user confirms, call `create_pin` with the same arguments, `dry_run=false`, and `idempotency_key` set to `resolved.idempotency_key` from the dry run. Reuse that same key on any retry so the pin is never posted twice.
7. **Follow up.** Poll `get_pin` until the status is `published`, `failed` or `deferred`.
   - `published`: give the user the pin link.
   - `deferred`: PinBridge is pacing the publish because of quota or rate limits. Report the `error_code` and wait. Don't resubmit.
   - `failed`: read `error_code` and `error_message`. `retry_pin` can retry, or move the pin to another board or account.

## Schedule for later

- Use `create_schedule` with `run_at` as ISO 8601 with a timezone offset. A time without an offset is rejected. If the user says "9am" and hasn't said where, ask for their timezone once.
- Dry-run it the same way, and get a yes before the real call.
- A repeat `create_schedule` makes a second schedule. After a timeout, check `list_schedules` before sending again.
- Wrong time, board or copy on a pending schedule: `update_schedule`. Don't cancel and recreate for a small fix.

## Several pins

- Up to 100 pins at once: `create_pins_batch`. It needs the workspace's bulk feature and enough quota for every entry, so check `get_billing_status` first. Without the feature it fails with `payment_required`; publish one by one with `create_pin` instead, or schedule them.
- Pinning a batch to the same board in one burst reads as spam to Pinterest. Offer to spread them with schedules, for example a few per day, and use `get_rate_meter(account_id)` to see the room left.
- Give each entry its own `idempotency_key` so a resend doesn't duplicate.
- Show the user the full list (board, time, title for each) before publishing a batch.

## Writing the pin

- **Title:** 100 characters max. Lead with what the pin is about, in words people search for.
- **Description:** 800 characters max. Two or three plain sentences, with the main keywords worked in naturally. `list_related_terms` suggests terms people search for; use the ones that fit. No hashtag walls.
- **Alt text:** describe the image for someone who can't see it, 500 characters max. Always include it.
- **Link:** the page the pin should send people to. Keep any UTM tags the user gives.
- Write in the user's language and voice. If they gave you copy, use it as written and only point out limits it breaks.

Pinterest re-reads the destination page and may show that page's Open Graph/meta title instead of the title sent through the API. If a published pin shows a different title, that's why. The fix is the page's `og:title`, not re-publishing the pin.

## Measuring

- One pin: `get_pin_analytics(pin_id)` for impressions, saves, outbound clicks.
- A whole account: `get_account_analytics`.
- New pins take a day or two to show numbers. Say so rather than reading zeros as failure.
- When comparing pins, point out what the better ones have in common (board, image style, title wording, time) and suggest one concrete thing to try next.

## Fixing mistakes

- Wrong title, description, link, alt text or board on a published pin: `update_pin`.
- Pin shouldn't exist: `delete_pin`, after the user confirms. Never delete a board to get rid of pins.
- `delete_pin`, `delete_schedule`, `delete_board` and `delete_webhook` can't be undone. Name exactly what will be removed and wait for a clear yes.

## Errors

Every error has a stable `code` and a remediation sentence. Follow the remediation and tell the user what it means for them in one line. Don't retry blindly. See `references/errors.md` for what each code means and what to tell the user.

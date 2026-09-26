---
name: pinterest-publishing
description: Publish, schedule, bulk-post, fix and measure Pinterest pins through the PinBridge MCP server. Use when the user wants to post a pin, schedule pins for later, spread a batch of pins over days, pick or create a board, move or cancel a scheduled pin, report how publishing went over a period, check how pins performed, or deal with a pin that failed, went out wrong or was deleted on Pinterest.
---

# Pinterest publishing with PinBridge

The `pinbridge` MCP server does the Pinterest work: account tokens, queueing, retries and status tracking. Each tool's description lists its parameters, return fields and failure codes; read it there rather than guessing. This skill covers the order to call things in, what to confirm with the user, how to write the pin, and the rules the tools can't enforce on their own. Anything published lands on the user's public Pinterest profile, so nothing is published or scheduled before the user has seen the final payload and said yes.

## Before anything else

- No PinBridge tools available: tell the user to add the PinBridge MCP server (`https://mcp.pinbridge.io`) to their assistant. The first call opens a PinBridge sign-in page; clients without sign-in support can send a PinBridge API key as `Authorization: Bearer pb_...` instead. Setup guide: https://www.pinbridge.io/docs/mcp/setup/
- `list_pinterest_accounts` returns nothing: the user has to connect a Pinterest account in the dashboard at https://app.pinbridge.io. Connecting and disconnecting accounts is dashboard-only on purpose; don't look for a tool that does it.
- Plan features: `get_billing_status` returns the monthly quota, what's used, and two feature flags. `uploaded_media_assets` covers `upload_asset`, `asset_id` and video pins. `bulk_imports` covers `create_pins_batch`. Both are off on the free Playground plan. A call that needs a missing feature fails with `feature_unavailable`. Check once before a task that depends on either feature.

## Publish one pin now

1. **Account.** `list_pinterest_accounts`. One account: use it. Several: ask which.
2. **Board.** `list_boards(account_id)`. Suggest the board that fits the pin and confirm it. If the board is new to this session or a publish to it failed before, run `check_board_access` and follow its remediation. Only `create_board` when the user asks for a new board.
3. **Image.** A public URL goes in as `image_url`. An image the user attached, one you generated, or a URL behind a login goes through `upload_asset` first (the exact file bytes as `content_base64`, or `source_url`), then `asset_id`. Pass one or the other, never both. Uploads need `uploaded_media_assets`; without it, ask the user for a public link to the image, or mention that paid plans accept uploads. Video pins always need an uploaded asset and a cover image.
4. **Copy.** Draft it, following "Writing the pin" below.
5. **Dry run.** `create_pin(..., dry_run=true)`. Show the user the resolved title, description, link, board and every check. A failed check means stop and fix the input. A `rate_paced` warning only means the publish will wait for Pinterest's rate limit; tell the user it may go out a little later.
6. **Publish.** After the user confirms, call `create_pin` with the same arguments, `dry_run=false`, and `idempotency_key` set to `resolved.idempotency_key` from the dry run. Reuse that same key on any retry so the pin is never posted twice.
7. **Follow up.** Poll `get_pin` until the status is `published`, `failed` or `deferred`.
   - `published`: give the user the pin link.
   - `deferred`: PinBridge is pacing the publish because of quota or rate limits. Report the `error_code` and wait. Don't resubmit.
   - `failed`: read `error_code` and `error_message` (see `references/errors.md`). Bad copy: fix it with `update_pin`, then `retry_pin`. Bad board or account: `retry_pin` with a new `board_id` or `account_id`. A failed pin stays failed until `retry_pin` re-queues it.

## Schedule for later

- Use `create_schedule` with `run_at` as ISO 8601 with a timezone offset. A time without an offset is rejected. If the user says "9am" and hasn't said where, ask for their timezone once.
- Dry-run it the same way, and get a yes before the real call.
- Schedules carry no alt text: `create_schedule` and `update_schedule` have no `alt_text` field. If alt text matters to the user, say so, and offer to publish at the chosen time with `create_pin` instead.
- `create_schedule` has no idempotency key: a repeat call makes a second schedule. After a timeout, check `list_schedules` (search the title with `q`) before sending again.
- Wrong time, board, copy or image on a pending schedule: `update_schedule`. Don't cancel and recreate for a small fix. Once a schedule has started publishing it fails with `schedule_not_editable`; treat the result like any published pin.
- The pin shouldn't go out at all: `cancel_schedule`, after the user confirms. `delete_schedule` only removes finished records (done, failed or canceled). It can't stop a pending schedule.
- A failed schedule: `retry_schedule`. To move it to another board or account first, `retry_pin` on its `pin_id`.

## Several pins

- Up to 100 pins at once: `create_pins_batch`. It needs `bulk_imports` and enough quota for every entry, so check `get_billing_status` first. Without the feature it fails with `feature_unavailable`; publish one by one with `create_pin` instead, or schedule them.
- Pinning a batch to the same board in one burst reads as spam to Pinterest. Offer to spread them with schedules, for example a few per day, and use `get_rate_meter(account_id)` to see the room left.
- Give each entry its own `idempotency_key`. Without one, the server generates a key and a resend publishes the entry again.
- Show the user the full list (board, time, title for each) before publishing a batch.
- The batch reports each entry separately. Tell the user which entries failed and why. Resend only those entries, with the same keys.

## Writing the pin

- **Title:** 100 characters max. Lead with what the pin is about, in words people search for.
- **Description:** 800 characters max. Two or three plain sentences, with the main keywords worked in naturally. `list_related_terms` suggests terms people search for; use the ones that fit. No hashtag walls.
- **Alt text:** describe the image for someone who can't see it, 500 characters max. Include it on every `create_pin` and batch entry.
- **Link:** the page the pin should send people to. Keep any UTM tags the user gives.
- Write in the user's language and voice. If they gave you copy, use it as written and only point out limits it breaks.

Pinterest re-reads the destination page and may show that page's Open Graph/meta title instead of the title sent through the API. If a published pin shows a different title, that's why. The fix is the page's `og:title`, not re-publishing the pin.

## Reporting and measuring

- "How did publishing go this week / last month?": `get_dashboard_summary(start, end, tz)` with the user's timezone. It gives counts per status, success rate, the previous period for comparison, and what's queued or scheduled. Outcomes count when they happened: a pin submitted last month and published today counts as published today.
- "How many ...?": call `list_pins` or `list_schedules` with the filters and `limit=1`, and read `total`. Never page through rows just to count.
- Finding pins: `list_pins` with `q` (searches title, description and link), `status`, `error_code`, `board_id` or `since`/`until`. Lists come back one page at a time. For the next page, repeat the call with `offset + limit` while `has_more` is true.
- Engagement for one pin: `get_pin_analytics(pin_id)` for impressions, saves and outbound clicks. For a whole account: `get_account_analytics`. Pass `include_daily=false` when totals are enough. Stored history reaches back 366 days; live reads from Pinterest reach 90 days. Comments and reactions only come back with `source="live"`.
- New pins take a day or two to show numbers. Say so rather than reading zeros as failure.
- When comparing pins, point out what the better ones have in common (board, image style, title wording, time) and suggest one concrete thing to try next.
- "What happened to this pin?" or changes made outside this chat: `list_activity_logs`.

## Pins deleted on Pinterest

A published pin that was later deleted on Pinterest keeps status `published` and gains `removed_from_pinterest_at`. `list_pins(removed=true)` finds them. Their analytics come from PinBridge's stored history, and editing them fails with `pin_removed_on_pinterest`. Offer to publish the pin again with `create_pin`, or to clear the record with `delete_pin`.

## Fixing mistakes

- **Not published yet** (queued, deferred or failed): `update_pin` fixes the title, description, link, alt text or board in place. A pin in the middle of publishing fails with `pin_publishing`; wait for `get_pin` to settle.
- **Already published:** it can't be edited. Pinterest's API doesn't allow changing a live pin, so `update_pin` fails with `pin_already_published`. Offer to delete it and publish a corrected pin, and say that the new pin starts again from zero saves and clicks. Do both only after the user confirms. A different image works the same way.
- **Pin shouldn't exist:** `delete_pin`, after the user confirms. By default it also removes the pin from Pinterest. `delete_from_pinterest=false` only drops PinBridge's record and leaves the pin live.
- Never delete a board to get rid of pins. `delete_board` deletes the board on Pinterest together with every pin on it.
- `delete_pin`, `delete_schedule`, `delete_board` and `delete_webhook` can't be undone. Name exactly what will be removed and wait for a clear yes.

## Errors

Every error has a stable `code` and a remediation sentence. Follow the remediation and tell the user what it means for them in one line. Don't retry blindly; a dry run or `check_board_access` tells you more than a second attempt. Two families are easy to mix up. `insufficient_scope` and `account_not_permitted` are about the PinBridge key. `token_expired`, `token_revoked` and `scope_missing` are about the Pinterest connection. `pinterest_feature_unavailable` and `pinterest_not_permitted` are Pinterest refusing the action itself, so reconnecting won't help. See `references/errors.md` for what each code means and what to tell the user.

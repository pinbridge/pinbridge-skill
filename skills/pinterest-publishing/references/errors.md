# PinBridge error codes

Every failure from the PinBridge MCP server carries a `code` and a `remediation` sentence. Follow the remediation first; this table is for explaining it to the user.

| Code | What it means | What to do |
|---|---|---|
| `token_expired`, `token_revoked`, `scope_missing` | The Pinterest account's connection is broken or missing a permission. | Ask the user to reconnect that Pinterest account at https://app.pinbridge.io. Don't retry until they have. |
| `insufficient_scope`, `account_not_permitted` | The PinBridge key Claude is using isn't allowed to do this, or can't use this Pinterest account. | This is about the PinBridge key, not Pinterest. The user needs a key with wider access from a workspace admin, or to reconnect PinBridge with the right workspace. |
| `board_not_found`, `board_deleted` | The board no longer exists. | Call `list_boards` again and pick another board, or create one if the user asks. |
| `board_not_owned`, `board_access_denied` | The account can't post to this board, often a group board it was removed from. | Run `check_board_access` and follow its remediation, or choose a board the account owns. |
| `quota_exceeded` | The workspace used its monthly publishing quota. | Check `get_billing_status` for the reset date. Offer to schedule the pins after the reset, or point the user to a plan upgrade. |
| `rate_limited` | Too many requests right now. | Wait `retry_after_seconds`, or spread the remaining pins out with `create_schedule`. |
| `payment_required` | The feature (for example bulk publishing) isn't in the workspace's plan. | Use the non-bulk path (`create_pin` per pin or schedules), or point the user to a plan that includes it. |
| `validation_error` | A field is wrong: too long, missing, a past or timezone-less `run_at`, both `image_url` and `asset_id`, and so on. | Read the message, fix the field, dry-run again. |
| `not_found` | The pin, schedule or webhook ID doesn't exist in this workspace. | List again (`list_pins`, `list_schedules`, `list_webhooks`) and use a current ID. |

A plan error on connect (the server requires a minimum plan) means the workspace needs an upgrade at https://www.pinbridge.io.

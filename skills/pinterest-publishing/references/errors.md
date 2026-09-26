# PinBridge error codes

Every failed call to the PinBridge MCP server carries a `code` and a `remediation` sentence. Follow the remediation first; these tables are for explaining it to the user. Codes not listed here come with the same remediation sentence, so rely on it.

## Errors a tool call returns

### Connections and keys

| Code | What it means | What to do |
|---|---|---|
| `token_expired`, `token_revoked`, `scope_missing` | The Pinterest account's connection is broken or missing a permission. | Ask the user to reconnect that Pinterest account at https://app.pinbridge.io. Don't retry until they have. `scope_missing` on a secret board means the account was connected before secret boards were supported; reconnecting fixes it. |
| `account_revoked` | `check_board_access` reports the Pinterest account was disconnected from PinBridge. | Same as above: reconnect it in the dashboard. |
| `insufficient_scope`, `account_not_permitted` | The PinBridge key in use isn't allowed to do this, or can't use this Pinterest account. | This is about the PinBridge key, not Pinterest. The user needs a key with wider access from a workspace admin, or to reconnect PinBridge with the right workspace. |
| `pinterest_feature_unavailable`, `pinterest_not_permitted` | Pinterest refuses this action through PinBridge. The connection itself is fine. | Reconnecting won't help. Change the request, or tell the user to make this change on Pinterest directly. |

### Boards

| Code | What it means | What to do |
|---|---|---|
| `board_not_found`, `board_deleted` | The board no longer exists. | Call `list_boards` again and pick another board, or create one if the user asks. |
| `board_not_owned`, `board_access_denied` | The account can't post to this board, often a group board it was removed from. | Run `check_board_access` and follow its remediation, or choose a board the account owns. |

### Pins and schedules

| Code | What it means | What to do |
|---|---|---|
| `pin_already_published` | Published pins can't be edited. Pinterest's API doesn't allow it. | Offer to delete the pin and publish a corrected one. Do it only after the user confirms. |
| `pin_removed_on_pinterest` | The pin was deleted on Pinterest. Editing it fails with this code, and so do analytics when PinBridge stored no history for it. | Offer to publish it again with `create_pin`, or clear the record with `delete_pin`. |
| `pin_publishing` | The pin is being published right now. | Wait, check `get_pin`, then act on its final status. |
| `pin_not_published` | Analytics were asked for a pin that hasn't published yet. | Check `get_pin`; numbers exist only after it publishes. |
| `schedule_not_editable` | The schedule has already started publishing. | It can't be changed now. Once its pin is published, treat it like any published pin. |
| `conflict`, `bad_request` | The pin or schedule isn't in a state that allows this, for example retrying a pin that didn't fail, or deleting a schedule that is still pending. | Re-read it with `get_pin` or `get_schedule`. A pending schedule is stopped with `cancel_schedule`, not `delete_schedule`. |
| `not_found` | The pin, schedule, board or webhook ID doesn't exist in this workspace. | List again (`list_pins`, `list_schedules`, `list_webhooks`) and use a current ID. |

### Input

| Code | What it means | What to do |
|---|---|---|
| `validation_error` | A field is wrong: too long, missing, a past or timezone-less `run_at`, both `image_url` and `asset_id`, and so on. | Read the message, fix the field, dry-run again. |
| `invalid_image` | The image file is broken or truncated. | Upload a valid file. Re-encode the exact original bytes; never retype base64 by hand. |
| `invalid_date_range`, `invalid_timezone` | A report's range is backwards or longer than 366 days, or the timezone isn't an IANA name. | Fix the range, or use a name like `Europe/Paris`. |

### Plan and billing

| Code | What it means | What to do |
|---|---|---|
| `feature_unavailable` | The workspace's plan doesn't include this feature, for example image uploads or bulk publishing. The free Playground plan has neither. | Uploads: ask for a public image URL instead. Bulk: publish with `create_pin` per pin, or schedule them. Or point the user to a paid plan. |
| `quota_exceeded` | The workspace used its monthly publishing quota. | Check `get_billing_status` for the reset date. Offer to schedule the pins after the reset, or point the user to a plan upgrade. |
| `credits_exhausted` | The monthly quota is used up and no extra credits are left. | The user can buy a credit pack in the dashboard or wait for the monthly reset. |
| `billing_inactive`, `payment_required` | Publishing is blocked by the plan or a payment problem. | The user needs to update their payment method or plan in the dashboard. Nothing to retry until then. |

### Throttling and outages

| Code | What it means | What to do |
|---|---|---|
| `rate_limited` | Too many requests right now. | Wait `retry_after_seconds`, or spread the remaining pins out with `create_schedule`. |
| `pinterest_unavailable` | Pinterest didn't answer. | Retry in about 30 seconds. If it keeps failing, tell the user Pinterest is having trouble. |
| `service_unavailable`, `internal_error` | PinBridge had a temporary problem. | Retry once shortly. If it repeats, tell the user and include the `request_id` for support. |

A plan error when connecting (the server requires a minimum plan) means the workspace needs an upgrade at https://www.pinbridge.io.

## `error_code` on a failed or deferred pin

`get_pin` and `list_pins` show why a pin didn't publish. Nothing is retried for a failed pin until `retry_pin` is called.

| `error_code` | What it means | What to do |
|---|---|---|
| `rate_limited`, `quota_exceeded` on a `deferred` pin | PinBridge is holding the pin until there's room. | Nothing. It publishes on its own; tell the user it's waiting. |
| `token_expired`, `token_revoked`, `scope_missing` | The Pinterest connection broke before the publish. | After the user reconnects in the dashboard, `retry_pin`. Or `retry_pin` with another `account_id`. |
| `board_access_denied` | Pinterest wouldn't accept the board. | `check_board_access`, then `retry_pin` with a working `board_id`. |
| `resource_not_found` | Pinterest couldn't find something the pin needs, usually the board. | `check_board_access` first. If the board is fine, check the image link, then `retry_pin`. |
| `invalid_image` | Pinterest couldn't read the image. | The image can't be changed on an existing pin. Publish a new pin with a valid image, and offer to delete the failed one. |
| `media_url_unreachable` | Pinterest couldn't download `image_url`. | Check the URL is public, or upload the file with `upload_asset`, then publish a new pin. |
| `asset_not_found`, `missing_video_asset` | The uploaded file behind `asset_id` is gone. | Upload it again and publish a new pin. |
| `invalid_payload` | Pinterest rejected a field. | Read `error_message`, fix it with `update_pin`, then `retry_pin`. |
| `media_url_temporarily_unavailable`, `transient_upstream`, `publish_timeout`, `internal_error` | A temporary failure. | `retry_pin` once. If it fails the same way again, tell the user. |

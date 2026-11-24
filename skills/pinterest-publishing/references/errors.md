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
| `invalid_video` | The uploaded video isn't a video file PinBridge can read: another format, or broken. | Upload the original MP4 or MOV file. |
| `image_url_unreachable` | `image_url` or `cover_image_url` doesn't open for Pinterest: it needs a sign-in, answers with an error, or (reason `not_an_image`) opens a web page instead of the image file, usually the page the image is shown on. | Use the address of the image itself: open the page, right-click the image, choose "Copy image address". It must open in a private browser window without signing in. Or upload the file with `upload_asset` and pass `asset_id`. |
| `media_type_mismatch` | `image_url` or `cover_image_url` opens a video, not an image, for example a Google Drive download link to an MP4. | Give a link to an image file. To publish the video itself, make it a video pin. |
| `pinterest_pin_link` | `link_url` is a pin that is already on Pinterest (`pinterest.com/pin/...`). Pinterest doesn't let apps save an existing pin again. Profile and board links are fine. | Use the website the idea comes from as the link, or publish without a link. To keep someone else's pin, the user clicks Save on it in Pinterest. |
| `invalid_date_range`, `invalid_timezone` | A report's range is backwards or longer than 366 days, or the timezone isn't an IANA name. | Fix the range, or use a name like `Europe/Paris`. |

### Templates

| Code | What it means | What to do |
|---|---|---|
| `templates_not_available` | Pin templates aren't turned on for this workspace yet. | Make the pin from an image instead (`image_url` or `asset_id`). |
| `template_inputs_invalid` | Some `template_inputs` don't fit the template. `problems` lists each one by input: `required`, `too_long`, `wrong_type`, `photo_unreachable` and similar. | Fix every listed input and dry-run again. `get_template` shows each input's type and limits. A `photo_unreachable` photo link needs a sign-in or is broken: use another link or upload the file with `upload_asset`. |
| `template_not_found`, `template_version_not_found` | The template isn't in this workspace or the gallery, or has no such version. | Call `list_templates` again and use a current `template_id`. Leave `template_version` out to use the current version. |
| `template_photos_too_large` | The photos are too large to draw together. | Send smaller photos; 2000 px on the long side is plenty. |
| `template_image_unreadable` | A photo isn't an image PinBridge can read. | Use a PNG, JPEG or WebP image. |
| `template_file_missing`, `template_not_drawable` | The template uses a font or image that was deleted, or this version can't be drawn. | The user fixes the template in the PinBridge app's template builder. Meanwhile, use another template or an image. |
| `render_busy` | Every renderer is busy drawing other pins. | Wait the seconds the error gives, then send the same call again with the same `idempotency_key`. |
| `render_unavailable` | Pins can't be drawn from templates right now. | Try again in a few minutes, or publish with an image instead. |

### Plan and billing

| Code | What it means | What to do |
|---|---|---|
| `feature_unavailable` | The workspace's plan doesn't include this feature, for example video uploads or bulk publishing. The free Playground plan has neither. | Videos: point the user to a paid plan. Bulk: publish with `create_pin` per pin, or schedule them. |
| `image_upload_limit` | The free Playground plan has uploaded its 5 images this month. | Ask for a public image URL instead, wait for the reset date in the error, or point the user to a paid plan. |
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

## Warnings on a new or edited schedule

`create_schedule` and `update_schedule` can return a `warnings` list. The schedule was still created or changed. Tell the user what each warning means and offer the fix; don't treat it as a failure.

| Code | What it means | What to do |
|---|---|---|
| `media_link_may_expire` | The link in `field` (`image_url` or `cover_image_url`) is a temporary one, such as a Canva download, a Dropbox temporary link or a signed link, and may stop working before `run_at`. `expires_at` says when, if the link shows it. The pin would then fail. | Offer to upload the file with `upload_asset` and put its `asset_id` on the schedule with `update_schedule`, or to use a permanent link. |
| `quota_may_be_exceeded` | The month the schedule runs in already has as many pins as the plan includes (`limit`, `used`). | Tell the user the pin may wait for the monthly reset or extra credits. `recommended_plan_name` and `upgrade_url` name the plan that would cover it. |

## `error_code` on a failed or deferred pin

`get_pin` and `list_pins` show why a pin didn't publish. `error_message` usually says what to check; pass it on in plain words. Nothing is retried for a failed pin until `retry_pin` is called.

| `error_code` | What it means | What to do |
|---|---|---|
| `rate_limited`, `quota_exceeded` on a `deferred` pin | PinBridge is holding the pin until there's room. | Nothing. It publishes on its own; tell the user it's waiting. |
| `pinterest_unreachable` | PinBridge couldn't reach Pinterest because of a network problem. On a `deferred` pin it is still retrying, every hour for about 8 hours. On a `failed` pin those retries ran out. | Deferred: nothing to do. Failed: `retry_pin` later. If it keeps failing, tell the user. |
| `pinterest_connection_lost` | The connection dropped after the pin was sent, so Pinterest may have created it. PinBridge doesn't retry, so the pin isn't posted twice. | Ask the user to look at the board on Pinterest first. `retry_pin` only if the pin isn't there. |
| `token_expired`, `token_revoked`, `scope_missing` | The Pinterest connection broke before the publish. | After the user reconnects in the dashboard, `retry_pin`. Or `retry_pin` with another `account_id`. |
| `board_access_denied` | Pinterest wouldn't accept the board. | `check_board_access`, then `retry_pin` with a working `board_id`. |
| `board_not_found` | The board no longer exists on Pinterest; it was probably deleted there. | `list_boards`, then `retry_pin` with a `board_id` the account still has, or create a new board if the user asks. |
| `resource_not_found` | Pinterest couldn't find something the pin needs. A missing board shows as `board_not_found` instead. | `check_board_access` first. If the board is fine, check the image link, then `retry_pin`. |
| `pinterest_pin_link` | `link_url` is a pin that is already on Pinterest, and Pinterest won't save an existing pin again. | Change `link_url` with `update_pin` to the website the idea comes from, then `retry_pin`. |
| `pinterest_site_refused` | Pinterest answered "This site doesn't allow you to save Pins". Despite the wording, it usually means Pinterest couldn't load the image or the link's page: an expired or sign-in image link, or a site that was down or blocking Pinterest. | Ask the user to open `image_url` and `link_url` in a private browser window. If both open, `retry_pin`. A bad `link_url` can be changed with `update_pin` before `retry_pin`; a bad image needs a new pin, ideally with an uploaded file. |
| `invalid_image` | Pinterest couldn't read the image. With a link, it often opens a web page, a video or a sign-in page instead of the image file. With an uploaded file, the file is broken. | The image can't be changed on an existing pin. Publish a new pin with the image's own address (open the page, right-click the image, "Copy image address") or an uploaded file, and offer to delete the failed one. |
| `media_url_unreachable` | Pinterest couldn't download `image_url`: the link needs a sign-in, has expired, or the site blocks Pinterest. | Check the link opens in a private browser window without signing in. If it does, `retry_pin`. If not, upload the file with `upload_asset` and publish a new pin. |
| `video_processing_timeout` | Pinterest was still processing the video after 5 minutes. | `retry_pin` once. If it times out again, publish a new pin with a shorter or smaller video. |
| `video_processing_failed` | Pinterest couldn't process the video. `error_message` carries Pinterest's code when it gave one. | Publish a new pin with a video Pinterest accepts: MP4 or MOV, H.264 or H.265, 4 seconds to 15 minutes. |
| `asset_not_found`, `missing_video_asset` | The uploaded file behind `asset_id` is gone. | Upload it again and publish a new pin. |
| `invalid_payload` | Pinterest rejected a field. | Read `error_message`, fix it with `update_pin`, then `retry_pin`. |
| `media_url_temporarily_unavailable`, `transient_upstream`, `publish_timeout`, `internal_error` | A temporary failure. | `retry_pin` once. If it fails the same way again, tell the user. |

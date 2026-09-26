# PinBridge plugin for Claude

Publish, schedule and track Pinterest pins by asking Claude. The plugin connects Claude to the hosted PinBridge MCP server and adds a skill that tells Claude how to use it: dry run first, confirm with you, then publish, and never post the same pin twice on a retry.

You need a PinBridge account with at least one Pinterest account connected in the [dashboard](https://app.pinbridge.io). The free Playground plan works with images at a public URL. Uploading images or videos from your computer, and bulk publishing, need a paid plan.

## Install

In Claude Code:

```
/plugin marketplace add pinbridge/pinbridge-skill
/plugin install pinbridge@pinbridge
```

The first time Claude calls a PinBridge tool, it opens a sign-in page. Log in, pick your workspace, and you're connected. There's no API key to paste. The key that sign-in creates shows up in your dashboard key list, and you can revoke it there any time.

Using a client without OAuth support? Point it at `https://mcp.pinbridge.io` and send a PinBridge API key as `Authorization: Bearer pb_...`. The [setup guide](https://www.pinbridge.io/docs/mcp/setup/) covers both ways.

## What you can ask for

- "Pin this image to my Recipes board, linking to my post."
- "Schedule these 20 pins over the next two weeks, two a day at 9am Casablanca time."
- "Why did my last pin fail?"
- "How did publishing go last week?"
- "Which of last month's pins got the most clicks?"
- "Move tomorrow's scheduled pin to the Dinner Ideas board and fix the typo in the title."

Claude shows you the final title, description, link and board before anything goes out, and waits for your OK.

Pinterest doesn't let apps edit a pin once it's live. To correct a published pin, Claude offers to delete it and publish a fixed one, and asks you first.

## What's in here

```
.claude-plugin/plugin.json       plugin manifest
.claude-plugin/marketplace.json  lets Claude Code install it from this repo
.mcp.json                        the PinBridge MCP server (https://mcp.pinbridge.io)
skills/pinterest-publishing/     the publishing workflow, plus an error-code reference
```

The plugin has no hooks and runs no local code. Everything goes through the hosted server, scoped to the workspace you sign in to.

Connecting or disconnecting a Pinterest account stays in the dashboard. Pinterest needs you to approve that in a browser, and disconnecting drops every pending schedule on the account.

## Other assistants

The skill in `skills/pinterest-publishing/` doesn't depend on Claude. It's plain Markdown written against the PinBridge MCP server, so any assistant that can connect to `https://mcp.pinbridge.io` and load skills or custom instructions can use it.

## Links

- [MCP docs](https://www.pinbridge.io/docs/mcp/)
- [Usage policy](https://www.pinbridge.io/docs/mcp/usage-policy/)
- [PinBridge](https://www.pinbridge.io)

## License

MIT. See [LICENSE](LICENSE).

---
name: postlybee
description: Schedule, draft, and publish social media posts with PostlyBee across Instagram, Facebook, LinkedIn, X, TikTok, YouTube, Pinterest, Threads, Bluesky, Discord, Telegram, WordPress, Dev.to, and more. Use when the user wants to post to social media, plan a content calendar, find open posting slots, upload images or videos for a post, check scheduled or failed posts, or read social analytics through PostlyBee.
---

# PostlyBee

PostlyBee schedules and publishes posts to the social accounts a user has
connected in their PostlyBee workspace. You can reach it two ways:

- **MCP tools** named `postlybee` (for example `integrationList` or
  `integrationSchedulePostTool`). If they are available in this session, prefer
  them for listing channels, scheduling, and analytics. They cannot upload
  files from disk.
- **The `postlybee` CLI**, described below. Use it when MCP tools are not
  available, when you need to upload a local file, or when you are writing a
  script.

## Rules

1. **Confirm before anything goes out.** Before creating a post, show the user
   the channels, the text for each one, any media, and the time, and wait for
   a clear yes. Only skip this when the user has already approved that exact
   content in this conversation.
2. **Prefer drafts when unsure.** If the user has not said when to publish, or
   you are running without a person watching, create a draft
   (`"type": "draft"`).
3. **Never delete.** Do not run `posts:delete`, `accounts:delete`, or
   `accounts:connect` unless the user explicitly asks for that exact action.
   `posts:delete` does not ask for confirmation.
4. **Never handle the token yourself.** Do not ask the user to paste an API
   token into the chat, and do not print or echo it. Ask them to run
   `postlybee auth:login` themselves, or to set `POSTLYBEE_API_TOKEN` in their
   environment.
5. **Check the platform before writing.** Run `accounts:settings` for each
   channel before you write the post, and keep within its `maxLength` and
   `rules`.

## Setup

Check whether the CLI is installed and signed in:

```bash
postlybee --version
postlybee --raw-json auth:status
```

If the command is missing, install it (Node.js 18 or newer):

```bash
npm install -g postlybee-cli
```

If it is not signed in, ask the user to create a `pbk_` API token in PostlyBee
under **Settings → Public API** and then run `postlybee auth:login` in their own
terminal. In CI, they set these instead:

```bash
export POSTLYBEE_API_TOKEN="pbk_..."        # from a secret, never committed
export POSTLYBEE_WORKSPACE_ID="workspace-id"
```

If the token can reach more than one workspace, pass `--workspace <id>` or set
`POSTLYBEE_WORKSPACE_ID`. `postlybee --raw-json workspaces:list` shows the
choices and each workspace's timezone.

Always add `--raw-json` so output is one line of JSON you can parse.

## Workflow: create a post

### 1. Find the channels

```bash
postlybee --raw-json accounts:list
```

Each account has an `id` (use this in posts), a `name`, an `identifier` (the
platform, such as `x`, `linkedin`, or `instagram`), and `disabled`. Skip
disabled accounts and tell the user they need to reconnect them in PostlyBee.

### 2. Read the platform rules and settings

```bash
postlybee --raw-json accounts:settings ACCOUNT_ID
```

The `output` has `rules` (plain-language posting rules), `maxLength`,
`settings` (the provider settings schema, such as a YouTube title or a
Pinterest board), and `tools` (methods that load dynamic options). Fill in any
required settings before creating the post.

### 3. Load dynamic options when a setting needs them

```bash
postlybee --raw-json accounts:trigger ACCOUNT_ID METHOD_NAME
postlybee --raw-json accounts:trigger ACCOUNT_ID METHOD_NAME --data '{"key":"value"}'
```

Use a `methodName` from `tools`, for example `boards` for Pinterest.

### 4. Upload media

Posts only accept media that PostlyBee already stores. Upload first, then use
the returned `id` and `path`:

```bash
postlybee --raw-json media:upload ./image.png          # image or MP4 from disk
postlybee --raw-json media:upload-url https://example.com/photo.jpg
```

Instagram, TikTok, YouTube, and Pinterest need at least one image or video.

### 5. Pick a time

- A time the user gave you: convert it to an ISO 8601 UTC string, using the
  workspace timezone from `workspaces:list` when the user gave a local time.
- The next open slot for a channel: `postlybee --raw-json find-slot ACCOUNT_ID`
  returns `{"date": "2026-10-06T09:00:00.000Z"}` in UTC, or `null` when no
  slot is open. If a date comes back without `Z`, treat it as UTC. Convert it
  to the workspace timezone when you show it to the user.
- Let PostlyBee choose: set `"autoSchedule": true` and leave out `date`. Only
  do this when the user wants the post scheduled. To save a draft in the next
  open slot, call `find-slot` and pass that `date` with `"type": "draft"`.

### 6. Show the user, then create

After the user confirms, send the payload through stdin:

```bash
postlybee --raw-json posts:create --stdin <<'JSON'
{
  "type": "schedule",
  "date": "2026-10-06T09:00:00.000Z",
  "shortLink": false,
  "tags": [],
  "posts": [
    {
      "integration": { "id": "ACCOUNT_ID" },
      "value": [{ "content": "Post text", "image": [] }],
      "settings": {}
    }
  ]
}
JSON
```

The response lists a `postId` for each channel. Tell the user what was
created and when it will publish. See
[references/post-payload.md](references/post-payload.md) for threads, media,
several channels, drafts, and repeating posts.

## Other tasks

| Task | Command |
| --- | --- |
| List posts (defaults to 30 days back and forward) | `postlybee --raw-json posts:list --start 2026-10-01T00:00:00Z --end 2026-10-31T23:59:59Z` |
| Move a post to draft or back to scheduled | `postlybee --raw-json posts:status POST_ID --status draft` |
| Change settings on an unpublished post | `postlybee --raw-json posts:settings POST_ID --data '{"key":"value"}'` |
| Account analytics (both dates required) | `postlybee --raw-json analytics:account ACCOUNT_ID --start 2026-09-01T00:00:00Z --end 2026-09-30T23:59:59Z` |
| Analytics for one published post | `postlybee --raw-json analytics:post POST_ID --days 30` |
| Workspace notifications, such as failed posts | `postlybee --raw-json notifications:list` |
| Check the token and workspace | `postlybee --raw-json ping` |

## Errors

Failed commands exit with code 1 and print JSON such as
`{"error": "...", "status": 403, "details": {...}, "retryAfter": null}`.

- **401**: the token is missing or invalid. Ask the user to run
  `postlybee auth:login` again.
- **403 with `requiredScopes`**: the token lacks a scope. Tell the user which
  scope to add to the token in Settings → Public API.
- **400**: the payload failed validation. Read `details`, fix the payload, and
  try again.
- **429**: wait for `retryAfter` seconds before retrying. Do not retry in a
  tight loop.

## Token scopes

| Commands | Scope |
| --- | --- |
| `workspaces:list`, `ping` | `workspaces:read` |
| `accounts:list`, `accounts:settings`, `accounts:trigger`, `find-slot` | `accounts:read` |
| `posts:list` | `posts:read` |
| `posts:create`, `posts:status`, `posts:settings`, `posts:delete` | `posts:write` |
| `media:*` | `media:write` |
| `analytics:*` | `analytics:read` |
| `notifications:list` | `notifications:read` |

For research and reporting, a read-only token is enough. Suggest the user add
`posts:write` and `media:write` only where you should create posts.

# PostlyBee post payload

`postlybee posts:create` takes one JSON object through `--stdin`, `--file`, or
`--data`. Use exactly one of them.

## Fields

| Field | Required | Notes |
| --- | --- | --- |
| `type` | Yes | `"draft"`, `"schedule"`, or `"now"` |
| `date` | Yes, unless `autoSchedule` is `true` | ISO 8601 date string, for example `"2026-10-06T09:00:00.000Z"`. Needed for drafts too |
| `shortLink` | Yes | `true` shortens links in the post, `false` leaves them as they are |
| `tags` | Yes | Array of `{ "value": "...", "label": "..." }`. Use `[]` for none |
| `posts` | Yes | One entry per channel (see below) |
| `autoSchedule` | No | `true` lets PostlyBee pick the next open slot. Use it with `"type": "schedule"` |
| `autoScheduleOptions` | No | `{ "scheduleAfter": "<date>", "scheduleBefore": "<date>", "topOfQueue": false }` |
| `repeat` | No | `{ "enabled": true, "times": 3, "intervalValue": 1, "intervalUnit": "WEEK" }`. `times` 1–30, `intervalValue` 1–365, unit `DAY`, `WEEK`, or `MONTH` |
| `recycle` | No | `{ "enabled": true, "intervalValue": 30, "intervalUnit": "DAY", "maxCycles": 5 }` |

Each entry in `posts`:

| Field | Required | Notes |
| --- | --- | --- |
| `integration.id` | Yes | Account `id` from `accounts:list` |
| `value` | Yes | Array of `{ "content": "...", "image": [] }`. More than one item makes a thread or follow-up comments on platforms that support them |
| `settings` | Yes | Provider settings from `accounts:settings`. Use `{}` when the platform needs none. PostlyBee adds the platform type for you |

Each item in `image` is a media object from `media:upload` or
`media:upload-url`: `{ "id": "...", "path": "..." }`, with optional `alt`.

## Examples

### Schedule one post to two channels, each with its own text

```json
{
  "type": "schedule",
  "date": "2026-10-06T09:00:00.000Z",
  "shortLink": false,
  "tags": [],
  "posts": [
    {
      "integration": { "id": "LINKEDIN_ACCOUNT_ID" },
      "value": [{ "content": "Longer LinkedIn version of the announcement.", "image": [] }],
      "settings": {}
    },
    {
      "integration": { "id": "X_ACCOUNT_ID" },
      "value": [{ "content": "Short X version.", "image": [] }],
      "settings": {}
    }
  ]
}
```

### Save a draft with an uploaded image

Upload first:

```bash
postlybee --raw-json media:upload ./launch.png
# {"id":"MEDIA_ID","name":"launch.png","path":"https://.../launch.png"}
```

Then use the returned `id` and `path`:

```json
{
  "type": "draft",
  "date": "2026-10-06T09:00:00.000Z",
  "shortLink": false,
  "tags": [],
  "posts": [
    {
      "integration": { "id": "INSTAGRAM_ACCOUNT_ID" },
      "value": [
        {
          "content": "Our new collection is here.",
          "image": [{ "id": "MEDIA_ID", "path": "https://.../launch.png", "alt": "Three jackets on a rail" }]
        }
      ],
      "settings": {}
    }
  ]
}
```

### A thread

Each item in `value` becomes the next post in the thread:

```json
{
  "type": "schedule",
  "date": "2026-10-08T15:00:00.000Z",
  "shortLink": false,
  "tags": [],
  "posts": [
    {
      "integration": { "id": "X_ACCOUNT_ID" },
      "value": [
        { "content": "1/3 We rebuilt our search from scratch. Here is what changed.", "image": [] },
        { "content": "2/3 Results now load in under 100 ms.", "image": [] },
        { "content": "3/3 Try it today and tell us what you think.", "image": [] }
      ],
      "settings": {}
    }
  ]
}
```

### Let PostlyBee pick the time

This schedules the post for publishing. For a draft in the next open slot, use
the `date` from `find-slot` with `"type": "draft"` instead.

```json
{
  "type": "schedule",
  "autoSchedule": true,
  "autoScheduleOptions": { "scheduleAfter": "2026-10-06T00:00:00.000Z" },
  "shortLink": false,
  "tags": [],
  "posts": [
    {
      "integration": { "id": "LINKEDIN_ACCOUNT_ID" },
      "value": [{ "content": "Tip of the week: ...", "image": [] }],
      "settings": {}
    }
  ]
}
```

### Platform settings

Read the schema with `accounts:settings ACCOUNT_ID` and put the values in
`settings`. Two common cases:

- **YouTube** needs a `title` (2–100 characters) and a `type` for visibility:
  `"public"`, `"private"`, or `"unlisted"`.
- **Pinterest** needs a `board`. Load the choices with
  `accounts:trigger PINTEREST_ACCOUNT_ID boards`, which returns boards as
  `{ "name": "...", "id": "..." }`, then set `"board": "<id>"`. `title` and
  `link` are optional.

When a platform's settings are not clear from the schema, ask the user rather
than guessing, or save the post as a draft so they can finish it in PostlyBee.

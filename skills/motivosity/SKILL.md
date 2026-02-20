---
name: motivosity
description: Interact with the Motivosity employee recognition platform. Search users, send appreciations, give awards, view feeds, check leaderboard, manage org chart, and more via the Motivosity API.
---

# Motivosity Skill

Interact with Motivosity's API to recognize employees, manage awards, view feeds, and more.

## Getting Started

**First thing:** Bootstrap the session to get the current user's ID and company ID:

```
GET /api/private/context?page=%2Fhome
```

This returns `response.user.id`, `response.company.id`, and feature flags. Do this before any other call so you have the IDs you need.

## Authentication

This skill requires browser access via the **chrome-devtools MCP**. Make sure `app.motivosity.com` is open in Chrome before proceeding.

### chrome-devtools MCP

**Step 1: Select the Motivosity tab**
```
list_pages → select_page (pick the app.motivosity.com tab)
```

**Step 2: Reload and grab a real request's Cookie and xct headers**
```
navigate_page (reload) → list_network_requests → get_network_request (pick any /api/private/ request)
```

Note the full `cookie` and `xct` request header values. These are needed to understand what the browser is sending.

**Step 3: Make API calls via evaluate_script**

Use `evaluate_script` to make `fetch` calls from within the browser — this sends the browser's own cookies automatically:

```js
async () => {
  const res = await fetch('/api/private/context?page=%2Fhome', {
    headers: { 'Accept': 'application/json' }
  });
  return await res.json();
}
```

> **Why evaluate_script:** Cookies (including httpOnly ones like `JSESSIONID`) are sent automatically by the browser. No need to extract or pass them manually.

**On 401/403:** Reload the page (`navigate_page reload`) and retry.

### Service account (admins only)

Admins can create API keys at **Setup → Integrations → API Key → + New API Key**. This gives you an `APP_ID` and `APP_SECRET` (also called `SECURE_TOKEN`).

Exchange for a short-lived access token (15 min):

```
POST https://app.motivosity.com/auth/v1/servicetoken
Content-Type: application/json

{"appId":"<APP_ID>","secureToken":"<APP_SECRET>"}
```

Returns `{"accessToken":"...","expiresIn":"900"}`. Use as `Authorization: Bearer <accessToken>`. Refresh on 401.

### OAuth2 (admins — web app integrations)

For client-side integrations, Motivosity supports OAuth2:

- **Authorization:** `GET /oauth2/v1/auth`
- **Token exchange/refresh:** `POST /oauth2/v1/token`

See the [Motivosity API docs](https://help.motivosity.com/en/articles/9044579-motivosity-api) and the [reference implementation](https://github.com/scjnsn/Motivosity-Demo-app) for the full OAuth2 flow.

---

## Base URLs

```
https://app.motivosity.com/api/v2        # public documented API
https://app.motivosity.com/api/private   # private internal API
```

Endpoint paths below are relative to `https://app.motivosity.com`. `/api/v2/` paths use the public API; `/api/private/` paths require browser session auth.

---

## Endpoints

### Bootstrap

```
GET /api/private/context?page=%2Fhome
```

Returns current user ID, company ID, permissions, currency settings, and feature flags. **Always call this first.**

Key response fields:
- `response.user.id` — your userId for subsequent calls
- `response.company.id` — companyId
- `response.preference.monthlyPeerToPeerBonus` — how many bucks you can give per month
- `response.preference.currencyNickname` — e.g. "Motivosity Buck"
- `response.permission.recognizeLicense` — whether you can send appreciations

### Users

| Task | Method | Path |
|------|--------|------|
| Search by name (typeahead) | GET | `/api/private/usertypeahead?name=<query>&ignoreSelf=true&includeUserGroups=false` |
| My cash balances | GET | `/api/private/usercash` |

> **Note:** `/api/v2/user/search` requires a Bearer token and won't work with session cookies. Use `/api/private/usertypeahead` instead — it works with the cookie-based auth and returns user IDs needed for appreciations.

### Appreciations

**Get company values** (included in appreciation setup response):
```
GET /api/private/appreciation/setup
```
Returns `response.values[]` with `id` and `name` for each value.

**Send appreciation:**
```
PUT /api/private/post?type=APPR
```

```json
{
  "options": {
    "isPrivate": false,
    "postAsPlatform": false,
    "scheduledDate": null,
    "postAsUserId": null,
    "featureUntilDate": null
  },
  "noteText": "Great work on the project!",
  "recipients": [
    {
      "userId": "<target-user-id>",
      "amount": 1,
      "purseType": "GM",
      "giftId": null
    }
  ],
  "otherProperties": {
    "companyValueId": "<value-id>"
  }
}
```

- `purseType`: `GM` = giving money
- Minimum amount is `1` (whole dollar)
- `userId` from `usertypeahead` search

### Awards

| Task | Method | Path |
|------|--------|------|
| List available awards | GET | `/award?awardScope=RLVT` |
| My received awards | GET | `/award/myawards` |
| Give award | PUT | `/award/<awardId>/giveaward` |

```json
{
  "toUserEmails": ["user@company.com"],
  "note": "Congrats!",
  "amount": "10"
}
```

### Feed

| Task | Method | Path |
|------|--------|------|
| Recent company feed | GET | `/feed?scope=CMPY&page=0&pageLimit=10` |
| Feed by date range | GET | `/feed/all?startDate=2024-01-01&endDate=2024-12-31` |
| Specific feed item | GET | `/feed/<feedId>` |
| Like | PUT | `/feed/<feedId>/like` |
| Comment | PUT | `/feed/<feedId>/comment` |
| Delete (admin) | DELETE | `/feed/<feedId>` |

Feed scopes: `TEAM`, `EXTM` (extended team), `DEPT`, `CMPY`

Feed types: `APPR` (appreciation), `BDGE` (award), `BDAY`, `ANVY`, `ANNC`, `GNRL`, `HGLT`

### Leaderboard

```
GET /leaderboard?type=month    # month | year | top10
```

### Org Chart

| Task | Method | Path |
|------|--------|------|
| Find user in org | GET | `/orgchart/find?userId=<id>` |
| Supervisor | GET | `/orgchart/sup?userId=<id>` |
| Direct reports | GET | `/orgchart/dr?userId=<id>` |
| Subtree (N levels) | GET | `/orgchart/partial?userId=<id>&levels=3` |

### Comments

```
POST /comment                     {"commentText":"...", "feedId":"..."}
POST /comment/<commentId>         {"commentText":"..."}   # edit
DELETE /comment/<commentId>
```

### Store

```
GET  /store?type=digital          # digital | charity | local
POST /store/purchase              {"id":"<itemId>", "currentQty":1}
```

### User Relationship (Private)

```
GET /api/private/permission/relation?userId=<userId>
```

Returns whether the target user is a descendant in your org tree. Useful before sending appreciation.

```json
{
  "currentUserId": "<currentUserId>",
  "subjectUserId": "<targetUserId>",
  "isDescendant": false,
  "isNoteCreated": false
}
```

### Admin / Sync

```
POST /user/sync                   # bulk sync users
POST /user/<id>/move              # move user in org chart
GET  /user/recalculateorgchart
POST /user/changePayrollID
```

---

## Tips

- **Always bootstrap first** — call `/api/private/context` to get userId, companyId, and monthly budget before doing anything else.
- **Use evaluate_script for all API calls** — fetch from within the browser so cookies are sent automatically. No need to extract or pass cookies manually.
- **Requires chrome-devtools MCP** — `app.motivosity.com` must be open in Chrome.
- **Finding users** — use `GET /api/private/usertypeahead?name=<query>` (not `/api/v2/user/search` which requires Bearer auth).
- **Company values** — use `GET /api/private/appreciation/setup` which returns values alongside other setup data in one call.
- **Send appreciations** — use `PUT /api/private/post?type=APPR` (not `/api/private/appreciation` which 404s).
- **Minimum appreciation amount** — $1 (whole dollars only).
- **Feed pagination** — `page=0,1,2...` with `pageLimit` (default 10, max ~50).
- **401/403** — reload the page, grab a fresh network request, re-extract `xct` and cookies.

---

## References

- [Motivosity API docs](https://help.motivosity.com/en/articles/9044579-motivosity-api)

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

Most users don't have a service account. Default to the DevTools approach.

### DevTools (most users)

Ask the user to do this once while logged in to `app.motivosity.com`:

> "I need two things from your browser DevTools:
>
> **1. xct token** — Open DevTools (`F12`) → Network tab → reload the page → click any request to `/api/` → look under Request Headers → copy the `xct` value.
>
> **2. Cookies** — DevTools → Application tab → Cookies → `https://app.motivosity.com` → copy the **Value** for:
> - `MONSESSION`
> - `AWSALB`
> - `AWSALBCORS`
>
> Paste all four values here and I'll take it from there."

Use on every request:
```
Cookie: MONSESSION=<value>; AWSALB=<value>; AWSALBCORS=<value>
xct: <value>
Content-Type: application/json
```

**Credential lifetime:**
- `MONSESSION` — days/weeks, tied to browser session
- `xct` — short-lived CSRF JWT, expires after ~15–30 min of inactivity
- `AWSALB`/`AWSALBCORS` — 7-day expiry

**On 401/403:** Re-extract `xct` first (most common expiry). Still failing → re-extract all cookies.

### chrome-devtools MCP (if available)

If the `chrome-devtools` MCP is configured and `app.motivosity.com` is open in Chrome, skip manual extraction entirely. Use `list_pages` → `select_page` to select the Motivosity tab, then make calls with `evaluate_script`:

```javascript
async () => {
  const xct = document.cookie
    .split(';')
    .find(c => c.trim().startsWith('__Host-X-CSRF-Token='))
    .split('=').slice(1).join('=');

  const resp = await fetch('https://app.motivosity.com/api/private/context?page=%2Fhome', {
    credentials: 'include',  // sends httpOnly MONSESSION automatically
    headers: { 'xct': xct }
  });
  return await resp.json();
}
```

Notes:
- `MONSESSION` is httpOnly — can't read it via JS, but `credentials: 'include'` sends it automatically
- `xct` = value of the `__Host-X-CSRF-Token` cookie
- `/api/private/*` works well this way; `/api/v2/*` may 401 (use private equivalents)

### Service account (admins only)

```
POST https://app.motivosity.com/auth/v1/servicetoken
Content-Type: application/json

{"appId":"<APP_ID>","secureToken":"<SECURE_TOKEN>"}
```

Returns `{"accessToken":"...","expiresIn":"900"}`. Use as `Authorization: Bearer <accessToken>`. Token lasts 15 min — refresh on 401.

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
| Current user | GET | `/user/me` |
| Search by name/email | GET | `/user/search?name=<query>` |
| List all users | GET | `/user?scope=CMPY&page=0&pageLimit=20` |
| Individual user | GET | `/user/<userId>` |
| User profile | GET | `/user/<userId>/profile` |
| My cash balances | GET | `/usercash` |

### Appreciations

| Task | Method | Path |
|------|--------|------|
| Send appreciation | PUT | `/appreciation` |
| List company values | GET | `/companyvalue` |

**Workflow:** Get company values first → let user pick one → send appreciation.

```json
{
  "amount": 5,
  "amountType": "GM",
  "note": "Great work on the project!",
  "companyValueID": "<value-id>",
  "toUserEmails": ["user@company.com"]
}
```

- `amountType`: `GM` = giving money, `SM` = spending money
- Target by `toUserIDs`, `toUserEmails`, or `toUserPayrollIDs`

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
- **Finding users** — use `/user/search?name=<query>` to get their ID, then use the ID for appreciation/award calls.
- **Company values** — required for appreciations. Fetch with `GET /companyvalue`, match by name or let user pick.
- **Feed pagination** — `page=0,1,2...` with `pageLimit` (default 10, max ~50).
- **401/403** — re-extract `xct` from DevTools first. If still failing, re-extract all cookies.

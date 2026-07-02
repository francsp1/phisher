# Phishing Lab — End-to-End Flow

How a campaign travels through this lab, from launch to captured credentials.

## Components

| Component | Where | Exposed to outside? | Role |
|-----------|-------|---------------------|------|
| **gophish – admin server** | `gophish:3333` (HTTPS) | Yes (per lab choice) | Dashboard: build campaigns, view results |
| **gophish – phish server** | `gophish:8080` (HTTP) | **No** — internal only | Records events (open/click/submit), stores captures |
| **nginx** | `nginx:80` (HTTP) | Yes | Public face: serves the landing page, proxies tracking + submissions to gophish |

The target's browser and mail client **only ever talk to nginx**. gophish's phish
server sits behind it on the internal `phisher-net` and is never directly reachable.

Key parameter: **`rid`** (recipient id) — a per-target token gophish embeds in
every link and pixel. It's how gophish attributes each event to the right person.

---

## Step by step

### 1. The campaign is started in gophish
You launch a campaign in the admin UI (`https://<host>:3333`). It ties together a
**sending profile** (SMTP), an **email template**, a **landing page** (with
*Capture Submitted Data* / *Capture Passwords* enabled), and a **target group**.
The campaign's **URL** field is set to the **nginx** address (`http://<host>`),
**not** `gophish:8080` — this is what gets baked into the email links.

### 2. gophish sends the emails
gophish's background worker connects to the configured SMTP server and sends one
email per target. Into each message it injects that target's unique `rid`:
- the phishing link  → `http://<host>/?rid=<RID>`
- the tracking pixel → `<img src="http://<host>/track?rid=<RID>">` (if enabled)

*(Log: `msg="Email sent" email="..."`)*

### 3. The target opens the email → "Email Opened"
When the mail client renders the message, it tries to load the tracking pixel:

```
target mail client → GET http://<host>/track?rid=<RID>
        → nginx (location /track) → gophish:8080/track?rid=<RID>
        → gophish records "Email Opened", returns a 1x1 GIF
```

⚠️ **Unreliable by design.** Many clients suppress this:
- **Gmail** proxies all images through Google and does **not** pre-fetch pixels
  hidden with `display:none` → no request reaches the server → no open recorded.
- **Apple Mail Privacy Protection** pre-fetches *all* images → **false** opens.
So a missing "Email Opened" usually means the client blocked the pixel, not that
the lab is broken. Confirm the server path works with:
`curl -v 'http://<host>/track?rid=<RID>'`.

### 4. The target clicks the link → "Clicked Link"
The link points at nginx. nginx serves **its own static landing page** — gophish
never sees this GET, so we register the click with a background ping:

```
1. target browser → GET http://<host>/?rid=<RID>
      → nginx (location = /) serves /usr/share/nginx/html/index.html
2. page JavaScript reads rid from the URL, then fires:
      → GET http://<host>/click?rid=<RID>
      → nginx (location = /click) → gophish:8080/click?rid=<RID>
      → gophish's landing handler sees a GET → records "Clicked Link"
   (the response — gophish's own landing HTML — is discarded by fetch())
```

*(Logs: `nginx "GET /?rid=<RID>" 200` immediately followed by
`gophish "GET /click?rid=<RID>" 200`.)*

Note: because we set `X-Forwarded-For`, gophish logs the **real target IP**, not
nginx's internal address.

### 5. The target submits the form → "Submitted Data"
The static page's login form carries the `rid` in a hidden field and POSTs it:

```
target browser → POST http://<host>/login   (email, password, rid=<RID>)
      → nginx (location = /login) → gophish:8080  (path passed through as-is)
      → gophish records "Submitted Data" and stores the captured fields
        (password stored only if "Capture Passwords" was enabled)
      → gophish responds with the campaign's redirect (302) or re-serves the page
      → nginx passes that response back to the target
```

### 6. Results appear in the dashboard
gophish stores every event in its SQLite DB (`/db/gophish.db`, on a Docker
volume). The admin UI timeline for each target now shows the progression:

```
Email Sent → Email Opened → Clicked Link → Submitted Data
```

Captured credentials appear under the result's details.

---

## Quick reference: which path records what

| Target action | HTTP request (to nginx) | Proxied to gophish | Event |
|---------------|-------------------------|--------------------|-------|
| Opens email | `GET /track?rid=` | `/track` | Email Opened |
| Clicks link | `GET /?rid=` then JS `GET /click?rid=` | `/click` | Clicked Link |
| Submits form | `POST /login` (+ `rid`) | `/login` | Submitted Data |

`rid` is the thread that ties every request back to the correct target. Any path
except `/track` and `/robots.txt` is treated by gophish as a landing-page hit —
GET = click, POST = submission — which is why `/click` and `/login` are just
convenient names, not magic ones.

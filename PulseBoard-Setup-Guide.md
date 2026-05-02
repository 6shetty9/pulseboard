# PulseBoard Setup Guide

A personal Tesla monitoring app with sentry alerts via Pushover, SMS, and Email.

---

## Prerequisites

- Tesla account with at least one vehicle
- GitHub account
- Cloudflare account (free)
- Google account (for Cloudflare signup)

---

## Step 1 — Generate your key pair

PulseBoard uses a public/private key pair to authenticate with Tesla.

Run these commands in Terminal:

```bash
openssl ecparam -name prime256v1 -genkey -noout -out private-key.pem
openssl ec -in private-key.pem -pubout -out com.tesla.3p.public-key.pem
```

- **`com.tesla.3p.public-key.pem`** — goes on GitHub (public)
- **`private-key.pem`** — keep safe on your computer, never upload

---

## Step 2 — Create GitHub repos

### Repo 1: `pulseboard` (hosts the app)

1. Go to github.com/new
2. Name it `pulseboard`, set to Public, add README
3. Enable GitHub Pages: Settings → Pages → Deploy from `main` branch

### Repo 2: `6shetty9.github.io` (hosts the public key)

1. Go to github.com/new
2. Name it exactly `6shetty9.github.io`, set to Public, add README
3. GitHub Pages is automatic for this repo type

### Add the public key to the root repo

1. In `6shetty9.github.io`, click "Add file" → "Create new file"
2. Type the filename: `.well-known/appspecific/com.tesla.3p.public-key.pem`
3. Paste in the contents of your `com.tesla.3p.public-key.pem` file
4. Add a `_config.yml` file with this content (required for Jekyll to serve dot-folders):

```yaml
include:
  - .well-known
```

5. Verify the key is live at: `https://6shetty9.github.io/.well-known/appspecific/com.tesla.3p.public-key.pem`

---

## Step 3 — Apply for Tesla Fleet API access

1. Go to [developer.tesla.com](https://developer.tesla.com)
2. Create a developer account and register a new application
3. Fill in:
   - **Application name:** PulseBoard
   - **Description:** Personal vehicle monitoring application for my own Tesla vehicle. Monitors battery, charging, climate, location, and sentry mode. Sends alerts to personal devices. For personal use only.
   - **Allowed Origin:** `https://6shetty9.github.io`
   - **Allowed Redirect URI:** `https://6shetty9.github.io/pulseboard/callback`
   - **OAuth Grant Type:** Authorization Code and Machine-to-Machine
   - **Scopes:** vehicle_device_data, vehicle_cmds, vehicle_charging_cmds, vehicle_location
4. Wait for approval (can take a few days)
5. Save your **Client ID** and **Client Secret**

---

## Step 4 — Set up Cloudflare Worker (CORS proxy)

Tesla's API blocks direct browser requests. The Cloudflare Worker proxies all API calls.

1. Go to [workers.cloudflare.com](https://workers.cloudflare.com) and sign up
2. Create a Worker → "Start with Hello World"
3. Deploy it, then click "Edit code"
4. Replace all code with the worker script (see `tesla-proxy-worker.js`)
5. Hit Deploy
6. Note your worker URL: `https://your-worker-name.workers.dev`

### Worker code (`tesla-proxy-worker.js`)

```javascript
export default {
  async fetch(request, env) {
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    };

    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }

    const url = new URL(request.url);
    const path = url.pathname;

    let targetUrl;
    if (path === '/token') {
      targetUrl = 'https://auth.tesla.com/oauth2/v3/token';
    } else {
      targetUrl = 'https://fleet-api.prd.na.vn.cloud.tesla.com' + path + url.search;
    }

    const headers = {};
    if (request.headers.get('Content-Type')) headers['Content-Type'] = request.headers.get('Content-Type');
    if (request.headers.get('Authorization')) headers['Authorization'] = request.headers.get('Authorization');

    const body = request.method !== 'GET' ? await request.text() : undefined;
    const response = await fetch(targetUrl, { method: request.method, headers, body });
    const data = await response.text();

    return new Response(data, {
      status: response.status,
      headers: { ...corsHeaders, 'Content-Type': response.headers.get('Content-Type') || 'application/json' },
    });
  }
}
```

---

## Step 5 — Upload PulseBoard app files to GitHub

Upload these files to your `pulseboard` repo:

```
pulseboard/
├── index.html        ← main app
├── 404.html          ← routing helper
└── callback/
    └── index.html    ← OAuth callback handler
```

The app will be live at `https://6shetty9.github.io/pulseboard/`

---

## Step 6 — Register your app with Tesla Fleet API (one-time)

This is required before Tesla will respond to API calls. Open PulseBoard, sign in, then open DevTools (Cmd+Option+I) → Console tab and run:

```javascript
(async () => {
  const cid = localStorage.getItem('pb_cid');
  const csec = localStorage.getItem('pb_csec');
  const PROXY = 'https://YOUR-WORKER.workers.dev';

  // Get partner token
  const tr = await fetch(PROXY + '/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'client_credentials',
      client_id: cid,
      client_secret: csec,
      scope: 'openid vehicle_device_data vehicle_cmds vehicle_charging_cmds vehicle_location',
      audience: 'https://fleet-api.prd.na.vn.cloud.tesla.com'
    })
  });
  const td = await tr.json();

  // Register app domain
  const rr = await fetch(PROXY + '/api/1/partner_accounts', {
    method: 'POST',
    headers: { Authorization: 'Bearer ' + td.access_token, 'Content-Type': 'application/json' },
    body: JSON.stringify({ domain: '6shetty9.github.io' })
  });
  const rd = await rr.json();
  console.log('Registration:', JSON.stringify(rd));
})()
```

Success looks like: response containing `client_id`, `domain`, `public_key`, `created_at`.

---

## Step 7 — Sign in to PulseBoard

1. Go to `https://6shetty9.github.io/pulseboard/`
2. Enter your Tesla Client ID and Client Secret
3. Click "Sign in with Tesla"
4. Log in with your Tesla account credentials
5. After approval, the callback page automatically redirects back
6. PulseBoard completes the token exchange and loads your vehicles

---

## Step 8 — Set up alert notifications (optional)

Go to the **Notifications** tab in PulseBoard and configure any or all of:

### Pushover (recommended — fastest setup)
1. Sign up free at [pushover.net](https://pushover.net)
2. Install the Pushover app on your phone
3. Create an application to get an app token
4. Enter your user key and app token in PulseBoard
5. Toggle Pushover on

### SMS via Twilio
1. Sign up at [twilio.com](https://twilio.com) (free trial available)
2. Get a Twilio phone number
3. Enter Account SID, Auth Token, Twilio number, and your number in PulseBoard
4. Toggle SMS on

### Email via SendGrid
1. Sign up at [sendgrid.com](https://sendgrid.com) (free tier: 100 emails/day)
2. Verify your sender email address
3. Create an API key
4. Enter API key, from email, and to email in PulseBoard
5. Toggle Email on

---

## Step 9 — Start monitoring

1. Go to the **Sentry monitor** tab
2. Click **Start monitor**
3. PulseBoard polls your vehicle every 60 seconds
4. Alerts fire to all enabled channels when:
   - Sentry mode turns off unexpectedly
   - Vehicle is unlocked
   - Battery drops below 20%
   - Vehicle goes offline

---

## Architecture overview

| Service | Role |
|---|---|
| GitHub Pages (`pulseboard`) | Hosts the PulseBoard web app |
| GitHub Pages (`6shetty9.github.io`) | Hosts the Tesla public key at `/.well-known/` |
| Cloudflare Worker | CORS proxy — routes token exchange and Fleet API calls |
| Tesla Auth (`auth.tesla.com`) | OAuth 2.0 + PKCE authentication |
| Tesla Fleet API | Vehicle data and commands |
| Pushover / Twilio / SendGrid | Alert delivery to your phone |

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Load failed` on token exchange | CORS blocking direct browser request | Ensure Cloudflare worker is deployed with `Access-Control-Allow-Origin: *` |
| `412` on `/vehicles` | App not registered with Fleet API | Run the partner registration script (Step 6) |
| `The string did not match the expected pattern` | Missing PKCE code verifier | Sign out, clear localStorage, sign in fresh |
| `Could not reach vehicle` | Vehicle is asleep | Click "Wake vehicle" button and wait 30 seconds |
| `No vehicles found` | Vehicle ID is null | Sign out, clear localStorage (`pb2` key), sign in fresh |
| Public key 404 | Jekyll blocking `.well-known` folder | Add `_config.yml` with `include: [.well-known]` |
| `Unable to Grant Third-Party Access` | App not fully registered | Complete Step 6 partner registration first |

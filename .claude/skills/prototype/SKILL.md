---
name: prototype
description: Build a working app prototype and deploy it to the tailnet. Use when the user wants to create a new frontend application, personal tool, dashboard, or UI. Handles app creation, local dev serving, systemd service setup for auto-start on boot, and Tailnet access via Tailscale. Triggers on "build an app", "create a prototype", "make a [thing] app", "new app", "prototype", "build me a", "I want an app that".
---

# App Prototyping

Build a working frontend application from a description — no mockups, no design docs. Just describe what it should do and get a working prototype running on the tailnet.

**Default stack**: Single-file HTML + CDN React + vanilla Node.js server
**Full-featured stack**: Vite + React + TypeScript + Tailwind + shadcn/ui
**Storage**: JSON files (simple) or SQLite via better-sqlite3 (structured)
**Hosting**: Systemd user service, accessible via Tailscale

---

## 1. Understand the Request

Ask the user:
- What does the app do? What problem does it solve?
- Who's the audience (just Owen, Owen + Emma, others on the tailnet)?
- Does it need to persist data, or is stateless OK?
- Any specific UI feel or reference apps in mind?

If the description is clear enough to infer these, skip the questions and proceed.

---

## 2. Choose a Stack

Use AskUserQuestion to offer these options — or choose automatically based on complexity:

### Pattern A: Simple (Single-File)
Best for: personal tools, single-view dashboards, quick utilities

```
index.html       # React via CDN (no build step), all JS/CSS inline
server.js        # Vanilla Node.js HTTP server, 0-dependency persistence
data/            # JSON files for storage
```

- CDN React 18 (UMD) + Babel standalone for JSX transpilation
- Custom CSS inline (no framework)
- Node.js stdlib only — no npm install needed beyond optional packages
- Great for: single-view tools, calendars, trackers, planners

### Pattern B: Vite + React (Full-Featured)
Best for: multi-view apps, complex state, reusable components

```
src/             # React components, config, lib
server.js        # Express or vanilla Node.js API server
data/            # JSON files or SQLite DB
package.json     # npm-managed
```

- Vite + React 18 + TypeScript
- Tailwind CSS + shadcn/ui components
- Framer Motion for animations
- better-sqlite3 or JSON files for persistence

**Rule of thumb**: If the app has 1-2 views and relatively static data structures, use Pattern A. If it has multiple views, complex interactions, or you expect to iterate heavily on components, use Pattern B.

---

## 3. Choose a Port

Pick an unused port in the 3000–9000 range. Check what's running:

```bash
ss -tlnp | grep -E ':(3[0-9]{3}|[4-9][0-9]{3}) '
```

Prefer memorable ports for personal tools (e.g., 3456 for the planning app). Avoid 3000 (common dev default), 8080, 5000.

---

## 4. Create the App

Place all app files in `/home/owen/workspace/<appname>/`. Create the directory if it doesn't exist.

```bash
mkdir -p /home/owen/workspace/<appname>
```

### Design Philosophy

Before writing any code, commit to an aesthetic. Reference the frontend-design-ultimate skill for detailed guidance. Key rules:

**Typography** — never generic:
- Display/Headlines: Fraunces, Playfair Display, Cabinet Grotesk, Satoshi, Space Grotesk
- Body: DM Sans, Plus Jakarta Sans, Instrument Sans, General Sans
- Mono: DM Mono, JetBrains Mono
- Load via Google Fonts CDN in `<head>`

**Color** — commit to a palette:
- 60-30-10 rule: dominant background, secondary surfaces, sharp accent
- Use CSS custom properties (`--bg`, `--surface`, `--text`, `--accent`)
- Dark OR light — not a muddy gray middle ground
- High-contrast CTAs

**Atmosphere** — no solid backgrounds:
- Subtle gradient meshes
- Grain/noise texture (SVG filter, opacity 2-3%)
- Geometric patterns (dots, lines)

**Composition** — avoid symmetric/centered layouts:
- Asymmetry with purpose
- Generous whitespace OR controlled density (pick one)
- Diagonal flow or grid-breaking moments

**Motion** — orchestrated, not scattered:
- Staggered page load reveals with `animation-delay`
- Hover states that surprise (scale, color shift, shadow)
- 200-400ms durations (snappy)

**Mobile** — always responsive:
- Hero sections: `display: flex` on mobile (not grid)
- All grids collapse to single column at 768px
- Font sizes scale down: 3x jumps, not 1.25x increments
- Minimum touch targets: 44x44px
- `font-size >= 16px` on inputs (prevents iOS zoom)

### Pattern A: Single-File Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>App Name</title>
  <link href="https://fonts.googleapis.com/css2?family=Fraunces:wght@700&family=DM+Sans:wght@400;500&display=swap" rel="stylesheet">
  <style>
    :root {
      --bg: #0f0e0d;
      --surface: #1a1917;
      --text: #f0ece4;
      --text-secondary: #a09890;
      --accent: #e8845d;
      --border: rgba(255,255,255,0.08);
    }
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      background: var(--bg);
      color: var(--text);
      font-family: 'DM Sans', sans-serif;
      min-height: 100vh;
    }
    /* ... */
  </style>
</head>
<body>
  <div id="root"></div>
  <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <script type="text/babel">
    const { useState, useEffect, useCallback } = React;

    function App() {
      // ...
      return <div>...</div>;
    }

    const root = ReactDOM.createRoot(document.getElementById('root'));
    root.render(<App />);
  </script>
</body>
</html>
```

```js
// server.js — vanilla Node.js, no dependencies
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = <PORT>;
const DATA_FILE = path.join(__dirname, 'data', 'state.json');

// Ensure data directory exists
fs.mkdirSync(path.join(__dirname, 'data'), { recursive: true });
if (!fs.existsSync(DATA_FILE)) {
  fs.writeFileSync(DATA_FILE, JSON.stringify({}), 'utf8');
}

const server = http.createServer((req, res) => {
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  if (req.method === 'OPTIONS') { res.writeHead(204); res.end(); return; }

  if (req.method === 'GET' && req.url === '/api/state') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(fs.readFileSync(DATA_FILE, 'utf8'));
    return;
  }

  if (req.method === 'POST' && req.url === '/api/state') {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try {
        JSON.parse(body);
        fs.writeFileSync(DATA_FILE, body, 'utf8');
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ ok: true }));
      } catch {
        res.writeHead(400, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Invalid JSON' }));
      }
    });
    return;
  }

  // Static file serving
  const filePath = req.url === '/' ? '/index.html' : req.url;
  const full = path.join(__dirname, filePath);
  const types = { '.html': 'text/html', '.js': 'text/javascript', '.css': 'text/css' };
  fs.readFile(full, (err, data) => {
    if (err) { res.writeHead(404); res.end('Not found'); return; }
    res.writeHead(200, { 'Content-Type': types[path.extname(full)] || 'text/plain' });
    res.end(data);
  });
});

server.listen(PORT, '0.0.0.0', () => {
  console.log(`http://localhost:${PORT}`);
});
```

### Pattern B: Vite + React Initialization

```bash
cd /home/owen/workspace
npm create vite@latest <appname> -- --template react-ts
cd <appname>
npm install
npm install -D tailwindcss @tailwindcss/vite
npm install lucide-react framer-motion
npx shadcn@latest init --defaults
npx shadcn@latest add button card badge dialog accordion tabs
```

Configure Vite to bind to 0.0.0.0 for tailnet access:
```typescript
// vite.config.ts — add server config
server: {
  host: '0.0.0.0',
  port: <PORT>,
}
```

For Pattern B apps that need persistence, add a separate `server.js` (Express or vanilla) and run Vite in dev mode alongside it, or build and serve the dist with the API server.

---

## 5. Test Locally

Run the server and verify it works before setting up systemd.

**Pattern A:**
```bash
node /home/owen/workspace/<appname>/server.js &
curl http://localhost:<PORT>
```

**Pattern B (dev):**
```bash
cd /home/owen/workspace/<appname> && npm run dev &
```

Open the app in a browser. Iterate on the UX until it's working well before proceeding to systemd setup.

---

## 6. Set Up Systemd Service

Create a systemd user service so the app starts on boot automatically.

**Pattern A:**
```ini
# ~/.config/systemd/user/<appname>.service
[Unit]
Description=<App Name>
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/owen/workspace/<appname>
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

**Pattern B (production build):**

First build the static assets:
```bash
cd /home/owen/workspace/<appname> && npm run build
```

If it's a pure static app (no API), serve with npx serve or a tiny static file server. If it has an API server:
```ini
# ~/.config/systemd/user/<appname>.service
[Unit]
Description=<App Name>
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/owen/workspace/<appname>
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Create the service file, then enable and start:
```bash
mkdir -p ~/.config/systemd/user
# Write the service file to ~/.config/systemd/user/<appname>.service

systemctl --user daemon-reload
systemctl --user enable <appname>
systemctl --user start <appname>
systemctl --user status <appname>
```

Verify it's running:
```bash
curl http://localhost:<PORT>
```

**Enable lingering** so the service runs even when not logged in:
```bash
loginctl enable-linger $USER
```

---

## 7. Custom DNS + Tailnet Access

Apps are served at `http://<appname>.home` on the tailnet. No ports, no IP addresses.

### Infrastructure (already set up on hal — skip if done)

This is a one-time setup. If nginx, dnsmasq, and Tailscale DNS are already configured, skip to "Register a new app" below.

**Install:**
```bash
sudo apt install -y nginx dnsmasq
```

**Configure dnsmasq** — binds only to the Tailscale interface (avoids conflict with systemd-resolved on 127.0.0.53):
```bash
TAILSCALE_IP=$(tailscale ip -4)
sudo tee /etc/dnsmasq.d/hal.conf << EOF
listen-address=$TAILSCALE_IP
bind-interfaces
address=/.home/$TAILSCALE_IP
server=127.0.0.53
EOF
sudo systemctl restart dnsmasq
```

**Remove nginx default site:**
```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo systemctl reload nginx
```

Each app gets its own nginx config file — see "Register a new app" below.

**Advertise DNS via Tailscale** (web UI — one time):
1. Go to **https://login.tailscale.com/admin/dns**
2. Under **Nameservers** → **Add nameserver → Custom**
3. Enter your Tailscale IP (`tailscale ip -4`)
4. Check **"Restrict to domain"** → enter `home`
5. Save

All devices on the tailnet will now automatically resolve `*.home` — no per-device config needed.

**Fallback access:** Apps also respond at the Tailscale IP and MagicDNS hostname — useful before DNS propagates or for debugging:
```bash
tailscale ip -4   # e.g. http://<tailscale-ip>:<PORT>
```
Or via MagicDNS hostname: `http://hal:<PORT>` from any tailnet device.

**Note on browsers:** `.home` is a non-standard TLD so browsers treat bare `todo.home` as a search query. Always use the `http://` scheme: `http://todo.home`. Set bookmarks or phone home screen shortcuts to avoid typing it.

---

### Register a new app

Each app gets its own nginx config file, making it easy to add, edit, or remove independently:

```bash
sudo tee /etc/nginx/sites-available/<appname>.home << 'EOF'
server {
    listen 80;
    server_name <appname>.home;
    location / {
        proxy_pass http://localhost:<PORT>;
        proxy_set_header Host $host;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/<appname>.home /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Verify:
```bash
curl -s -H "Host: <appname>.home" http://localhost/ | head -3
```

The app is now live at `http://<appname>.home` from any device on the tailnet.

**To remove an app:**
```bash
sudo rm /etc/nginx/sites-enabled/<appname>.home
sudo nginx -t && sudo systemctl reload nginx
```

### Registered apps on hal

| URL | Port | Service |
|-----|------|---------|
| http://todo.home | 4321 | todo |
| http://planning.home | 3456 | planning |

---

## 8. Iterate on the UX

After the app is running, offer to iterate. Common improvements:

- **Layout refinements**: tighten spacing, improve visual hierarchy
- **Mobile fixes**: ensure responsiveness at 375px and 768px viewports
- **Animation polish**: add staggered reveals, hover transitions
- **Typography**: adjust scale, weight contrast, line-height
- **Color**: tweak the palette, improve contrast
- **Data model changes**: add fields, restructure storage
- **New features**: add views, modals, filters

Ask "What feels off or missing?" and implement the changes directly.

---

## Common Patterns

### Persistent State (Pattern A)
Load state on mount, save on change:
```jsx
const [state, setState] = useState({});

// Load
useEffect(() => {
  fetch('/api/state').then(r => r.json()).then(setState);
}, []);

// Save
useEffect(() => {
  if (Object.keys(state).length === 0) return;
  fetch('/api/state', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(state),
  });
}, [state]);
```

### Color System
Always use CSS custom properties, never hardcode:
```css
:root {
  --bg: #0f0e0d;
  --surface: #1c1917;
  --surface-raised: #252220;
  --text: #f0ece4;
  --text-dim: #8a8078;
  --accent: #e8845d;
  --accent-hover: #f09070;
  --border: rgba(255,255,255,0.07);
  --shadow: 0 4px 24px rgba(0,0,0,0.4);
}
```

### Mobile-First Hero
```css
.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 3rem 1.5rem;
}

@media (min-width: 768px) {
  .hero {
    display: grid;
    grid-template-columns: 1fr 1fr;
    text-align: left;
    padding: 6rem 4rem;
  }
}
```

### Staggered Reveal
```css
@keyframes fadeUp {
  from { opacity: 0; transform: translateY(16px); }
  to { opacity: 1; transform: translateY(0); }
}

.hero-badge   { animation: fadeUp 0.5s ease-out 0.1s both; }
.hero-title   { animation: fadeUp 0.5s ease-out 0.25s both; }
.hero-sub     { animation: fadeUp 0.5s ease-out 0.4s both; }
.hero-actions { animation: fadeUp 0.5s ease-out 0.55s both; }
```

---

## Service Management

```bash
# Status
systemctl --user status <appname>

# Restart after code changes
systemctl --user restart <appname>

# View logs
journalctl --user -u <appname> -f

# Stop and disable
systemctl --user stop <appname>
systemctl --user disable <appname>
```

---

## Pre-Ship Checklist

- [ ] App works at `localhost:<PORT>`
- [ ] Data persists across page refreshes
- [ ] Mobile layout works at 375px width
- [ ] Typography is distinctive (not Inter/Arial)
- [ ] Background has atmosphere (not solid)
- [ ] Touch targets are at least 44x44px
- [ ] Systemd service is enabled and running
- [ ] `loginctl enable-linger` done for persistence after logout
- [ ] nginx server block added and reloaded
- [ ] App reachable at `http://<appname>.home` from another tailnet device
- [ ] Registered app in the "Registered apps on hal" table in this skill

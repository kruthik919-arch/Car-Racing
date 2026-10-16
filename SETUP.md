# M Circuit – Multiplayer + Sound setup guide (for beginners)

## What's in this zip

```
index.html            The updated game (use this one inside your Android WebView app)
server/
  server.js           The multiplayer server (accounts, friends, rooms, live races)
  package.json        Tells hosts how to start it (no packages to install!)
  public/
    index.html        Same game file, served by the server (keep the two copies identical)
    (put your models/, intro.mp4, three.min.js, GLTFLoader.js here too)
SETUP.md              This file
```

The server has **no dependencies**: it uses only what's built into Node.js, so there's no `npm install` step to go wrong.

---

## Part 1 – Try everything on your own computer (15 minutes, free)

1. **Install Node.js** (the "LTS" version, 18 or newer) from https://nodejs.org. Accept all the defaults.
2. **Copy your game assets** into `server/public/`: the `models` folder, `intro.mp4`, and, if you use local copies, `three.min.js` and `GLTFLoader.js`. Keep them next to `index.html` exactly as you have them today.
3. **Open a terminal** (Windows: search "Command Prompt"; Mac: "Terminal") and go into the server folder:
   ```
   cd path/to/server
   node server.js
   ```
   You should see: `M Circuit server running on port 3000`. Leave this window open.
4. **Open the game**: visit http://localhost:3000 in Chrome.
5. **Test multiplayer with yourself**: open a second *Incognito/Private* window (or a different browser) at the same address. Register two different accounts, one in each window.
   - In window 1: Friends → **+ Add Friend** → type the other username → Send.
   - In window 2: Friends → **Requests** → Accept.
   - In window 1: Friends → **Invite** next to your friend. Window 2 gets an invite banner → **Join**.
   - In window 1 (the host, marked ★): **Start race**. Both windows race live and see each other's cars.
   - Arrow keys work on a computer: Up = gas, Down = brake, Left/Right = steer.
6. **Test from your phone** (same Wi-Fi): find your computer's local IP (Windows: `ipconfig`, Mac: System Settings → Wi-Fi → Details), then open `http://192.168.x.x:3000` on the phone.

Your accounts are saved in `server/data.json`. Delete that file to wipe all accounts.

---

## Part 2 – Put the server on the internet

Players need a server that is always on and reachable over **HTTPS** (Android blocks plain `http://` / `ws://` to public servers by default). Two good options:

> **Important about free hosting:** free tiers (for example Render's free plan) sleep after about 15 minutes with no traffic and **erase local files whenever they restart**. That means all accounts and friend lists would vanish. Free is fine for testing, but for real players use a plan with a **persistent disk** or a small VPS.

### Option A – Render (easiest, no server admin)

1. Create a free account at https://github.com and make a **new repository**. Upload the whole `server` folder's contents so that `server.js` and `package.json` are at the top level of the repo (drag and drop in the browser works).
2. Create an account at https://render.com → **New → Web Service** → connect your GitHub repo.
3. Fill in:
   - **Runtime:** Node
   - **Build command:** leave empty (or `echo ok`)
   - **Start command:** `node server.js`
   - **Instance type:** *Free* to just test; a **paid** type if you want accounts to survive restarts.
4. For a paid instance, open **Disks → Add disk**, mount path `/var/data`, then **Environment → Add variable**:
   `DATA_FILE` = `/var/data/data.json`
5. Click **Create Web Service**. After a few minutes you get an address like `https://m-circuit.onrender.com`. Open it in a browser: the game should load and you can register.
6. Your WebSocket address is the same host with `wss://` and `/ws`: **`wss://m-circuit.onrender.com/ws`**

### Option B – a small VPS (about $4–6/month, accounts persist, most control)

1. Rent a small Ubuntu server (DigitalOcean, Hetzner, Vultr, …) and point a domain name at it.
2. Install Node and a web server that gets free HTTPS certificates automatically (Caddy):
   ```
   sudo apt update && sudo apt install -y nodejs caddy
   ```
   (Make sure `node -v` prints 18 or higher; otherwise install Node from nodesource.com.)
3. Copy the `server` folder to `/opt/mcircuit` (for example with `scp`), then keep it running with systemd. Create `/etc/systemd/system/mcircuit.service`:
   ```
   [Service]
   WorkingDirectory=/opt/mcircuit
   ExecStart=/usr/bin/node server.js
   Environment=PORT=3000
   Environment=DATA_FILE=/opt/mcircuit/data.json
   Restart=always
   [Install]
   WantedBy=multi-user.target
   ```
   Then: `sudo systemctl enable --now mcircuit`
4. Put this in `/etc/caddy/Caddyfile` (replace the domain), then `sudo systemctl reload caddy`:
   ```
   yourdomain.com {
       reverse_proxy localhost:3000
   }
   ```
   Caddy handles HTTPS and WebSockets for you.
5. Your WebSocket address is **`wss://yourdomain.com/ws`**.

**Back up `data.json`** now and then. It contains every account.

---

## Part 3 – Connect your Android WebView app

You have two ways; the first is simplest.

**Way 1 (recommended): load the game from your server.**
```kotlin
webView.loadUrl("https://m-circuit.onrender.com/")
```
The game finds the server by itself, and you can update the game by redeploying the server without publishing a new app version.

**Way 2: keep `index.html` inside the app** (assets). Then open `index.html`, find this line near the top of the script, and fill it in:
```js
const SERVER_URL='wss://m-circuit.onrender.com/ws';
```

Either way, make sure your WebView is set up like this:
```kotlin
webView.settings.javaScriptEnabled = true
webView.settings.domStorageEnabled = true            // REQUIRED: keeps the login session
webView.settings.mediaPlaybackRequiresUserGesture = false   // lets the intro video / audio start
```
and in `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.INTERNET" />
```
In your Activity, call `webView.onPause()` / `webView.onResume()` from `onPause()` / `onResume()` so music stops when the player leaves the app.

---

## What was added to the game

**Accounts** – real, server-side. Passwords are hashed (scrypt) on the server, never stored in plain text. The app remembers you with a secret login token, so you skip the login screen next time. A **Play offline** link on the login screen lets someone play solo without an account (no friends/multiplayer). Old local-only accounts from the previous version are not migrated.

**Friends** – send, accept or decline requests, see who's online, invite a friend to your room. Search finds real players.

**Multiplayer** – Quick match (auto-starts 10 seconds after 2+ drivers join, or when full), Create room, Join from the open rooms list, host-only Start. Up to 4 drivers. During a race everyone sees each other's cars with name tags, a live ranking list and mini-map dots. Cars don't collide ("ghost" racing), which keeps it smooth on mobile connections. Other players' cars use a light built-in body so phones don't run out of memory; your own car keeps its full model.

**Fair results** – the server measures finish times itself, computes the score, and ignores impossible jumps in position, so a modified client can't simply claim a win.

**Sound** (all generated in code, no files needed) – engine pitch rises with speed and drops on gear changes, tire squeal when cornering/braking hard/launching, wind noise, menu click sounds, 3-2-1-GO beeps, a finish jingle, and looping background music (quieter during races). The **Sound effects** and **Music** sliders in Settings control the volume; 0 is silent, and moving the sound slider plays a preview beep.

**One small fix** – the countdown showed 4-3-2-1 although it was meant to be 3-2-1; it's now 3-2-1-GO.

## Honest limits

- I tested the server and the game logic with simulated players (registration, friends, invites, rooms, a full two-player race with finish times, results, logout/login) and the sound engine against a simulated audio device. I couldn't run it on a real phone, so **listen to the sound and tweak by ear**: the numbers are in the `SND` block of `index.html` (e.g. `34+rpm*130` is the engine pitch, `.5` scales music loudness).
- One server process handles roughly hundreds of simultaneous players, which is plenty for launch. Beyond a few thousand accounts, move `data.json` to a real database.
- There's no password reset (no email). A forgotten password means deleting that user from `data.json`.
- Mobile networks add delay: other cars are smoothed and predicted a fraction of a second ahead, so they can look slightly off during sudden moves.

## Troubleshooting

| Problem | Fix |
|---|---|
| Login screen says "Can't reach the server" | Is `node server.js` running? In the app, is `SERVER_URL` correct and starting with `wss://`? |
| Works in browser, not in the app | `domStorageEnabled` must be true; use `https`/`wss`, not `http`/`ws` |
| Accounts disappear after a while | Your host erased local files. Use a persistent disk (Render paid) or a VPS, and set `DATA_FILE` |
| First connection takes ~30 s on Render free | The free service was asleep; it wakes up on the first request |
| No sound until I tap | Normal: browsers and WebView only allow audio after a touch; the first tap enables it |
| Friend shows offline but is online | They may be on a different server instance; run only one |

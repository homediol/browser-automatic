# Orbit Browser

Orbit Browser is a multi-user remote Google Chrome platform. Users register, create isolated headed or headless Chrome sessions, control them from a React dashboard, capture screenshots, and run reusable Playwright workflows.

This workspace started without application files or Git history. The administrator dashboard and administrator-only features are intentionally omitted at the user's direction.

## Architecture

```mermaid
flowchart LR
  UI[React dashboard + noVNC] -->|REST / authenticated Socket.IO| API[Express API]
  API --> DB[(PostgreSQL)]
  API --> REDIS[(Redis / BullMQ)]
  API -->|local process or optional Docker| W1[Per-session Chrome worker]
  API --> DOCKER[Docker Engine when enabled]
  DOCKER --> W2[Per-session Chrome container]
  W2 -->|Docker mode outbound HTTP(S)| PROXY[SSRF-filtering Squid proxy]
  W1 -->|private RFB| VNC[x11vnc on isolated Xvfb]
  API -->|owner-checked relay| VNC
  API --> SHOTS[(Private screenshot volume)]
  EXT[Browser Bot Connector extension] <-->|Native Messaging| AGENT[Local Node agent]
  AGENT <-->|Short-lived token / Socket.IO| API
  EXT -->|tabCapture / WebRTC| UI
```

Each browser session runs in its own worker process with a separate persistent profile directory. The optional Docker runtime places each worker in its own container and named volume. Workers launch installed Google Chrome Stable through Playwright's `channel: 'chrome'`. A headed session gets a dedicated Xvfb display and x11vnc server; noVNC renders and controls that actual desktop in the dashboard through a short-lived, owner-checked WebSocket relay. VNC listeners stay on loopback in npm mode and on the private worker network in container mode. Headed screenshots come from that X display; headless sessions use real Chrome headless mode and page screenshot previews.

## Requirements

- Linux x86-64 host with Node.js 22+, npm, PostgreSQL 17, Redis 7, and Google Chrome Stable.
- Xvfb, x11vnc, and FFmpeg for headed Chrome desktops and screenshots. The noVNC client is installed as an npm dependency.
- Docker Engine and Docker Compose v2 only if using the optional container runtime below.
- Enough RAM and CPU for the configured concurrent Chrome workers. Defaults reserve up to 1.5 GiB and 1.5 CPU cores per worker, with 20 platform-wide active sessions.
- OpenSSL to create local secrets.

In Docker mode Chrome runs as a non-root account inside each browser container. The Chrome sandbox remains enabled; the worker does not pass `--no-sandbox`. Browser worker containers do not publish ports. Their HTTP control API and Chrome debugging endpoint are reachable only on the private Docker bridge and worker requests require a random session token.

## Run using npm

The default `BROWSER_RUNTIME=local` runs each browser worker as a separate local Node process. It uses the installed Google Chrome Stable executable and stores each session profile in its own directory. PostgreSQL and Redis must already be running as local services; npm starts the API and dashboard.

On Debian or Ubuntu, install the services if needed:

```sh
sudo apt update
sudo apt install postgresql postgresql-contrib redis-server xvfb x11vnc ffmpeg
sudo systemctl enable --now postgresql redis-server
```

Create a database account and database (enter the password you plan to use in `.env` when prompted):

```sh
sudo -u postgres createuser --pwprompt browser
sudo -u postgres createdb --owner=browser browser_platform
```

Then configure and start the app:

```sh
cp .env.example .env
openssl rand -hex 32
```

Replace the placeholders in `.env`. Use separate generated values for `JWT_ACCESS_SECRET`, `WORKER_TOKEN_ENCRYPTION_KEY`, and `TURN_SHARED_SECRET`. Set `POSTGRES_PASSWORD` and the password in `DATABASE_URL` to the password you chose for the `browser` database account. Keep `BROWSER_RUNTIME=local`; leave `TURN_URL` empty for local-only use. To monitor rules signals, set `RULES_API_URL`. The backend uses the API's authenticated WebSocket stream when `RULES_PROJECT_TOKEN` is configured; if the stream is unavailable, it falls back to polling the public action endpoint every second. Keep the project token in the backend `.env` only. To use the dashboard from another machine, configure a reachable TURN server and set `TURN_URL` accordingly.

Install the JavaScript dependencies and launch both the API and dashboard:

```sh
npm install
npm run dev
```

The development launcher starts the API first and waits for its database and Redis health check before starting Vite, so the dashboard does not proxy requests before the API is ready. Open `http://localhost:5173`, register an account, and launch a Chrome session. The API is available at `http://localhost:4000`; migrations run automatically on startup. Use `Ctrl+C` to stop the dashboard, API, and active browser workers.

Local npm mode enforces per-user and global session-count limits and keeps profiles separate. It does not provide Docker's hard CPU, memory, PID, or network isolation, so use it for trusted local development. The optional Docker runtime provides those container limits and the Squid egress proxy.

## Connect an existing Chrome browser

Connected Chrome is an independent browser engine. It does not replace or change Playwright sessions. The extension and Native Messaging agent run on the computer that owns Chrome; the dashboard and backend may run elsewhere.

1. Install this repository's npm dependencies with `npm install` and run the Orbit server as above. For remote browsers, use an HTTPS dashboard/API origin and configure reachable STUN/TURN servers in the server environment.
2. In Chrome, open `chrome://extensions`, enable Developer mode, and choose **Load unpacked**. Select the `browser-extension` directory. Copy its extension ID.
3. On Linux, run `bash local-agent/installers/linux/install.sh EXTENSION_ID https://YOUR-ORBIT-SERVER`. On Windows PowerShell, run `local-agent/installers/windows/install.ps1 -ExtensionId EXTENSION_ID -ApiUrl https://YOUR-ORBIT-SERVER`. Use `http://localhost:4000` for local development.
4. Fully restart Chrome. In Orbit, open **Connected Browsers** and generate a pairing code. Enter it in the Browser Bot Connector popup.
5. To run page automation, select a tab and grant **Allow this website** for that origin in the extension popup. To see the real tab, activate the page you want to share, open the extension popup from that tab, and click **Share active tab**; Orbit starts the live view automatically. For real mouse and keyboard input, reload the updated extension and accept Chrome's permission notice.

Pairing codes expire after five minutes and are single-use. A local agent stores a rotating credential in a user-only file and makes no local TCP listener. Live capture is explicit, browser-indicated, and stops when the user stops sharing or disconnects. Extensions cannot control Chrome's address bar or browser settings; dashboard navigation opens URLs in the selected tab. Tab streaming uses WebRTC and needs reachable ICE servers for browsers outside the server's LAN. See [local-agent/README.md](local-agent/README.md) for install and uninstall details.

The opt-in extension integration harness uses headed Playwright Chromium to load the unpacked MV3 extension. Current Google Chrome Stable releases no longer support the command-line flags Playwright uses to sideload unpacked extensions, so this harness does not verify installation in Stable. The Linux local-agent installer registers the Native Messaging host for Chrome Stable, Chrome for Testing, and Chromium; `bash local-agent/installers/linux/check.sh EXTENSION_ID` checks the host name, extension origin, manifest path, and executable launcher.

```sh
CONNECTED_CHROME_NATIVE_HOST_REGISTRATION_DIR="$HOME/.config/chromium/NativeMessagingHosts" RUN_CONNECTED_CHROME_EXTENSION=true TEST_DATABASE_URL="$DATABASE_URL" npm run test:connected-extension
```

The previous opt-in extension run did not complete Native Messaging pairing (`Specified native messaging host not found`). The installer and diagnostics now cover the missing-registration case, but the full connected-browser extension flow still needs a successful rerun; this Chromium harness does not substitute for a manual Chrome Stable install and interaction check.

## Optional Docker deployment

Copy the example environment file and replace every secret placeholder with a fresh value:

```sh
cp .env.example .env
openssl rand -hex 32
```

Set the generated values in `.env`:

- `JWT_ACCESS_SECRET`: random access-token signing secret, at least 32 characters.
- `WORKER_TOKEN_ENCRYPTION_KEY`: exactly 64 hexadecimal characters (32 random bytes).
- `TURN_SHARED_SECRET`: random coturn REST authentication secret.
- `POSTGRES_PASSWORD`: strong database password.
- `TURN_PUBLIC_IP`: the address remote browsers can reach.
- `TURN_URL`: `turn:<public-ip>:3478?transport=udp` for dashboard clients.
- `TURN_INTERNAL_URL`: leave as `turn:coturn:3478?transport=udp` for browser workers.
- `APP_ORIGIN`: browser dashboard origin. Production requires an exact match; local development also allows HTTP `localhost` and loopback origins on any port so Vite can use its next available port.
- `PUBLIC_API_ORIGIN`: optional public API origin used in Data Watch share links; defaults to `APP_ORIGIN`. Set this when consumers reach the API on a separate origin.
- `VITE_API_BASE_URL`: API path or URL baked into the frontend image. The default `/api` uses the same-origin reverse proxy; set an absolute URL only when the API is hosted separately.
- `BROWSER_RUNTIME`: set to `docker` for the container runtime.
- `RULES_API_URL`: optional server-side base URL for the rules signal integration.
- `RULES_PROJECT_TOKEN`: optional consumer token used only to obtain a one-use WebSocket ticket; it is never sent to the public latest-action endpoint or frontend.
- `RULES_SIGNAL_POLL_INTERVAL_MS`: polling fallback interval from 1000 to 60000 milliseconds; defaults to 1000.
- `RULES_SIGNAL_WS_RETRY_MS`: how long to poll before retrying the WebSocket; defaults to 30000 milliseconds.

For a local single-host setup, the example points TURN at `127.0.0.1`. For a remote deployment, replace it with the host's public address. Allow inbound UDP and TCP 3478 and UDP 49160–49200 in the host firewall and cloud security group. coturn advertises the public/private address mapping for its Docker network address `172.30.0.5`; change the Compose subnet and mapping together if that subnet conflicts with the host network.

Build the Chrome worker image separately. It installs the current Google Chrome Stable package, Xvfb, x11vnc, FFmpeg, Playwright, and the WebRTC native module:

```sh
docker compose --profile build-only build browser-worker-image
docker compose build backend frontend
docker compose up -d postgres redis browser-egress coturn backend frontend
```

The backend waits for PostgreSQL and Redis, applies SQL files from `database/migrations`, then starts the API and BullMQ workers. The UI is available at `http://localhost:5173`; the API health endpoint is `http://localhost:4000/api/health`.

Useful commands:

```sh
docker compose ps
docker compose logs -f backend
docker compose logs -f coturn browser-egress
docker compose down
```

`docker compose down` keeps PostgreSQL, Redis, screenshot, and browser-profile volumes. `docker compose down -v` deletes persistent data, including user accounts and profiles.

## User features

- Email and password registration/login. Passwords are bcrypt-hashed; short-lived access tokens are JWTs, and refresh tokens are random opaque values stored only as hashes and rotated on use. Reuse revokes the token family.
- Browser session create, stop, restart, delete, URL navigation, and status. Each user has configurable active and saved-session limits.
- Headed desktop viewing and mouse, keyboard, scroll, double-click, right-click, and drag control through noVNC and the actual session X display. Navigation, reload, stop-loading, and screenshots use the authenticated browser controls.
- Headed screenshots capture the complete Chrome desktop at the configured Xvfb resolution. Headless previews remain independent and capture the Chrome page.
- Headless screenshot preview, explicit screenshot capture, and automatic screenshot history every five minutes (configurable). History retains up to 100 screenshots per session.
- Persistent Data Watch runs CSS selectors in the selected page or iframe on the backend, stores changed readings, resumes after backend restarts, and can share read-only data through revocable links, paginated history, and a live SSE stream.
- Playwright workflow editor and BullMQ execution for navigation, DOM selectors, typing, form fill, select, wait, scroll, screenshots, and JavaScript that the user explicitly approves for each run. Selector steps can target one iframe by CSS selector. Connected Chrome discovers HTTP(S) child frames on the selected page and lets the user choose the page or any child frame. Its popup lists detected iframe origins and provides a separate permission button for each origin. Inspection and clicks bind to that tab ID, frame ID, and frame origin/path. Find elements and Start remain disabled until Chrome grants access to the selected origin.
- Latest Rules API signal page with the action and index, automatic five-second refresh, and manual refresh.
- User profile, browser defaults, theme, activity history, session/resource usage, and a free-plan usage summary.

There is no billing provider integration; the subscription row represents the local free plan. There is no administrator dashboard or administrator API.

## Security boundaries

- REST and Socket.IO access tokens are checked before actions. Browser-session, screenshot, workflow, execution, profile, and settings queries are scoped to the authenticated owner. Composite database foreign keys prevent attaching a session or screenshot to another user's profile.
- Worker bearer tokens are random per session and AES-GCM encrypted in PostgreSQL. Refresh tokens are hashed. Screenshot files are written with owner-only permissions and returned only through owner-checked API routes.
- Local x11vnc servers listen only on `127.0.0.1`; container VNC servers have no published host port and are reachable only on the worker network. The dashboard obtains a 20-second, one-use Redis ticket after session ownership validation; the WebSocket consumes it, rechecks the owner/account/session state, and relays only to that session's VNC port. VNC passwords are random per session and AES-GCM encrypted at rest. The reverse proxy suppresses VNC WebSocket access logs so tickets are not written to request logs.
- Input URLs must be HTTP(S), standard ports, non-local, and resolve only to public addresses. Every browser request and redirect is checked. In Docker mode Chrome is forced through Squid, which also denies private, loopback, link-local, metadata, reserved, and non-web destinations after resolving the connection address. Local npm mode does not include that network-level proxy defense and is intended for trusted development.
- In Docker mode, per-worker memory, CPU, PID and shared-memory limits are applied; workers have all Linux capabilities dropped, no-new-privileges enabled, no published ports, and session-specific profile volumes.
- The optional Docker runtime requires API access to `/var/run/docker.sock` to create and remove session containers. Treat it as a privileged control-plane service: keep it private, restrict deployment access, and place it on a dedicated host. The Docker socket is not mounted into browser workers or the frontend.
- In production set `NODE_ENV=production`, use HTTPS for both app and API, set the exact public `APP_ORIGIN`, and put the API behind a TLS reverse proxy. Same-site app/API domains are recommended so the secure `SameSite=Strict` refresh cookie works.

URL checks and the outbound proxy are defense in depth, not an authorization grant. Users must only automate sites and activities they are authorized to control and must respect site terms, CAPTCHA challenges, access restrictions, and anti-bot protections. The platform has no stealth fingerprint spoofing.

## Resource settings

Configure capacity in `.env`:

- `MAX_SESSIONS_PER_USER`, `MAX_SAVED_SESSIONS_PER_USER`, and `MAX_CONCURRENT_SESSIONS`.
- `BROWSER_SESSION_MEMORY_MB`, `BROWSER_SESSION_CPU_CORES`, and `BROWSER_SESSION_PIDS_LIMIT`.
- `SESSION_IDLE_MINUTES`, `SCREENSHOT_INTERVAL_SECONDS`, `BROWSER_WIDTH`, and `BROWSER_HEIGHT`.

The Redis maintenance worker records process-tree CPU/memory samples in local mode or Docker CPU/memory samples in container mode, takes automatic snapshots, stops idle sessions, recovers crashed workers, and prunes old usage samples and screenshots. Local profile directories preserve session cookies and storage after stopping; container mode uses named volumes. Set an appropriate Docker volume/storage quota at the host or volume-driver level for container deployments; the local Docker volume driver does not enforce a per-volume byte quota.

## API overview

All routes are under `/api`.

| Area | Routes |
|---|---|
| Auth | `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me` |
| Sessions | `GET/POST /sessions`, `GET /sessions/:id`, `GET /sessions/:id/state`, `POST /sessions/:id/control`, `POST /sessions/:id/stop`, `POST /sessions/:id/restart`, `DELETE /sessions/:id` |
| Live headed desktop | `POST /sessions/:id/vnc/ticket`, authenticated WebSocket `/api/vnc/:id` |
| Screenshots | `POST/GET /sessions/:id/screenshots`, `GET /sessions/:id/preview`, `GET /screenshots`, `GET /screenshots/:id/image` |
| Data watches | `GET/POST /data-watches`, `POST /data-watches/:id/pause`, `POST /data-watches/:id/resume`, `POST /data-watches/:id/stop`, `POST/DELETE /data-watches/:id/share`, `DELETE /data-watches/:id` |
| Public Data Watch | `GET /public/data-watches/:token`, `/latest`, `/history`, `/stream` (share link; no API key or Authorization header) |
| Automation | `GET/POST /workflows`, `PATCH/DELETE /workflows/:id`, `POST /workflows/:id/execute`, `GET /workflows/executions/:executionId` |
| Rules signals | `GET /signals/latest` (authenticated; requires `RULES_API_URL`) |
| Signal automation | `GET /automatic-interactions/monitor`, `GET /automatic-interactions/usage`, `GET/POST /automatic-interactions/signal-rules`, `PATCH/DELETE /automatic-interactions/signal-rules/:id`, `GET/POST /automatic-interactions/signal-click-rules`, `PATCH/DELETE /automatic-interactions/signal-click-rules/:id`, `GET /automatic-interactions/signal-executions/:id`, `POST /automatic-interactions/signal-executions/:id/confirm`, `POST /automatic-interactions/signal-executions/:id/cancel` |
| Account | `GET /account/overview`, `GET /account/usage`, `GET /account/activity`, `PATCH /account/profile`, `GET/PATCH /settings` |

The rules signal endpoint proxies `GET /api/v1/signals/latest/action` configured by `RULES_API_URL` and returns the direct `{ "action": "OK", "index": "#21091" }` response, or `null` when there is no signal yet. Signal monitoring prefers the API's ticket-authenticated WebSocket stream and falls back to configurable one-second polling. It records each new index once. Workflow destinations match their configured action/index filters; click destinations automatically inspect their saved selector and perform one click for each new `OK` index, without extra signal filters or a click interval. Each browser target allows only one active interaction at a time, and sensitive controls still require confirmation. Signal triggers, runs, and successful clicks are persisted and counted. In frontend code, call `fetchLatestSignal()` from `frontend/src/services/api.js` for a one-time read of the latest action.

```js
import { fetchLatestSignal } from '../services/api.js';

const signal = await fetchLatestSignal();
if (!signal) console.log('NO_SIGNALS_YET');
else console.log(signal.action, signal.index);
```

### Share Data Watch readings with another app

In **Data Watch**, select a saved watch and choose **Create public link**. The API returns a link with a random 256-bit capability token in its path. Consumers do not need an API key, login, or `Authorization` header. Anyone who has the URL can read that watch, so share it only with intended consumers. The database stores a SHA-256 hash of the token; the original link is shown only when created. **Replace public link** rotates the token and immediately invalidates the previous URL. **Revoke link** disables public reads and closes connected streams. The signed-in owner can also create/rotate a link with `POST /api/data-watches/:id/share` and revoke it with `DELETE /api/data-watches/:id/share`.

Public links are read-only and expose the selector, status, read timestamps, match count, and up to 100 matching text values. `count` is the total number of CSS matches. History stores each changed snapshot, not every poll of unchanged page content; a new payout list is emitted when its values change. The `id`/cursor is a decimal string so PostgreSQL's 64-bit sequence remains precise in JavaScript.

| Consumer endpoint | Response |
|---|---|
| `GET {link}` or `GET {link}/latest` | Watch metadata, latest reading, and current cursor |
| `GET {link}/history?after=0&limit=100` | Readings after the cursor in ascending order, `nextCursor`, and `hasMore`; `limit` is 1–500 |
| `GET {link}/stream` | Server-Sent Events: one initial `snapshot`, then each new `reading`; reconnect resumes with `Last-Event-ID` |

Example `curl` calls need no credentials:

```sh
DATA_LINK='https://your-api.example/api/public/data-watches/PASTE_SHARE_TOKEN'
curl "$DATA_LINK/latest"
curl "$DATA_LINK/history?after=0&limit=100"
curl -N "$DATA_LINK/stream"
```

Fetch latest and page through the full history from JavaScript:

```js
const shareLink = 'https://your-api.example/api/public/data-watches/PASTE_SHARE_TOKEN';

const latestResponse = await fetch(`${shareLink}/latest`);
if (!latestResponse.ok) throw new Error(`Data Watch API returned ${latestResponse.status}`);
const { latest } = await latestResponse.json();
console.log('Latest reading:', latest);

let cursor = '0';
let hasMore = true;
while (hasMore) {
  const response = await fetch(`${shareLink}/history?after=${cursor}&limit=100`);
  if (!response.ok) throw new Error(`Data Watch API returned ${response.status}`);
  const page = await response.json();
  for (const reading of page.readings) {
    console.log(reading.checkedAt, reading.items.map(item => item.value));
  }
  cursor = page.nextCursor;
  hasMore = page.hasMore;
}
```

In a browser app, use `EventSource` to receive readings as they arrive:

```js
const events = new EventSource(`${shareLink}/stream`);
events.addEventListener('snapshot', event => {
  console.log('Current value:', JSON.parse(event.data).latest);
});
events.addEventListener('reading', event => {
  const reading = JSON.parse(event.data);
  console.log('New reading:', reading.checkedAt, reading.items.map(item => item.value));
});
events.addEventListener('revoked', () => events.close());
```

Public read endpoints allow browser CORS requests from any origin and do not require or read cookies. Set `PUBLIC_API_ORIGIN` to the public API origin if it differs from `APP_ORIGIN`; configure your reverse proxy to pass `/api/public/data-watches/*` through without buffering the SSE response.

## Tests and verification

Run the fast unit tests and syntax checks:

```sh
npm test
npm run check
npm run build
```

The cross-user integration test is opt-in because it creates real Chrome workers and needs PostgreSQL, Redis, Xvfb, x11vnc, FFmpeg, and installed Google Chrome Stable. Start the services, then run:

```sh
RUN_PLATFORM_INTEGRATION=true PLATFORM_BASE_URL=http://localhost:4000 npm test
```

It registers two users; checks login and refresh-token rotation; launches independent headed sessions and a headless session; checks cross-user access denial; verifies VNC tickets and the live RFB handshake; captures owner-only display and headless screenshots; exercises Socket.IO input and a workflow; and stops, restarts, and removes sessions. For cleanup of the temporary test accounts, set `TEST_DATABASE_URL` to a host-reachable URL for the same test database.

Do not expose the test database or run the integration test against real user data. The suite uses installed Google Chrome Stable and does not substitute a mock browser.
# browser-automatic

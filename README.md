<div align="center">

# 🧅 OffCode Tor-Chat E2EE

**Anonymous - End-to-End Encrypted - Self-Hosted Chat over Tor Hidden Services**

[![Platform](https://img.shields.io/badge/platform-Windows-blue?logo=windows)](https://dotnet.microsoft.com/en-us/download/dotnet/9.0)
[![.NET](https://img.shields.io/badge/.NET-9.0-purple?logo=dotnet)](https://dotnet.microsoft.com/en-us/download/dotnet/9.0)
[![Tor](https://img.shields.io/badge/Tor-Hidden%20Service-.onion-7D4698?logo=torproject)](https://www.torproject.org/)
[![Encryption](https://img.shields.io/badge/encryption-AES--256--GCM-green)](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto)
[![Source](https://img.shields.io/badge/source%20code-$49-orange)](https://www.dev-offcode.com/TorChatPage.html)

*Self-hosted anonymous chat over Tor. Password-protected rooms are server-blind E2EE; open rooms are public encrypted channels (key = room id).*

</div>

---

## What is TorChat?

TorChat is a **self-hosted anonymous chat server** that runs entirely over the Tor network as a hidden service (`.onion` address). It combines independent layers of protection:

1. **Tor Network** — hides both the server and users’ real IP addresses
2. **Optional Full Vanguards** — hardens long-lived hidden services against guard-discovery / traffic-analysis attacks
3. **End-to-End Encryption** — messages are encrypted in the browser (AES-256-GCM) before send; the server stores ciphertext only
4. **Encrypted Storage** — data at rest is protected by SQLCipher (AES-256)

**Room types matter:** open rooms encrypt in the browser but derive the key from the room id (not server-blind). Password-protected rooms keep the key secret from the host. Hidden rooms stay off the public list and use invite links; add a password for server-blind E2EE.

Anyone who knows the `.onion` address (and invite / password when required) can connect with Tor Browser. No accounts, no phone numbers.

---

## Key Features

| Feature | Details |
|---------|---------|
| Tor Hidden Service | Automatic `.onion` address generation and management |
| Server-blind E2EE | Password rooms: AES-256-GCM; host cannot derive the room key |
| Open rooms | Public channels — encrypted in browser; key = f(room id) |
| ECDH Hybrid (1:1) | Locked 1:1: ephemeral P-256 + HKDF; safety number (3×5 decimal) + TOFU verify |
| Sender binding | Unique `senderId` per room; server overwrites packet identity |
| Encrypted Database | SQLCipher with DPAPI-protected key (AES-256 at rest) |
| Real-time Chat | ASP.NET Core SignalR WebSockets |
| Room System | Open, password-protected, and/or hidden (+ invite) |
| Hidden Rooms | Off public list; join via invite (password still required if locked) |
| Close room | Creator can block all new joins (invite included); existing members stay |
| Lock later | Open rooms can be locked once with a password (server-blind E2EE); wipes open history |
| Delete room | Creator can wipe the room + ciphertext from the server (secure_delete + VACUUM) |
| Slow mode | Optional per-room send cooldown (1 min … 4 h); applies to everyone including the creator |
| Creator recovery | Browser code binds rooms you create → Settings (and **My rooms** for hidden); Copy / Restore |
| Message display prefs | Hide date and/or time stamps; pick message time zone (Tor often shows UTC) |
| Message TTL | Required auto-expiry (30 min … 1 week; default 8 h). Forever retention is not offered |
| E2EE attachments | Images + small files via the same `SendEncrypted` path; Tor-safe metadata strip (canvas only when needed) |
| Secure erasure | `secure_delete` + WAL checkpoint/VACUUM on expiry & room wipe; Manager overwrites DB/key (optional HS keys) |
| Replay Protection | Fingerprint includes client timestamp; freshness window + restart-gap reject |
| CORS lock | `TORCHAT_ALLOWED_ONION` + live `POST /api/admin/cors-onion` (Manager; no ChatServer restart) |
| Clearnet warning | Client banner if page host is not `.onion` / localhost (does not block chat) |
| SRI verify | `tools/verify-sri.ps1` checks `index.html` hashes against wwwroot JS |
| GUI Manager | Windows Forms Server Manager — Start ChatServer first, then Tor; Copper/Forge UI |
| Docker host | Optional compose stack: ChatServer + Tor HS (no clearnet host ports) |
| No Accounts | Zero registration — join with a display name only |
| Privacy-focused storage | No IP logging; no passwords or plaintext; display names and ciphertext envelopes stored in SQLCipher until expiry or deletion |

---

## How It Works

```
+----------------------------------------------------------+
|                   Tor Browser (User)                     |
|                                                          |
|  Password -> PBKDF2 -> roomKey (AES-256) --+            |
|                   +-> joinProof -----------+--> SignalR  |
|  Locked 1:1: ECDH -> hybridKey -----------+             |
|  Message plaintext -> AES-GCM encrypt ----+             |
|                         | ciphertext only               |
+----------------------------------------------------------+
                          | Tor Network (.onion)
+--------------------------v--------------------------------+
|                   ChatServer (Host)                      |
|                                                          |
|  Receives: ciphertext blob - NEVER decrypts it           |
|  Stores:   encrypted payload in SQLCipher DB             |
|  Broadcasts: same ciphertext to room members             |
|                                                          |
|  +---------------------------------------------+        |
|  | Server Manager (WinForms)                   |        |
|  | - Starts/stops ChatServer process           |        |
|  | - Manages tor.exe hidden service            |        |
|  | - Displays live .onion address              |        |
|  +---------------------------------------------+        |
+----------------------------------------------------------+
```

---

## Security Architecture

### End-to-End Encryption (E2EE)

All cryptography runs in the **browser** via the WebCrypto API. The server never receives plaintext.

**Key Derivation (PBKDF2-SHA256 — see [docs/PBKDF_600kCrypt.md](docs/PBKDF_600kCrypt.md))**

| Purpose | Input material | Salt | Iterations | Server-blind? |
|---------|----------------|------|------------|:-------------:|
| Join proof (locked) | Room password | `torchat-join-v2:{roomId}` | **600_000** (browser) | n/a (auth only) |
| E2EE key (locked) | Room password | `torchat-e2ee-v2:{roomId}` | **600_000** | **Yes** |
| E2EE key (open) | `torchat-open-material-v2:{roomId}` | `torchat-open-v2:{roomId}` | **600_000** | **No** (key = f(room id)) |
| Server join hash | Client joinProof (base64) | 32-byte CSPRNG | **250_000** | Stored as `250000:salt:hash` |

- Separate KDF domains prevent join proof material from being reused as the message key
- Server stores a second PBKDF2 hash of the join proof — not the password and not the E2EE key
- Message envelopes use `encryptionVersion: pbkdf2-v2` (roomKey) or `ecdh-hybrid-v1` (locked 1:1)
- Open rooms are intentional public channels; UI and join notices state they are **not** server-blind
- **Breaking:** pre-v2 rooms / hashes are unsupported — wipe DB after upgrade

**Message Encryption (AES-256-GCM)**

```
ciphertext = AES-GCM(key=roomKey|hybridKey, plaintext=message,
             AAD="roomId|senderId|nonce|clientSentAtUnixMs")
```

- 12-byte random IV per message; 128-bit GCM tag
- AAD + replay fingerprint include client timestamp (anti-replay / envelope bind)
- `SendEncrypted` forces `SenderId` from the joined connection (unique per room)

---

### ECDH Hybrid + Safety Numbers (locked 1:1)

When exactly **two members** are in a **password** room, clients use hybrid encryption and show a **safety number**:

1. Ephemeral P-256 ECDH on join (non-extractable private key)
2. Public keys relayed via `PeerJoined` / `PeerList` (server is blind relay only)
3. Hybrid key: `HKDF(ECDH shared, salt=roomKeyBytes, info=torchat-ecdh-v1:…)`
4. **Safety number** (Signal-style compare): `SHA-256("torchat-safety-v1" ‖ sorted SPKI pubs)` → three 5-digit groups. Mark verified / TOFU warn if it changes (sessionStorage)
5. Locked group (3+): fallback to password roomKey broadcast (`pbkdf2-v2`); hybrid only in 1:1

| Scenario | Encryption | Forward secrecy |
|----------|-----------|:---:|
| Open room (any size) | PBKDF2 roomKey (`f(roomId)`, pbkdf2-v2) | No |
| Locked 1:1 | ECDH hybrid | Partial (per session; no Double Ratchet) |
| Locked 3+ | PBKDF2 roomKey | No |
| Password leak | Retained ciphertext decryptable until TTL | Prefer short retention |

There is **no Double Ratchet / Megolm** yet — document cryptoperiod expectations accordingly.

---

### Hidden Rooms

Rooms can be marked **hidden** during creation:

- Not listed in public `GetRooms` / `RoomsUpdated`
- Join requires a valid **invite token** (and password if the room is locked)
- Server Manager admin API still lists all rooms
- Generic `"Cannot join room."` / invite `404` (no existence oracle)
- Hidden ≠ server-blind by itself — use a **password** for private E2EE

### Close room to new members

Room creators (public or hidden) can toggle **Close room to new members** in Settings (`manageToken` / creator recovery code):

- All new `JoinRoom` attempts are rejected with generic `"Cannot join room."` (including old invite links / known room id)
- People already in the room stay; if they leave, they cannot rejoin until the creator reopens
- Closing automatically disables the invite link; reopening does **not** re-enable invite (creator must turn it on again)
- Closing joins is **not** server-blind E2EE — an open room still derives its key from the room id; use a **password** for private E2EE

### Lock room later (one-way password)

If a room was created **without** a password, the creator can set one later in Settings (section hidden once locked):

- Client sends `joinProof` only (never the raw password); hub stores the hash once
- Already-locked rooms cannot be unlocked or re-passworded from Settings
- Open-room ciphertext is wiped (`secure_delete` + VACUUM); everyone is ForceLeft and must rejoin with the password for server-blind E2EE

### Delete room (creator)

Creators can **Delete this room** in Settings (confirm checkbox). Same wipe path as Server Manager room delete: ForceLeave → message reclaim → remove room/invites from SQLCipher. The room disappears from Public rooms, My rooms, and the admin room list.

### Slow mode (send cooldown)

Creators can optionally enable **Limit how often members can send** in Settings (default **off**):

- When on, choose a minimum wait: **1 min … 4 h** (allowlisted intervals)
- After a successful send, that member’s `senderId` cannot send again in the room until the interval elapses
- Applies to **everyone in the room, including the room creator**
- Covers text and E2EE attachments (same `SendEncrypted` path)
- Policy is stored per room in SQLite; in-memory last-send times reset if ChatServer restarts
- Changing the setting broadcasts a short system notice to the room

### Creator recovery code & My rooms

Lobby **Profile** stores a **creator recovery code** in `localStorage` (`torchat-creator-secret`):

- **Every** room you create (public or hidden) is bound server-side to a hash of this code (`room_creators`) plus a per-room `manageToken`
- **My rooms** lists **hidden** rooms only via hub `GetMyRooms(creatorSecret)` — public rooms stay under Public rooms
- Join a room you created → **Settings** appears when ownership is proven (`manageToken` and/or recovery code)
- Copy the code and keep it safe; Tor New Identity / clear site data removes it from the browser
- **Restore** pastes a saved code so My rooms and Settings work again without the old per-room manage token
- This is not an E2EE password — it only restores creator listing / manage rights on that server

### Message display (Profile)

Under lobby **Profile** (client-only prefs, stored in the browser):

- **Hide date** / **Hide time** — omit date and/or clock from message timestamps in the chat UI
- **Message time zone** — Tor Browser often spoofs UTC; pick **UTC**, **browser local**, or a fixed offset so stamps match how you want to read them
- Preferences travel with message packets as display hints (`show_sent_date` / `show_sent_time`); they do not change server crypto

### Message expiry (required TTL)

Every message must expire. Send row **Message expiry** offers **30 min … 1 week** (default **8 hours**). There is **no** forever / Never option — the hub rejects missing or non-allowlisted `RetentionSeconds`.

### Tor Hidden Service

- ChatServer binds exclusively to `127.0.0.1` — no clearnet exposure whatsoever
- `tor.exe` is managed by Server Manager (Expert Bundle under `TorConnection/TorBundle/`)
- **Start ChatServer before enabling Tor** — Manager blocks Tor ON until ChatServer is running (and waits for `/api/health` before “ready”)
- When the HS hostname is known, Manager locks CORS to that onion via `POST /api/admin/cors-onion` (and `TORCHAT_ALLOWED_ONION` on next ChatServer start)
- Optional **Full Vanguards** (`Vanguard` checkbox): Python addon + ControlPort for long-lived HS guard-discovery mitigation — see `TorConnection/README_VANGUARDS.txt`
- All client IP addresses are hidden — even the server operator cannot identify users
- **Generate new address** rotates the Hidden Service identity (wipes HS keys). Room/SQL data is separate (**Remove SQL data**)
- Browser soft-warns if the page was opened outside `.onion` / localhost (clearnet proxy exposure risk)

**torrc hardening (applied on every start)**

| Option | Value | Purpose |
|--------|-------|---------|
| `HiddenServiceVersion` | `3` | v3 onion addresses (56-char, Ed25519 — no v2 fallback) |
| `SOCKSPort` | `0` | SOCKS proxy disabled — HS process cannot be abused as a proxy |
| `ClientOnly` | `1` | Prevents the process from acting as a public Tor relay |
| `SafeLogging` | `1` | Strips potentially identifying info (IP addresses) from Tor logs |
| `HiddenServiceMaxStreams` | `20` | Max simultaneous streams per rendezvous circuit. SignalR needs 1–3 per client; 20 is a safe ceiling that stops circuit-flooding DoS |
| `HiddenServiceMaxStreamsCloseCircuit` | `1` | Force-closes a circuit when it exceeds the stream limit, imposing a reconnection cost on the attacker |

---


### Full Vanguards — Guard-Discovery Protection

#### The Problem: Guard-Discovery Attacks

A standard Tor Hidden Service uses a small set of **guard nodes** (entry nodes) as the first hop of its onion circuits. Over time, a network-level adversary can observe which relays the hidden service connects to and gradually de-anonymize the server's real IP by:

1. Mounting repeated circuit-building to force the HS to use relays the attacker controls
2. Correlating traffic timing across many circuits (traffic analysis)
3. Narrowing down candidates until the guard node — and therefore the host — is identified

This is especially dangerous for **long-lived hidden services** (servers that stay online for days or weeks), because the attacker has more time to collect observations.

#### How Vanguards Defends Against This

Full Vanguards (the [mikeperry-tor/vanguards](https://github.com/mikeperry-tor/vanguards) Python addon) implements **multi-layer guard pinning**:

| Layer | What it does |
|-------|-------------|
| **L2 vanguards** | Pins a rotating set of ~4 middle-layer nodes; changes every 1–8 days |
| **L3 vanguards** | Pins a rotating set of ~8 pre-guard nodes; changes every 1–8 hours |

By pinning layers, the number of relays that ever observe the hidden service's traffic is drastically reduced and controlled — making large-scale guard-discovery statistically infeasible over the lifetime of the service.

```
Without Vanguards:   HS → random guard → random middle → rendezvous
With Vanguards:      HS → pinned L3 → pinned L2 → guard → rendezvous
                          (rotated hourly) (rotated daily)
```

#### How TorChat Integrates Vanguards

The feature is **opt-in** via the `Vanguard` checkbox in Server Manager:

1. **torrc changes** (`TorHiddenServiceSetup.cs`): When the checkbox is enabled, `ControlPort 127.0.0.1:27551` and `CookieAuthentication 1` are added to the torrc. Without the checkbox the torrc is minimal — no ControlPort is opened.

2. **Process management** (`VanguardsProcessHost.cs`): After `tor.exe` bootstraps, Server Manager waits until the ControlPort is accepting connections (up to 30 s), then launches:
   ```
   python vanguards.py --control_port 27551 --config vanguards.conf --disable_bandguards
   ```
   `--disable_bandguards` is intentional: bandwidth-based circuit-killing would disconnect chat sessions; the primary L2/L3 guard pinning defence remains fully active.

3. **Authentication**: ControlPort uses Tor's `CookieAuthentication`. The cookie file lives at `%AppData%\TorChat_ServerManager\TorData\control_auth_cookie` and is never exposed to the network — ControlPort listens on loopback only.

4. **Soft-fail design**: If Python 3 or the `stem` library is missing, the checkbox is silently ignored and the hidden service continues to operate normally (without guard pinning). The Server Manager log reports the exact reason.

#### What Vanguards Does NOT Do

- It does **not** replace E2EE — message content is still only protected by client-side AES-GCM
- It does **not** hide the existence of the server from its own guard nodes — it only limits how many relays can observe the HS's circuit-building
- It adds **latency** (~200–500 ms per circuit) due to longer paths; this is a deliberate privacy/performance tradeoff

> **Recommendation**: Enable Full Vanguards if your hidden service is intended to stay online for more than a few hours and you operate in a high-threat environment. Requires Python 3.10+ and `stem` — see `TorConnection/README_VANGUARDS.txt` for setup.

---

### Database Security (SQLCipher)

| Property | Value |
|----------|-------|
| Cipher | AES-256-CBC (SQLCipher) |
| Key storage | Windows DPAPI (`ProtectedData`) |
| Journal mode | WAL — concurrent readers do not block writers |
| Secure delete | `PRAGMA secure_delete=ON`; WAL checkpoint + VACUUM after bulk/room deletes |
| Tables | `messages` (+ `encryption_version`), `rooms`, `server_meta` |
| At-rest | `.db`, `.db-wal`, `.db-shm` all encrypted |

**Manager wipe:** Remove SQL data overwrites database files and `db.key.dpapi` (CSPRNG, one pass) before unlink — optional checkbox also wipes Tor Hidden Service keys. Generate new address securely wipes HS keys then issues a new `.onion`. SSD wear-leveling means absolute forensic erasure is not guaranteed; destroying the DB key is the strongest practical step.

---

### Replay Attack Protection

**In-memory fingerprint cache**

Every received message is fingerprinted:
```
fingerprint = SHA256( UTF-8 bytes of roomId + "|" + nonce + "|" + payloadB64 + "|" + clientSentAtUnixMs )
```
Identical fingerprints within the in-memory window (~120 s) are rejected. Packets older than **120 seconds** (or more than **60 seconds** in the future) are also rejected by the hub freshness checks.

**Restart-gap protection**

On shutdown, `TorChatDb.Dispose()` writes a timestamp to the `server_meta` table. On the next startup, `ReplayCache` reads this timestamp and rejects messages with `ClientSentAtUnixMs < (lastShutdown + 120s)`. This closes the window where in-memory fingerprints are lost during a restart — preventing replays immediately after a reboot.

---

### Admin Token Security

Admin HTTP APIs are protected by a cryptographically generated token:

- **Generated by Server Manager** on each ChatServer start (`ChatServerProcessHost`): 32 random bytes → Base64 (~44 chars)
- Passed to ChatServer via the `TORCHAT_ADMIN_TOKEN` environment variable
- ChatServer reads the token at startup; if missing or shorter than 16 characters, admin APIs return 401
- Requests must send header `X-TorChat-Admin`; comparison uses constant-time equality

| Endpoint | Purpose |
|----------|---------|
| `GET /api/rooms` | Manager room list (metadata) |
| `DELETE /api/rooms/{id}` | Wipe room + ciphertext |
| `POST /api/admin/cors-onion?onion=` | Lock live CORS allowlist to HS hostname |

Tor Browser clients cannot call these — only Server Manager (with the env token) can.

---

### Transport and Browser Security

| Control | Implementation |
|---------|---------------|
| Content Security Policy | `default-src 'self'`, `script-src 'self'`, `style-src 'self'`, `img-src 'self' data: blob:`, `connect-src 'self'`, `font-src 'self'`, `object-src 'none'`, `base-uri 'self'`, `form-action 'self'`, `frame-ancestors 'none'` |
| Subresource Integrity | `integrity=sha256-...` on SignalR + app scripts — browser-enforced |
| Security Headers | `X-Frame-Options: DENY`, nosniff, Referrer-Policy no-referrer, COOP, CORP |
| Permissions-Policy | `geolocation=()`, `microphone=()`, `camera=()` |
| Cache Control | `Cache-Control: no-store` on HTML and `/js/*` |
| AllowedHosts | `127.0.0.1;localhost;*.onion` |
| CORS | Loopback always; `.onion` via `CorsOnionAllowlist` (`TORCHAT_ALLOWED_ONION` and/or live admin POST). Empty allowlist = any `.onion` until locked. No `AllowCredentials` |
| Clearnet banner | Soft UI warning when host is not `.onion` / `127.0.0.1` / `localhost` |
| Request size limits | Kestrel / SignalR ≈ **4 MiB**; `ClientTimeoutInterval` **5 min** (large Tor image frames block pings mid-transfer) |
| Server Header | Removed (`AddServerHeader = false`) |
| DOM Safety | All user content via `textContent` / `createElement` — no `innerHTML` |
| No Third Parties | No analytics, CDN resources, or WebRTC. Creator recovery code may live in `localStorage` (not an E2EE secret) |
| Message cap | Max 500 ciphertext rows per room (oldest trimmed) |
| SRI check | `tools/verify-sri.ps1` — exit 1 on hash mismatch |

---

### Rate Limiting

All limits are enforced **per ConnectionId**. IP-based limits are intentionally not used because all Tor traffic arrives from `127.0.0.1`.

| Endpoint | Limit |
|----------|-------|
| Hub negotiate | 30 / min |
| JoinRoom | 20 / min |
| CreateRoom | 5 / min |
| SendEncrypted | 60 / min |
| GetRooms | 30 / min |
| GetMyRooms | 20 / min |
| Invite manage (status / enable / close / rotate / send interval) | 20 / min |
| `/api/*` (health, invite, admin) | 30 / min |
| Global HTTP | ~600 / min |

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend | ASP.NET Core 9, SignalR |
| Frontend | Vanilla HTML/CSS/JS, WebCrypto API |
| Database | SQLite + SQLCipher (AES-256) |
| Anonymity | Tor Hidden Service (tor.exe bundled) |
| GUI | Windows Forms (.NET 9) |
| Key Storage | Windows DPAPI (desktop); env / key file (Linux/Docker) |
| Real-time | SignalR WebSockets |
| Docker host | `docker/` compose: ChatServer + Tor (no host ports) |

---

## Requirements

**Desktop (Server Manager — recommended on Windows)**

- Windows 10 or later (64-bit)
- [.NET 9 Runtime](https://dotnet.microsoft.com/en-us/download/dotnet/9.0) (for ChatServer)
- [Tor Browser](https://www.torproject.org/) (to connect via .onion)
- Server Manager is self-contained — no separate .NET installation required

**Docker (Linux host / VPS)**

- Docker Engine + Compose v2
- See [docker/README.md](docker/README.md)

---

## Getting Started

### Windows — Server Manager

1. Download the latest release from [Releases](../../releases)
2. Extract to a folder of your choice
3. Launch **`ServerManager.exe`**
4. Set **Project Path** to the extracted folder
5. Click **Start** — ChatServer launches (Production; wait until status shows Running)
6. Click **Tor Connection ON** — requires ChatServer first; your `.onion` address appears
7. Share the `.onion` address with your contacts
8. Open it in **Tor Browser** and create or join a room

> First Tor bootstrap may take 30–60 seconds depending on network conditions.  
> Free binary release does not ship `config.ini` — Server Manager creates it on save/first use.

### Docker — ChatServer + Tor

For headless / VPS hosting without Server Manager. ChatServer is **not** published to the host; only the Tor Hidden Service is reachable.

```bash
cd docker
cp .env.example .env    # set TORCHAT_ADMIN_TOKEN and TORCHAT_DB_KEY
docker compose up --build -d
docker compose exec tor cat /var/lib/tor/hidden_service/hostname
# Set TORCHAT_ALLOWED_ONION in .env, then:
docker compose up -d --force-recreate chatserver
```

Full steps and security notes: **[docker/README.md](docker/README.md)**.

---

## Privacy Guarantees

What the server operator **cannot** see (password / locked rooms):
- Message plaintext (server-blind E2EE — host cannot derive the room key)
- Client IP addresses (all Tor traffic arrives from loopback to ChatServer)
- User accounts (none — display names only)

What the server operator **can** see:
- Room names and display names (metadata)
- When connections are made and dropped
- **Open-room** ciphertext **and** plaintext capability — key = `f(roomId)`; treat open rooms as public channels
- Ciphertext envelopes and timing for all rooms (blind relay + SQLCipher storage)

---

## Project Structure

Full tree, data paths, and flow diagrams: **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**.

```
TorChat/
+-- ServerManager/              WinForms GUI (ChatServer + Tor lifecycle)
|   +-- ChatServerProcessHost.cs    Spawns ChatServer, admin token, Production
|   +-- TorHiddenServiceSetup.cs    tor.exe Hidden Service (+ HS key wipe)
|   +-- SecureFileWipe.cs           CSPRNG overwrite before delete
|   +-- VanguardsProcessHost.cs     Optional Full Vanguards (Python)
|   +-- TorChatPalette.cs           Copper/Forge theme tokens
|   +-- Form1.cs                    Health wait + live CORS onion push
|
+-- ChatServer/                 ASP.NET Core 9 (net9.0 — Windows + Linux/Docker)
|   +-- Hubs/ChatHub.cs             SignalR: join/invite/close/slow-mode/lock/delete
|   +-- Security/ReplayCache.cs     Fingerprint cache + restart-gap protection
|   +-- Security/CorsOnionAllowlist.cs  Mutable CORS .onion allowlist
|   +-- Security/HubMethodRateLimiter.cs  Per-ConnectionId hub limits
|   +-- Services/TorChatDb.cs       SQLCipher: WAL, secure_delete, reclaim
|   +-- Services/RoomRegistryService.cs  Auth (250k iter:salt:hash), hidden, joins_closed, send_interval
|   +-- Services/RoomInviteService.cs    Invite + manage/creator secrets
|   +-- Services/RoomSendCooldownService.cs  Slow-mode last-send tracker
|   +-- Services/RoomLifecycleService.cs Shared room wipe (admin + creator)
|   +-- wwwroot/js/
|       +-- crypto.js               PBKDF2-v2 600k + AES-GCM + ECDH + safety number
|       +-- chat-hub.js             SignalR client
|       +-- room-invite.js          Invite + room Settings
|       +-- client-profile.js       Display name, recovery, time prefs
|       +-- file-attach.js          E2EE attachments
|       +-- app.js                  Lobby, chat, TTL, timezone, origin warning
|
+-- docker/                     Compose: ChatServer + Tor HS (no host ports)
+-- TorConnection/              Tor Expert Bundle + optional Vanguards addon
+-- tools/verify-sri.ps1        Compare index.html SRI to wwwroot JS
+-- docs/ARCHITECTURE.md        Repository map & architecture
+-- docs/PBKDF_600kCrypt.md     PBKDF2-v2 clean-break spec
+-- docs/OPEN_ROOM_GROUP_KEY.md Open rooms = public channel (room-id key)
+-- create_release.bat          Builds TorChat_Release\ (no shipped config.ini)
```

---

## Source Code

The binary release is **free to use**. If you want to study, audit, or build upon the source code, it is available for purchase:

---

> ### [Purchase Source Code — $49](https://www.dev-offcode.com/TorChatPage.html)
>
> Includes the complete C# source (ChatServer + ServerManager), JavaScript crypto layer, build scripts, and personal/educational use license.

---

## Terms of Service & Legal Disclaimer

**Please read these terms before using OffCode Tor Chat.** By hosting, operating, joining, or otherwise using this software, you acknowledge that you have read, understood, and agree to be bound by them.

### Important notice

This software is intended **only for lawful, authorized use**. Do not use it for illegal activity, unauthorized access, harassment, exploitation, trafficking, fraud, terrorism, child sexual abuse material, or any other criminal conduct under applicable law.

### Host responsibility (full transfer of operational liability)

OffCode Tor Chat is **self-hosted software**. The person or entity that runs ChatServer / Server Manager and publishes the Tor Hidden Service (the **“Host”**) is solely and fully responsible for:

- How the software is configured, deployed, and operated
- Who is invited or allowed to access the `.onion` address and rooms
- Content transmitted through or stored on the Host’s instance (including ciphertext metadata and any lawful requests for access)
- Compliance with all applicable local, national, and international laws in every jurisdiction where the Host or users operate
- Any misuse of the instance by the Host or by third parties the Host enables

The author(s), publisher(s), and distributor(s) of this software do **not** operate your chat service, do not control your Hidden Service, and are **not responsible** for your use of the software or for communications on your instance.

### User & guest responsibilities

Every user (Host or guest) is solely responsible for their own conduct and must comply with all applicable laws, regulations, court orders, and third-party rights. You must not use this software to violate privacy rights, commit fraud, harass or threaten others, distribute illegal content, or facilitate any crime.

### Prohibited uses

- Any criminal activity or conspiracy to commit crime
- Unauthorized access to systems, accounts, or data
- Harassment, stalking, intimidation, or doxxing
- Distribution of illegal content, including CSAM or content involving exploitation of minors
- Fraud, phishing, identity theft, or money laundering
- Terrorism, violent extremism, or trafficking
- Any activity that violates export, sanctions, cybercrime, or communications laws in your jurisdiction

### No warranty (AS IS)

TO THE MAXIMUM EXTENT PERMITTED BY APPLICABLE LAW, THE SOFTWARE IS PROVIDED **“AS IS” AND “AS AVAILABLE”**, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, TITLE, NON-INFRINGEMENT, SECURITY, AVAILABILITY, OR ERROR-FREE OPERATION. YOU USE THE SOFTWARE AT YOUR OWN RISK.

### Limitation of liability

TO THE MAXIMUM EXTENT PERMITTED BY LAW, THE AUTHOR(S), PUBLISHER(S), AND DISTRIBUTOR(S) SHALL NOT BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL, EXEMPLARY, OR PUNITIVE DAMAGES, OR ANY LOSS OF DATA, PROFITS, GOODWILL, OR BUSINESS INTERRUPTION, ARISING OUT OF OR RELATED TO THE SOFTWARE OR YOUR USE OF IT — INCLUDING MISUSE BY A HOST OR USERS — EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGES. WHERE LIABILITY CANNOT BE EXCLUDED, IT IS LIMITED TO THE GREATER OF ZERO OR THE AMOUNT YOU PAID (IF ANY) FOR THE SOFTWARE COPY AT ISSUE.

### Indemnification

You agree to indemnify, defend, and hold harmless the author(s), publisher(s), and distributor(s) from and against any claims, damages, losses, liabilities, costs, and expenses (including reasonable attorneys’ fees) arising from your hosting, configuration, or use of the software, your content or users’ content, or your breach of these terms or applicable law.

### Privacy & encryption notice

End-to-end encryption and Tor routing improve confidentiality against many third parties, but they do **not** make illegal use lawful, do not guarantee anonymity against all adversaries, and do not relieve the Host of legal duties that may apply to operating a communications service in their jurisdiction.

### Not legal advice

These terms are a general disclaimer and license condition, not legal advice. Laws differ by country. If you are unsure whether your intended use is lawful, consult a qualified attorney before proceeding.

### Acceptance

By downloading, installing, hosting, joining, or continuing to use OffCode Tor Chat, you accept these Terms of Service and Legal Disclaimer. If you do not agree, do not use the software and stop any running instance under your control.

---

## License

The **compiled binary** is free for personal and commercial use.  
The **source code** is available at [dev-offcode.com](https://www.dev-offcode.com/TorChatPage.html) ($49, personal developer license — ChatServer + Server Manager + E2EE client).  
Commercial redistribution / resale of source requires a separate license — contact via the website.

---

*Built for privacy. Questions or commercial inquiries: [dev-offcode.com](https://www.dev-offcode.com/TorChatPage.html)*

# 🧅 OffCode Tor-Chat E2EE

**Anonymous chat over Tor Hidden Services with browser E2EE**

![Platform](https://img.shields.io/badge/platform-Windows-blue?logo=windows)
![Platform](https://img.shields.io/badge/platform-macOS-lightgrey?logo=apple)
![Platform](https://img.shields.io/badge/platform-Ubuntu-E95420?logo=ubuntu)
![.NET](https://img.shields.io/badge/.NET-9.0-purple?logo=dotnet)
![Tor](https://img.shields.io/badge/Tor-Hidden%20Service-.onion-7D4698?logo=torproject)
![Encryption](https://img.shields.io/badge/encryption-AES--256--GCM-green)
![Source](https://img.shields.io/badge/source%20code-$49-orange)

*Self-hosted anonymous chat over Tor. Open the* `.onion` *in Tor Browser. Password rooms are server-blind against a passive relay; open rooms are public encrypted channels (key = room id).*

---

## What is TorChat?

TorChat is a **self-hosted anonymous chat server** that runs entirely over the Tor network as a hidden service (`.onion` address). It combines independent layers of protection:

1. **Tor Network** — hides both the server and users’ real IP addresses
2. **Optional Full Vanguards** — hardens long-lived hidden services against guard-discovery / traffic-analysis attacks
3. **End-to-End Encryption** — the Tor Browser page encrypts messages (AES-256-GCM) before send; the relay stores ciphertext only
4. **Encrypted Storage** — data at rest is protected by SQLCipher (AES-256)

**Room type matters:** open rooms derive their key from the room id and are not server-blind. Password-protected rooms keep the key secret from a **passive** relay. A host that serves the page can still replace JavaScript (SRI is not an independent anchor). Hidden rooms stay off the public list and require an invite; add a password for relay-blind message content.

Anyone who knows the `.onion` address (and invite / password when required) can connect in **Tor Browser**. No accounts, no phone numbers.

---

## Key Features

| Feature | Details |
|---------|---------|
| Tor Hidden Service | Automatic `.onion` (v3); intro DoS + HS PoW on Tor ≥ 0.4.8 |
| Server-blind E2EE | Password rooms: AES-256-GCM; a passive relay cannot derive the room key |
| Open rooms | Public channels — encrypted in browser; key = f(room id) |
| Optional Feed | Public plaintext posts/comments (default **off**). Not E2EE. Host can read and delete |
| ECDH Hybrid (1:1) | Locked 1:1: ephemeral P-256 + HKDF; safety number (8×5 decimal, 128-bit) + TOFU verify |
| Sender binding | Unique `senderId` per room; server overwrites packet identity |
| Encrypted Database | SQLCipher with DPAPI-protected key (AES-256 at rest) |
| Real-time Chat | ASP.NET Core SignalR WebSockets |
| Room System | Open, password-protected, and/or hidden (+ invite) |
| Hidden Rooms | Off public list; join via invite (password still required if locked) |
| Close room | Creator can block all new joins (invite included); existing members stay; **creator can rejoin** with recovery / manage token |
| Max members | Optional online cap (2–50); default off. New joins rejected when full; people already in stay |
| Lock later | Open rooms can be locked once with a password (server-blind E2EE); wipes open history |
| Delete room | Creator can wipe the room + ciphertext from the server (secure_delete + VACUUM) |
| Slow mode | Optional per-room send cooldown (1 min … 4 h); applies to everyone including the creator |
| Disable attachments | Creator can turn off file/image sending in a room (text still allowed); server size-gates ciphertext |
| Announcement | Optional read-only mode; only the room creator can send (default off) |
| Creator recovery | Profile **Generate code** (no room required). Binds rooms you create → Settings (and **My rooms** for hidden) and identifies Feed posts; Copy / Restore |
| Message display prefs | Hide date and/or time stamps; pick message time zone (Tor often shows UTC) |
| Message TTL | Required auto-expiry (30 min … 1 week; default 8 h). Forever retention is not offered |
| E2EE attachments | FileSeal v3: chunked AES-GCM (max 32 MiB); E2EE manifest via `SendEncrypted`; server stores opaque chunks only. Images: allowlist strip (JPEG EOI-cut, PNG chunks, GIF/WebP) in-browser before encrypt — no canvas when possible |
| Secure erasure | `secure_delete` + WAL checkpoint/VACUUM on expiry & room wipe; Manager overwrites DB/key (optional HS keys) |
| Replay Protection | Server: SHA-256 fingerprint + 120 s / 60 s window + restart-gap. Browser: `(senderId, nonce)` Map (room session / cap 5000); live 5 min / +2 min; history sync skips the clock |
| Pinned message | Room creator pins one ciphertext envelope (right-click / Shift+F10). Server stores a pointer only; members decrypt in the browser. TTL still applies |
| CORS lock | `TORCHAT_ALLOWED_ONION` + live `POST /api/admin/cors-onion` (Manager; no ChatServer restart). Empty allowlist denies `.onion` CORS and **403**s onion WebSocket Upgrade; same-origin SignalR still uses SSE / long polling (slower). Loopback always allowed |
| Clearnet / insecure host | Fail-closed overlay unless host is loopback (`localhost`, `127.0.0.1`, `::1`) or a Tor v3 `.onion`; requires `isSecureContext` + `crypto.subtle` before SignalR |
| Trusted local client (optional) | `TorChat.ClientHost` can embed the same UI on loopback; default is the `.onion` page in Tor Browser |
| SRI verify | `tools/verify-sri.ps1` checks `index.html` hashes against wwwroot JS (build consistency, not a malicious-host trust anchor) |
| Disable Room Creation | Manager checkbox; blocks onion Create room immediately; Manager Create room still works |
| GUI Manager | Avalonia (Windows / Linux / macOS): ChatServer + Tor HS; optional Full Vanguards; cancellable Tor start; orphan-process cleanup; Reset guards; Generate new address (stop Vanguards then Tor); public rooms; website create cap/disable; operator recovery (Admin key → Profile Restore); Save `config.ini` + optional logon start. WinForms `ServerManager/` remains in source only |
| Docker host | Optional compose stack: ChatServer + Tor HS (no clearnet host ports) |
| No Accounts | Zero registration — join with a display name only |
| Privacy-focused storage | No IP logging; no room passwords. Chat stores ciphertext envelopes; optional Feed stores plaintext bodies. SQLCipher until expiry or deletion |

---

## How It Works

```
+----------------------------------------------------------+
|                   Tor Browser (User)                     |
|  Opens http://ADDRESS.onion/                             |
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
|  Serves chat UI + SignalR/API                            |
|  Receives: ciphertext blob - NEVER decrypts it           |
|  Stores:   encrypted payload in SQLCipher DB             |
|  Broadcasts: same ciphertext to room members             |
|                                                          |
|  +---------------------------------------------+        |
|  | Server Manager (Avalonia)                   |        |
|  | - Starts/stops ChatServer process           |        |
|  | - Manages tor.exe hidden service            |        |
|  | - Optional Full Vanguards (ControlPort)     |        |
|  | - Displays live .onion address              |        |
|  +---------------------------------------------+        |
+----------------------------------------------------------+
```

---

## Security Architecture

### End-to-End Encryption (E2EE)

Default production use: Server Manager publishes a Hidden Service; guests open `http://ADDRESS.onion/` in **Tor Browser**. Encryption happens in the browser via WebCrypto. A **passive** relay does not receive the room password, room key, or plaintext in password rooms. A host that serves the HTML can replace JavaScript and SRI hashes together. Optional `TorChat.ClientHost` keeps a local copy of the UI. Open rooms remain public channels and are not server-blind.

**Key Derivation (PBKDF2-SHA256 — [docs/PROTOCOL.md](docs/PROTOCOL.md); historical write-up [docs/archive/PBKDF_600kCrypt.md](docs/archive/PBKDF_600kCrypt.md))**

| Purpose | Input material | Salt | Iterations | Server-blind? |
|---------|----------------|------|------------|:-------------:|
| Join proof (locked) | NFC(password) | `torchat-join-v3:{roomId}` | **600_000** (browser) | n/a (auth only) |
| E2EE key (locked) | NFC(password) | `torchat-e2ee-v3:{roomId}` | **600_000** | **Yes, with trusted client** |
| E2EE key (open) | `torchat-open-material-v3:{roomId}` | `torchat-open-v3:{roomId}` | **600_000** | **No** (key = f(room id)) |
| Server join hash | Client joinProof (base64) | 32-byte CSPRNG | **250_000** | Stored as `250000:salt:hash` |

- Separate KDF domains prevent join proof material from being reused as the message key
- Server stores a second PBKDF2 hash of the join proof — not the password and not the E2EE key
- Message envelopes use `encryptionVersion: pbkdf2-v3` (roomKey) or `ecdh-hybrid-v3` (locked 1:1). Unknown versions fail closed.
- Passwords: Unicode NFC then UTF-8 (not RFC 8265 OpaqueString)
- Open rooms are intentional public channels; UI and join notices state they are **not** server-blind
- **Breaking:** v2 rooms / hashes / envelopes are unsupported. First start after upgrade wipes SQLCipher (`protocol_epoch=3`)
- Byte-level spec: [docs/PROTOCOL.md](docs/PROTOCOL.md) (not an audit)

**Message Encryption (AES-256-GCM)**

```
ciphertext = AES-GCM(key=roomKey|hybridKey, plaintext=message,
             AAD="encryptionVersion|roomId|senderId|nonce|clientSentAtUnixMs")
```

- 12-byte random IV per message (NIST SP 800-38D 96-bit; decrypt and hub reject other lengths); 128-bit GCM tag
- AAD fields cannot contain `|` (`roomId` / `nonce` `A-Za-z0-9_-`, `senderId` lowercase 32 hex, `clientSentAtUnixMs` digits). Hub regexes use `\z` (no trailing LF)
- AAD + replay fingerprint include client timestamp (anti-replay / envelope bind)
- `SendEncrypted` forces `SenderId` from the joined connection (unique per room)
- Room salt for HKDF lives in a `key-material.js` closure (not `App.state`). Leave/join zeroes it. Non-extractable `CryptoKey` objects on the origin remain usable by any script on that page.

---

### ECDH Hybrid + Safety Numbers (locked 1:1)

When exactly **two members** are in a **password** room, clients use hybrid encryption and show a **safety number**:

1. Ephemeral P-256 ECDH on join (non-extractable private key)
2. Public keys relayed via `PeerJoined` / `PeerList` (the relay sees them; it does not receive private ECDH keys when the verified local client is used)
3. Hybrid key: `HKDF(ECDH shared, salt=deriveRoomKeyRawBytes() 32-byte PBKDF2 output, info=torchat-ecdh-v3:…)` — info sorts **by senderId** and pairs each id with that peer’s SPKI (length-prefixed). Swapping SPKIs between ids changes the session key. Duplicate senderIds are rejected.
4. **Safety number** (Signal-style compare): import both SPKIs as P-256, then `SHA-256("torchat-safety-v1" ‖ sorted SPKI pubs)` → eight 5-digit groups from the first 16 digest bytes, **least-significant group first** (last group `00000`–`03402`; 128-bit entropy). `hashHex` is the full SHA-256. TOFU in `sessionStorage`. Leave/rejoin generates a new pair, so **both clients show the same new number**; the UI says `Key rotated after rejoin. Compare again.` That is expected, not by itself MITM.
5. Locked group (3+): password `roomKey` (`pbkdf2-v3`); hybrid only in 1:1. Shared roomKey does **not** authenticate which member sent a packet.

After a reload or a new join the client generates a new P-256 pair (session PFS). Stored `ecdh-hybrid-v3` ciphertext from a prior ephemeral session cannot decrypt; the UI shows a previous-session placeholder. `pbkdf2-v3` history still opens with the room password. ECDH private keys are non-extractable and are not written to `localStorage` / `sessionStorage` / IndexedDB.

| Scenario | Encryption | Forward secrecy |
|----------|-----------|:---:|
| Open room (any size) | PBKDF2 roomKey (`f(roomId)`, pbkdf2-v3) | No |
| Locked 1:1 | ECDH hybrid (`ecdh-hybrid-v3`) | Partial (per session; no Double Ratchet) |
| Locked 3+ | PBKDF2 roomKey | No |
| Password leak | Retained ciphertext decryptable until TTL | Prefer short retention |

There is **no Double Ratchet / Megolm** yet — document cryptoperiod expectations accordingly.

---

### Hidden Rooms

Rooms can be marked **hidden** during creation:

- Not listed in public `GetRooms` / `RoomsUpdated`
- Join requires a valid **invite token** (and password if the room is locked)
- Server Manager admin API still lists all rooms
- Generic `"Cannot join room."` / invite `404` (error **text** does not leak existence). The hidden/closed gates also pay the same dummy PBKDF2 cost as every other denial, so **response timing** does not distinguish those rooms from missing ones either
- Hidden ≠ server-blind by itself — use a **password** for private E2EE

### Close room to new members

Room creators (public or hidden) can toggle **Close room to new members** in Settings (`manageToken` / creator recovery code):

- New `JoinRoom` attempts from non-owners are rejected with generic `"Cannot join room."` (including old invite links / known room id)
- People already in the room stay
- The **room creator can rejoin** while closed by proving ownership on join (`creatorSecret` recovery code and/or per-room `manageToken`) — same proof as Settings. Server verifies owner secrets only when the closed/hidden gate would reject; closing also disables the invite link; the owner bypass skips the invite requirement for hidden rooms
- Password rooms still require `joinProof` — recovery does not replace the room password
- Treat the recovery code as a **capability token** (Settings + closed rejoin + listing), not an E2EE secret
- Reopening does **not** re-enable invite (creator must turn it on again)
- Closing joins is **not** server-blind E2EE — an open room still derives its key from the room id; use a **password** for private E2EE
- Tor Browser New Identity / clear site data removes the recovery code from the browser; Copy / Restore it or you lose creator rejoin and Settings

### Max members (online cap)

Creators can optionally enable **Limit how many can be in the room** in Settings (default **off**):

- When on, set a number **2–50**. New `JoinRoom` attempts are rejected once that many people are **online**
- People already in the room stay if you lower the cap; they are not kicked
- Applies to **everyone, including the room creator** (no owner bypass)
- Public rooms: `"Room is full."` Hidden rooms: generic `"Cannot join room."` (no extra existence leak)
- Policy is stored per room in SQLite (`max_members`; `0` = unlimited). Changing the setting broadcasts a short system notice

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

### Disable attachments

Creators can toggle **Disable file and image sending** in Room Settings (default **off** — attachments allowed):

- Members can still send text; the Attach control is disabled when the policy is on
- All members receive an `AttachmentsPolicy` SignalR event on join and when the creator toggles
- Server cannot decrypt envelopes, so enforcement is a **smaller PayloadB64 size gate** (~16 KiB) when disabled — blocks practical attachment ciphertext; oversized text is also rejected. This is intentionally not content-type inspection (blind relay)
- Already-received attachments in history remain visible; only new sends are blocked
- Policy is stored per room in SQLite (`attachments_disabled`)

### Announcement (read-only)

Creators can optionally enable **Only the room creator can send** in Settings (default **off**):

- When on, members can read history but cannot send text or FileSeal attachments
- The creator can still send (slow mode still applies). Owner proof is checked on `SendEncrypted` / `BeginAttachmentUpload` (`manageToken` / recovery code) and is **not** stored with ciphertext
- Members get a disabled composer and a toast if they try; all members receive `ReadOnlyPolicy` on join and when the creator toggles
- Policy is stored per room in SQLite (`read_only`)

### Pinned messages

The room **creator** can pin **one** chat message at a time (new pin replaces the previous):

- In the room, right-click a message (or **Shift+F10** / long-press) → **Pin message** / **Unpin message**. The menu is an in-app overlay (`room-pin.js`), not a document-wide native context menu
- A **Pinned** bar above the transcript shows a short decrypted preview; click it to scroll to the original bubble
- Hub `PinMessage` / `UnpinMessage` require **presence in the room** plus owner proof (`manageToken` / recovery code). The client sends only `nonce` + `clientSentAtUnixMs` — **never plaintext**. The server copies the already-stored ciphertext envelope and broadcasts `PinState` to the group. Joiners get `PinState` after history replay
- Identification is `(roomId, nonce, clientSentAtUnixMs)` so the pin cannot bind to a different payload. `gk-wrap-v1` is rejected by the hub (`Message rejected.`) and cannot be pinned
- No pinner identity and no extra pin timestamp (the envelope already has UTC-minute clocks)
- The pin dies with the message: TTL cleanup CASCADE-deletes `room_pins`; wipe / lock-later history clear too. The 500-message cap **does not** trim the pinned row while it is still unexpired
- Owner-proof failures use `Cannot manage invite.`; missing / other-room / expired / wrap use `Cannot pin message.` Rate limit is the invite-manage budget (20 / min). Open rooms remain host-readable by design (`key = f(roomId)`)

### Creator recovery code & My rooms

Lobby **Profile** stores a **creator recovery code** (`torchat-creator-secret`) in `sessionStorage` **by default**:

- Treat it as a **capability token**, not an E2EE secret: possession proves room ownership on this server. Leave clears the chat transcript in the DOM; it does **not** delete this code. **Remember on this device** writes it to `localStorage` (until Tor Browser **New Identity** / site data). Tab close drops the default session copy. A leftover pre-hardening `localStorage` code (no persist flag) moves to this tab only after a successful session write, once the toast UI is loaded. English toast plus a Profile notice stay until **Remember on this device** or Copy.
- Capabilities include: Room **Settings** (close joins, max members, disable attachments, announcement, slow mode, invite, lock, delete), **pin one message** in the room, **My rooms** listing, restore toast with `IncludePublic`, **rejoin when joins are closed** / hidden without invite, and Feed post/comment/like/delete of **your** Feed rows (`feedKey` derived in the browser)
- Server stores only a **SHA-256** hash of the code (`room_creators.creator_hash`) — same model as manage tokens. Intentional for high-entropy tokens (not PBKDF2; join proofs use PBKDF2 because they derive from user passwords)
- **Every** room you create (public or hidden) is bound server-side to that hash plus a per-room `manageToken`. Create room **reuses** an existing Profile code
- **Generate code** (Profile, no code yet) writes a new token in this browser only — no hub call, no room required. Feed write needs this code; Feed does **not** create it on like/post
- **Generate new code** (Profile, code already exists) confirms, then hub `RebindCreatorSecret`: updates all bound rooms to the new hash and **immediately invalidates** the old code. Feed ownership does **not** move (old posts stay undeletable by you). Plain **Restore** only writes this browser’s storage and does **not** revoke an old code on the server
- Prefer the long autogenerated code; do not restore short or guessable strings
- **My rooms** lists **hidden** rooms only via hub `GetMyRooms(creatorSecret)` — public rooms stay under Public rooms (`IncludePublic` is for restore summary and is rate-limited more tightly)
- Join a room you created → **Settings** appears when ownership is proven (`manageToken` and/or recovery code)
- Copy the code and keep it safe; Tor New Identity / clear site data removes it from the browser
- **Restore** pastes a saved code so My rooms and Settings work again without the old per-room manage token; a toast lists which rooms are bound (Hidden → **My rooms**, Public → **Public rooms**)
- Password rooms still require `joinProof` — recovery does not replace the room password
- Client sends recovery / manage proof on `JoinRoom` only when this browser likely owns the room (manage token, owned-room cache, My rooms); the server verifies owner secrets only when a closed/hidden gate would otherwise reject

### Invite links (hashed at rest)

- Opaque invite tokens are shown once on create / **New link**; the server stores **SHA-256** of the token only (`room_invites.token`)
- The browser may cache the last plaintext link for copy/Settings (`sessionStorage` by default; `localStorage` if persist is on); after tab close / New Identity, use **New link** if the URL field is empty
- Legacy plaintext invite rows are hashed on ChatServer startup

### Message display (Profile)

Under lobby **Profile** (client-only prefs, stored in the browser):

- **Hide date** / **Hide time** — omit date and/or clock from message timestamps in the chat UI
- **Message time zone** — Tor Browser often spoofs UTC; pick **UTC**, **browser local**, or a fixed offset so stamps match how you want to read them
- Preferences travel with message packets as display hints (`show_sent_date` / `show_sent_time`); they do not change server crypto

### Message expiry (required TTL)

Every message must expire. Send row **Message expiry** offers **30 min … 1 week** (default **8 hours**). There is **no** forever / Never option — the hub rejects missing or non-allowlisted `RetentionSeconds`.

### Tor Hidden Service

- **Server Manager:** ChatServer binds exclusively to `127.0.0.1` — no clearnet/LAN bind
- **Docker:** ChatServer listens on `0.0.0.0:8080` *inside* the compose network only (no host `ports:`); Tor HS is the only public path
- `tor.exe` is managed by Server Manager (Expert Bundle under `TorConnection/TorBundle/`). **SHA-256 pin** in `TorConnection/tor-expert-bundle.pin.json` — currently **0.4.9.13** (`tor.exe` `90bbdcafd586feea608a5e9b7d3959f4ee194f7770755cfde8fab240e9773ad1`). Manager **does not download** Tor. A replaced or unpinned binary fails closed. This pin is ≥ 0.4.8 so HS intro DoS + Proof-of-Work are written to torrc.
- **Start ChatServer before enabling Tor** — Manager blocks Tor ON until ChatServer is running (and waits for `/api/health` on the admin port before “ready”)
- When the HS hostname is known, Manager locks CORS to that onion via `POST /api/admin/cors-onion` (and `TORCHAT_ALLOWED_ONION` on next ChatServer start)
- Optional **Full Vanguards** (`Vanguard` checkbox): Python addon + ControlPort for long-lived HS guard-discovery mitigation — see `TorConnection/README_VANGUARDS.txt`
- All client IP addresses are hidden — even the server operator cannot identify users
- **Generate new address** rotates the Hidden Service identity. Stop Vanguards then Tor first. It wipes HS keys, `vanguards.state`, and Tor’s L1 `state` so the new onion is not linkable via old guards, then waits for you to start Tor. Room/SQL data and the Admin key are separate (**Remove SQL data**). SSD overwrite is not forensic — use BitLocker.
- Chat UI **blocks** unless the page host is loopback (`localhost`, `127.0.0.1`, `::1`) or a Tor **v3** `.onion`, **and** Web Cryptography is available (`isSecureContext` + `crypto.subtle`). Overlay + `<noscript>` are fail-closed (`secure-boot.js`). Naive `*.onion` suffixes such as `127.0.0.1.onion` are rejected. Same-origin reverse proxies that rewrite HTML and drop SRI are out of scope

**torrc hardening (applied on every start)**

| Option | Value | Purpose |
|--------|-------|---------|
| `HiddenServiceVersion` | `3` | v3 onion addresses (56-char, Ed25519 — no v2 fallback) |
| `SOCKSPort` | `0` | SOCKS proxy disabled — HS process cannot be abused as a proxy |
| `ClientOnly` | `1` | Prevents the process from acting as a public Tor relay |
| `SafeLogging` | `1` | Strips potentially identifying info (IP addresses) from Tor logs |
| `HiddenServiceMaxStreams` | `20` | Max simultaneous streams per rendezvous circuit. SignalR needs 1–3 per client; 20 is a safe ceiling that stops circuit-flooding DoS |
| `HiddenServiceMaxStreamsCloseCircuit` | `1` | Force-closes a circuit when it exceeds the stream limit, imposing a reconnection cost on the attacker |
| `HiddenServiceEnableIntroDoSDefense` | `1` | Rate-limit INTRODUCE2 at honest intro points (Tor Proposal 305). Written only if `tor --version` ≥ 0.4.8 |
| `HiddenServiceEnableIntroDoSBurstPerSec` | `200` | Tor man-page default intro burst |
| `HiddenServiceEnableIntroDoSRatePerSec` | `25` | Tor man-page default intro rate |
| `HiddenServicePoWDefensesEnabled` | `1` | Equi-X v1 Proof-of-Work (Proposal 327). Dormant until the HS queue is overloaded; Tor Browser already has the client |
| `HiddenServicePoWQueueRate` | `250` | Tor man-page default PoW queue drain rate |
| `HiddenServicePoWQueueBurst` | `2500` | Tor man-page default PoW queue burst |
| `HiddenServicePort` | `80` + chat port | Only these two reach ChatServer. The admin port (`TORCHAT_ADMIN_PORT`) is deliberately **never** mapped, which is what keeps the admin API off the `.onion` |

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
| **L2 vanguards** | Pins a rotating set of ~4 middle-layer nodes; addon default lifetime 24–1080 h (~1–45 days) |
| **L3 vanguards** | Pins a rotating set of ~8 pre-guard nodes; addon default lifetime 1–48 hours |

By pinning layers, the number of relays that ever observe the hidden service's traffic is drastically reduced and controlled — making large-scale guard-discovery statistically infeasible over the lifetime of the service.

```
Without Vanguards:   HS → random guard → random middle → rendezvous
With Vanguards:      HS → pinned L3 → pinned L2 → guard → rendezvous
                          (rotated hourly) (rotated daily)
```

#### How TorChat Integrates Vanguards

The feature is **opt-in** via the `Vanguard` checkbox in Server Manager:

1. **torrc changes** (`TorHiddenServiceSetup.cs`): When the checkbox is enabled, `ControlPort 127.0.0.1:27551` and `CookieAuthentication 1` are added to the torrc. Without the checkbox the torrc is minimal — no ControlPort is opened.
2. **Process management** (`VanguardsProcessHost.cs`): After `tor.exe` starts, Server Manager waits until ControlPort `127.0.0.1:27551` is accepting connections (up to 30 s; typically the “Opened Control listener” line, not Bootstrapped 100%), then launches:
   ```
      python vanguards.py --control_port 27551 --config vanguards.conf --state <AppData>\TorData\vanguards.state --disable_bandguards
   ```

      Packaged builds use `TorConnection/Python/<rid>/python.exe` (stem included). From-source WinForms/Avalonia can fall back to PATH Python. `--disable_bandguards` is intentional: bandwidth-based circuit-killing would disconnect chat sessions; the primary L2/L3 guard pinning defence remains fully active. Unchecking Tor while Vanguards is on stops the addon **first**, then `tor.exe`. Checking Vanguard while Tor is already running without ControlPort asks to restart Tor; unchecking Vanguard only stops the addon.
3. **State file**: Avalonia uses `%AppData%\TorChat_ServerManager_MP\TorData\vanguards.state` (Linux `~/.config/`, macOS Application Support). WinForms source uses `%AppData%\TorChat_ServerManager\TorData\vanguards.state`. Absolute `--state`, next to Tor’s `DataDirectory`. A leftover `vanguards.state` beside the addon is migrated once, then wiped. The file is never shipped in a release. **Generate new address** wipes it (and `.bak`) with the HS keys and Tor L1 `state`.
4. **Authentication**: ControlPort uses Tor's `CookieAuthentication`. The cookie file lives next to that `TorData` folder (`control_auth_cookie`) and is never exposed to the network — ControlPort listens on loopback only.
5. **Soft-fail design**: If bundled (or PATH) Python / `stem` is missing, the checkbox is ignored and the hidden service continues without guard pinning. The Server Manager log reports the exact reason.

#### What Vanguards Does NOT Do

- It does **not** replace E2EE — message content is still only protected by client-side AES-GCM
- It does **not** hide the existence of the server from its own guard nodes — it only limits how many relays can observe the HS's circuit-building
- It adds **latency** (~200–500 ms per circuit) due to longer paths; this is a deliberate privacy/performance tradeoff

> **Recommendation**: Enable Full Vanguards if your hidden service is intended to stay online for more than a few hours and you operate in a high-threat environment. Binary releases already bundle Python + stem. From source, see `TorConnection/README_VANGUARDS.txt`.

---

### Database Security (SQLCipher)

| Property | Value |
|----------|-------|
| Cipher | AES-256-CBC (SQLCipher) |
| Key storage | Windows: DPAPI (`ProtectedData` → `db.key.dpapi`). Linux/Docker: `TORCHAT_DB_KEY` (64 hex), `TORCHAT_DB_KEY_FILE`, or `db.key` in the data dir |
| Journal mode | WAL — concurrent readers do not block writers |
| Secure delete | `PRAGMA secure_delete=ON`; WAL checkpoint + VACUUM after bulk/room deletes |
| Tables | `messages`, `rooms`, `server_meta`, `room_invites`, `room_manage_secrets`, `room_creators`, `room_pins`, `attachment_transfers` |
| At-rest | `.db`, `.db-wal`, `.db-shm` all encrypted |

**Manager wipe:** Remove SQL data overwrites database files and `db.key.dpapi` (CSPRNG, one pass) before unlink — optional checkbox also wipes Tor Hidden Service keys. It does **not** delete the operator recovery (WinForms `%LocalAppData%\TorChat_ServerManager\operator_creator.dpapi`; Avalonia keyring / `operator_creator.key` under `TorChat_ServerManager_MP`). Generate new address securely wipes HS keys, Vanguards state, and Tor’s L1 `state`, then waits for you to start Tor so the new `.onion` gets new guards. SSD wear-leveling means absolute forensic erasure is not guaranteed; use BitLocker (or equivalent). Destroying the DB key is the strongest practical step for chat data.

---

### Replay Attack Protection

**Server — in-memory fingerprint cache**

Every received message is fingerprinted:

```
fingerprint = SHA256( UTF-8 bytes of roomId + "|" + nonce + "|" + payloadB64 + "|" + clientSentAtUnixMs )
```

Identical fingerprints within the in-memory window (~120 s) are rejected. Packets older than **120 seconds** (or more than **60 seconds** in the future) are also rejected. Those freshness and replay failures all return the generic hub error `Message rejected.` so they cannot be used as a millisecond clock oracle. `ClientSentAtUnixMs` is floored to the UTC minute in the browser before AES-GCM AAD; hub `SentAtUnixMs` / `ExpireAtUnixMs` / room `CreatedAtUnixMs` use the same grain.

**Server — restart-gap protection**

On shutdown, `TorChatDb.Dispose()` writes a timestamp to the `server_meta` table. On the next startup, `ReplayCache` reads this timestamp and rejects messages with `ClientSentAtUnixMs < (lastShutdown + 120s)`. This closes the window where in-memory fingerprints are lost during a restart — preventing replays immediately after a reboot.

**Browser —** `(senderId, nonce)` **Map** (`replay-guard.js`)

After a successful decrypt of a **fresh** live packet (or any history packet), the client remembers `senderId|nonce` for the **whole room session** (`reset()` on join / lobby only). Packet `t0` is not used for retention. Hard cap **5000**: the oldest `seenAt` is evicted unless every entry was seen within **7 min** (flood → live packets fail closed). Live packets must be within **5 minutes** past / **2 minutes** future of `Date.now()`. **History sync skips the clock window** (JoinRoom can replay up to 500 messages, including TTL-old envelopes). Reconnect does not clear the map, so the same history is not duplicated unless a malicious relay evicts entries (open rooms). Own `sendEncrypted` nonces are remembered so a round-trip echo is dropped. Stale, far-future, and cache-full drops share one system line per 30 s (`[ignored N stale or replayed messages]` / `[ignored N replay cache full]`). This is defence in depth, not a substitute for hub `PacketValidator`.

---

### Admin Token Security

Admin HTTP APIs are protected by a cryptographically generated token:

- **Generated by Server Manager** on each ChatServer start (`ChatServerProcessHost`): 32 random bytes → Base64 (~44 chars)
- Passed to ChatServer via the `TORCHAT_ADMIN_TOKEN` environment variable
- ChatServer reads the token at startup; if missing or shorter than 16 characters, admin APIs return 401
- Requests must send header `X-TorChat-Admin`; comparison uses constant-time equality

| Endpoint | Purpose |
|----------|---------|
| `GET /api/rooms` | Manager room list (metadata, including hidden) |
| `POST /api/rooms` | Create open public room (`roomId` + operator `creatorSecret`) |
| `DELETE /api/rooms/{id}` | Wipe room + ciphertext |
| `PUT /api/admin/room-policy` | Live website CreateRoom disable + public-room cap |
| `POST /api/admin/cors-onion?onion=` | Lock live CORS allowlist to HS hostname |
| `GET /api/health` | Manager readiness gate |
| `PUT /api/admin/feed-policy` | Feed enable + retention (Manager Feed settings) |
| `GET /api/admin/feed/moderation` | Feed moderation list (plaintext previews) |
| `GET /api/admin/feed/posts/{id}`, `GET /api/admin/feed/comments/{id}` | Full Feed body for moderation |
| `DELETE /api/admin/feed/posts/{id}`, `DELETE /api/admin/feed/comments/{id}` | Host removes any Feed post or comment |

Tor Browser clients cannot call these — only Server Manager (with the env token) can. Hub `CreateRoom` from the onion is separate and follows the website policy.

**Separate admin port.** Server Manager reserves a second free loopback port and passes it as `TORCHAT_ADMIN_PORT`; the Hidden Service publishes only port 80 and the chat port, never this one. On the Tor-facing port the admin routes above return **404**, so they do not even appear to exist over the `.onion` — a bare 401 would otherwise fingerprint the server and expose the token to guessing. IP filtering cannot do this job: the local `tor.exe` forwards every visitor from `127.0.0.1`, so the listening port is the only thing that distinguishes Manager from a Tor client. If `TORCHAT_ADMIN_PORT` is unset (plain `dotnet run`, integration tests), admin routes answer on every bound port as before.

---

### Transport and Browser Security

| Control | Implementation |
|---------|---------------|
| Content Security Policy | ChatServer: `default-src 'none'` plus explicit `'self'` / `blob:` / `'none'` (`img-src` has `blob:` for attachments; no `data:`). `form-action 'none'`. `require-trusted-types-for 'script'` and `trusted-types 'none'` (no-op on browsers without Trusted Types). ClientHost: same, with `connect-src 'self' http://<v3-onion> ws://<v3-onion>` for the selected relay |
| Subresource Integrity | `integrity=sha256-...` on SignalR + app scripts — browser-enforced |
| Security Headers | `X-Frame-Options: DENY`, nosniff, Referrer-Policy no-referrer, COOP/CORP `same-origin`, COEP `require-corp`, `X-DNS-Prefetch-Control: off`, `X-Robots-Tag: noindex, nofollow` |
| Permissions-Policy | Broad disable list; `clipboard-write=(self)` kept for safety-number / invite copy |
| Cache Control | `Cache-Control: no-store` on HTML, `/js/*`, `/api/*`, `/hubs/*`; `/css/*` and `/fonts/*` `immutable` (already versioned with `?v=` in `index.html`) |
| Host fingerprint headers | `Date` coarsened to whole minutes; hub `SentAt`/`ExpireAt`/`CreatedAt` and client AAD `ClientSentAt` also UTC-minute; `ETag` / `Last-Modified` stripped from static files and the `index.html` fallback |
| AllowedHosts | `127.0.0.1;localhost;*.onion` |
| CORS | Loopback always; `.onion` via `CorsOnionAllowlist` (`TORCHAT_ALLOWED_ONION` and/or live admin POST). Empty allowlist denies cross-origin `.onion` until locked. Same-origin onion HTTP (SSE / long polling) does not need CORS. No `AllowCredentials` |
| Clearnet / Web Crypto gate | Fail-closed overlay: v3 `.onion` or loopback only; `isSecureContext` + `crypto.subtle` (SHA-256 probe) before SignalR. Reverse proxies that rewrite HTML and drop SRI are out of scope |
| Request size limits | Kestrel / SignalR ≈ **4 MiB**; `ClientTimeoutInterval` **5 min** (large Tor image frames block pings mid-transfer) |
| Server Header | Removed (`AddServerHeader = false`) |
| DOM Safety | All user content via `textContent` / `createElement` — no `innerHTML`. Trusted Types deny string HTML sinks where supported. Download names strip bidi/invisible overrides. Leave / ForceLeave clears `#messages`, drafts, safety-number text, and blob image URLs; this is not a forensic memory wipe and not account logout |
| Image metadata strip | Before FileSeal encrypt: JPEG allowlist + cut at EOI (drop trailer); PNG allowlist (drop unknown e.g. `caBX`); GIF comments; WebP EXIF/XMP + VP8X bits. Canvas only if over size budget |
| No Third Parties | No analytics, CDN resources, or WebRTC. Creator recovery code defaults to `sessionStorage` (not an E2EE secret); persist is opt-in |
| WebSocket Origin | Hub Upgrade whose `Origin` is not loopback (`127.0.0.1`, `[::1]`, `localhost`) or an allowlisted `.onion` is 403. Empty allowlist 403s `.onion` Upgrade (warn ≤1/min, Origin not logged); same-origin SignalR still works over SSE / long polling |
| Message cap | Max 500 ciphertext rows per room (oldest trimmed). The **pinned** row is kept until TTL/wipe |
| SRI check | `tools/verify-sri.ps1` — exit 1 on hash mismatch (wwwroot vs `index.html`; not independent of HTML) |
| Trusted client verify | `tools/verify-trusted-client.ps1` / `gh attestation verify` on a `trusted-client-v*` ClientHost artifact before running it |

---

### Rate Limiting

Limits are enforced **per ConnectionId** unless marked server-wide. IP-based limits are intentionally not used because all Tor traffic arrives from `127.0.0.1`.

| Endpoint | Limit |
|----------|-------|
| Hub negotiate | 30 / min |
| JoinRoom | 20 / min |
| CreateRoom | 5 / min |
| SendEncrypted | 60 / min |
| GetRooms | 30 / min |
| GetMyRooms | 20 / min |
| GetMyRooms (`IncludePublic: true`) | 5 / min |
| RebindCreatorSecret | 5 / min |
| AttachBegin | 10 / min |
| Attach Complete / Cancel | No hub limiter (HTTP chunk PUT/GET still 240 / min) |
| Invite manage (status / enable / close / rotate / send interval / attachments / max members / read-only / lock / delete / **pin / unpin**) | 20 / min |
| `/api/health`, `/api/invite/*`, admin APIs | 30 / min |
| `/api/attachments/*` chunk PUT/GET | 240 / min |
| Global HTTP | ~600 / min |
| `/api/invite/*` server-wide | 120 / min (in addition to the per-connection limit above) |

The `/api/invite` ceiling is **not** per connection: a per-connection partition resets whenever a client opens a new circuit, so it cannot bound the total cost of invite-token enumeration. 120 / min is generous on purpose — honest use is one lookup per guest.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend | ASP.NET Core 9, SignalR |
| Frontend | Vanilla HTML/CSS/JS, WebCrypto API (served from ChatServer; optional ClientHost embed) |
| Trusted client (optional) | `TorChat.ClientHost` (.NET 10 loopback host) |
| Database | SQLite + SQLCipher (AES-256) |
| Anonymity | Tor Hidden Service (tor.exe bundled) |
| GUI | Avalonia 12 (.NET 9); WinForms source still in `ServerManager/` |
| Key Storage | Windows DPAPI (desktop); env / key file (Linux/Docker) |
| Real-time | SignalR WebSockets |
| Docker host | `docker/` compose: ChatServer + Tor (no host ports) |

---

## Repository layout

Authoritative map: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md). Compact tree:

```
TorChat/
├── ChatServer/                 # net9 — Hubs/, Validation/, Services/, Security/, wwwroot/
│   └── wwwroot/js/             # crypto, file-seal, secure-boot; app/ (replay-guard, key-material, hub, join, send)
├── ChatServer.Tests/
├── ServerManager/              # net9-windows WinForms source (not the shipped exe)
├── ServerManager.Tests/        # Windows
├── ServerManager.Core/         # shared manager logic
├── Multiplatform servermanager/TorChat/  # Avalonia operator UI → TorChat Server Manager.exe
├── TorChat.ClientHost/         # net10 loopback UI
├── TorChat.ClientHost.Tests/
├── TorConnection/              # tor-expert-bundle.pin.json; TorBundle/; Vanguards/; Python pin
├── docker/                     # in-net chat 8080, admin 8081; torrc → 8080 only
├── tests/                      # Node: file-seal, file-attach, secure-boot, leave-wipe,
│                               # safety-number, hybrid-history, app-format, trusted-client,
│                               # replay-guard, key-hygiene, dom-sinks, storage-hygiene, feed
├── tools/                      # verify-sri.ps1, verify-crypto/, diy-security-check.ps1, release/
├── docs/
├── AGENTS.md
└── create_release.bat          # Windows Avalonia single-file TorChat Server Manager.exe
```

---

## Requirements

**Desktop (Avalonia Server Manager — recommended)**

- Windows 10 or later (64-bit). Linux / macOS packages: [tools/release/RELEASE_README.md](tools/release/RELEASE_README.md).
- [.NET 9 Runtime](https://dotnet.microsoft.com/en-us/download/dotnet/9.0) (for ChatServer; the Manager exe is self-contained)
- [Tor Browser](https://www.torproject.org/) (open the `.onion` UI; includes the HS PoW client)
- Tor Expert Bundle at `TorConnection/TorBundle/tor/tor.exe`. Server Manager **refuses** a binary that does not match `TorConnection/tor-expert-bundle.pin.json`. It does **not** download Tor (no clearnet fetch at HS start).
- Full Vanguards in a binary release uses the bundled `TorConnection/Python/<rid>/python.exe` (no system Python). From source, PATH Python 3.10+ and `stem` still work.

| Pin (Windows x86_64) | Value |
|----------------------|--------|
| Tor version | **0.4.9.13** (Expert Bundle `15.0.24`) |
| Archive | `tor-expert-bundle-windows-x86_64-15.0.24.tar.gz` |
| Archive SHA-256 | `e9dc6ccc93cd6afa507193f4de284d6424233ff5102155cd2c94b259e8a22b65` ([sha256sums-signed-build.txt](https://dist.torproject.org/torbrowser/15.0.24/sha256sums-signed-build.txt)) |
| `tor.exe` SHA-256 | `90bbdcafd586feea608a5e9b7d3959f4ee194f7770755cfde8fab240e9773ad1` |
| Official download | [torproject.org/download/tor](https://www.torproject.org/download/tor/) |
| Verify | GPG on SUMS + archive (key `EF6E286DDA85EA2A4BA7DE684E2C6E8793298290`) — [Tor verifying](https://support.torproject.org/little-t-tor/getting-started/verifying/) |

Setup steps: [TorConnection/README_TOR_BUNDLE.txt](TorConnection/README_TOR_BUNDLE.txt). Intro DoS + PoW need Tor ≥ 0.4.8 (this pin is 0.4.9.13).

- Server Manager is self-contained — no separate .NET installation required

**Docker (Linux host / VPS)**

- Docker Engine + Compose v2
- See [docker/README.md](docker/README.md)

---

## Getting Started

### Windows — Server Manager

1. Download the latest release from [Releases](../../releases)
2. Extract to a folder of your choice
3. Launch `TorChat Server Manager.exe` (Avalonia; self-contained)
4. Set **Project Path** to the extracted folder
5. Click **Start** — ChatServer launches (Production; wait until status shows Running)
6. Click **Tor connection ON** — requires ChatServer first; your `.onion` address appears
7. Share the `.onion` address with your contacts
8. Open `http://YOURADDRESS.onion/` in **Tor Browser** (Go Site copies it)

**Public rooms (Manager):** Create room (open, name only). **Maximum Number of Rooms** caps website public creates; **Disable Room Creation** blocks all onion `CreateRoom` immediately (Manager create still works). **Admin key** is the operator recovery (not in `config.ini`): copy into onion **Profile → Restore** for Settings on those rooms. The **i** button explains Restore. **Save** writes `config.ini` (cap, disable, startup, Tor, ports). **Start server on startup** + Save registers Windows logon (HKCU Run value `TorChat Server Manager MP`); uncheck + Save removes it.

> First Tor bootstrap may take 30–60 seconds depending on network conditions.  
> Free binary release does not ship `config.ini` — Server Manager creates it on Save. Operator data is `%AppData%\TorChat_ServerManager_MP` (Admin key: OS keyring, else `operator_creator.key`; not WinForms DPAPI).

### Docker — ChatServer + Tor

For headless / VPS hosting without Server Manager. ChatServer is **not** published to the host; only the Tor Hidden Service is reachable. Open the `.onion` in Tor Browser.

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

What the relay cannot see in password rooms **when the browser client is unmodified**:

- Message plaintext or the room key (server-blind E2EE under this client trust assumption)
- Client IP addresses (all Tor traffic arrives from loopback to ChatServer)
- User accounts (none — display names only)

What the server operator **can** see:

- Room names and display names (metadata)
- When connections are made and dropped
- **Open-room** ciphertext **and** plaintext capability — key = `f(roomId)`; treat open rooms as public channels
- Ciphertext envelopes and timing for all rooms (blind relay + SQLCipher storage)

This does not claim E2EE against a malicious client publisher, compromised build/signing identity, or compromised endpoint. A web page and its SRI hashes delivered by the relay cannot authenticate themselves; use the separately attested local client.

---

## Source Code

The binary release is **free to use**. If you want to study, audit, or build upon the source code, it is available for purchase:

---

> ### [Purchase Source Code — $49](https://www.dev-offcode.com/TorChatPage.html)
>
> Includes the complete C# source (ChatServer + ServerManager + TorChat.ClientHost), JavaScript crypto layer, build scripts, and personal/educational use license.

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

Only rooms protected with a room password are end-to-end encrypted against the server. In open rooms without a password, the encryption key is derived from the room ID, so anyone who knows the room ID - including the Host - can decrypt those messages. Metadata such as room names, display names, participant identifiers, and message timing is visible to the Host.

The optional Feed is public to anyone who knows the instance address and is not end-to-end encrypted. The instance administrator (Host) can read and remove all Feed content and is responsible for moderating it.

### Eligibility (18+)

You must be at least 18 years old to host, join, or otherwise use OffCode Tor Chat. By using the software, you confirm that you meet this requirement.

### No author access or telemetry

The software does not send telemetry, analytics, logs, messages, or keys to the author(s). The author(s) do not operate any instance, cannot access, monitor, moderate, delete, or disclose content on any instance, and hold no user data. Any request concerning an instance, its content, or its users (including requests from authorities) must be directed to the Host of that instance.

### Governing law and jurisdiction

These terms are governed by the laws of Finland, excluding its conflict-of-law rules. Disputes shall be resolved by a competent district court in Finland. This does not deprive a consumer of the protection afforded by the mandatory provisions of the law of their country of habitual residence, or of the right to bring proceedings in the courts of that country.

### Mandatory rights & severability

Nothing in these terms limits or excludes liability for intentional misconduct or gross negligence, or any mandatory consumer rights, where applicable law does not permit such limitation or exclusion. If any provision of these terms is held invalid or unenforceable, the remaining provisions remain in full force and effect, and the affected provision shall be applied to the maximum extent permitted by law.

### Changes to these terms

New versions of the software may include updated terms. Continued use of a new version constitutes acceptance of the terms included with that version.

### Not legal advice

These terms are a general disclaimer and license condition, not legal advice. Laws differ by country. If you are unsure whether your intended use is lawful, consult a qualified attorney before proceeding.

### Acceptance

By downloading, installing, hosting, joining, or continuing to use OffCode Tor Chat, you accept these Terms of Service and Legal Disclaimer. If you do not agree, do not use the software and stop any running instance under your control.

---

## License

The **compiled binary** is free for personal and commercial use.  
The **source code** is available at [dev-offcode.com](https://www.dev-offcode.com/TorChatPage.html) ($49, personal developer license — ChatServer + Server Manager + TorChat.ClientHost).  
Commercial redistribution / resale of source requires a separate license — contact via the website.

---

*Built for privacy. Questions or commercial inquiries: [dev-offcode.com](https://www.dev-offcode.com/TorChatPage.html)*

---

## Changelog

Newest first. Product narrative lives in the sections above; this list tracks shipped source changes.

### 2026-10-19 — MANAGER + CLIENT (Tor start, orphan cleanup, UI, boot gate)

- **Reset guards** — Connection tab button (Tor and Vanguards must be stopped). Wipes Tor L1 `state` (+ `.bak`); optional checkbox also wipes Vanguards L2/L3. `.onion` and HS keys stay. On a guard-circuit failure the Manager offers the same reset once per hour (`GuardsResetGate`). Logs state the wipe; no restart.
- **Cancellable Tor start** — turning Tor OFF, Stop, or closing the window while Tor is starting cancels it (`StackLifecycle`): reverse kill Vanguards → tor.exe → ChatServer, start task joined ≤ 15 s. A 300 s watchdog cancels a stalled start (`⏱ Tor ei valmistunut …`) and returns the UI to Stopped. Stop works during the ChatServer health wait; auto-restart does not revive it.
- **Orphan process guard** — children are recorded in `TorChat_ServerManager_MP\pids.json`; Windows children are tied to a Job Object (`KILL_ON_JOB_CLOSE`); tor.exe receives `__OwningControllerProcess`; shutdown hooks cover ProcessExit, unhandled exceptions, SIGTERM/SIGINT/SIGHUP/SIGQUIT. On the next start a sweep kills a recorded process only if its PID start time, executable path and bundled root all match. Tor Browser's `tor.exe` and other `dotnet` processes are never touched.
- **Port-in-use message** — if ControlPort 27551 or the chat port is held by another process, the log and dialog show PID and process name before anything starts. Vanguards is started with `--retry_limit 30`.
- **Tor log** — HS descriptor publish is detected; rend/intro chatter is filtered from the Manager log. The guard-failure event fires once per Tor run.
- **Avalonia UI** — minimum window 900×600; caption "TorChat Server Manager" with onion icon; Connection tab collapses Manager settings under an expander; tab styling refreshed.
- **Boot gate** — redesigned fail-closed screen: tagline, animated boot-code panel (`boot-code.css`), `boot-gate.css`, and a `<noscript>` note on setting Tor Browser Security Level to Standard or Safer. Still blocks unless loopback or a v3 `.onion` with `crypto.subtle`.

### 2026-10-17 — MANAGER (Avalonia Windows release)

- **Shipped GUI** — `create_release.bat` publishes Avalonia as `TorChat Server Manager.exe` (win-x64 single-file, copper onion icon). WinForms `ServerManager/` stays in the source tree. Operator data: `%AppData%\TorChat_ServerManager_MP` (not the WinForms folder).
- **Dialogs** — feed / room / Tor confirms await `ConfirmAsync` / `ShowNoticeAsync` (`ShowDialog`). Sync `DialogService.Pump` is gone so Remove selected post no longer hangs the UI.
- **crash.log** — unhandled exceptions append type/message/stack under `TorChat_ServerManager_MP` (Unix 0600). No admin token, keys, or Feed bodies.
- **Full Vanguards Python** — Windows release bundles pinned `Python\win-x64\python.exe` + stem (`python-runtime.win-x64.pin.json`). No system Python required for the checkbox.
- **Release scan** — fail-closed check is `tools/release/create_release.ps1 -ScanOnly` + `forbidden-files.txt` (also `operator_creator.`*, `terms_accepted.sha256`).

### 2026-10-16 — CLIENT (Feed recovery UX)

- **Generate code** — a new visitor can create the recovery code in Profile without creating a room (`ensureCreatorSecret`, no `RebindCreatorSecret`).

### 2026-10-15 — FEED (optional public plaintext stream)

- **Feed** — optional tab (default **off**, Manager **Enable feed**). Public text posts/comments/likes. **Not E2EE**; SQLCipher at rest. Separate SignalR hub `/hubs/feed` (16 KiB cap). Recovery code required to write; reading needs none. Host moderates from Manager (`FeedPostListBox1` / **Remove selected post**). Emoji and invisible/bidi characters are rejected. 
- **Legal** — Terms now state that Feed is public and Host-moderated (re-accept on next Manager start).

### 2026-10-12 — CLIENT + SERVER (storage migrate + WS fallback)

- **Storage** — leftover `localStorage` recovery is copied to `sessionStorage` only if `setItem` succeeds (same as manage/invite tokens). Migration waits until `TorChatToast` is loaded. English toast; Profile keeps a notice until **Remember on this device** or Copy.
- **WebSocket** — empty onion allowlist **403**s `.onion` Upgrade only (warning at most once per minute, Origin not logged). Same-origin SignalR still connects over SSE / long polling (slower). Loopback including `[::1]` stays allowed.

### 2026-10-12 — CLIENT + SERVER (CSP / XSS hardening)

- **CSP** — enforced `default-src 'none'` with explicit `'self'` / `blob:` / `'none'`. Noise texture moved to `/css/noise.svg` so `img-src` no longer needs `data:`. `form-action 'none'`. Trusted Types: `require-trusted-types-for 'script'; trusted-types 'none'`.
- **Headers** — COEP `require-corp`, expanded Permissions-Policy (`clipboard-write=(self)` kept), `X-DNS-Prefetch-Control: off`, `X-Robots-Tag: noindex, nofollow`. `Cache-Control: no-store` on `/api/`* and `/hubs/`*. ClientHost CSP matches (plus onion `connect-src`).
- **Storage** — creator / manage / invite tokens default to `sessionStorage`. Profile **Remember on this device** opts into `localStorage`. Leftover local recovery (no persist flag): see **storage migrate** above.
- **Downloads** — `sanitizeFileName` strips bidi and invisible overrides (U+202E and related).
- **WebSocket Origin** — Upgrade requests use the same loopback / onion allowlist as CORS (403 otherwise). Empty-list fallback: see **WS fallback** above.

### 2026-10-12 — CLIENT (replay-guard retention)

- **Session-lifetime Map** — `(senderId, nonce)` entries last the room session (`reset()` on join / lobby). No 7 min prune on packet `t0` (a future timestamp cannot pin the cache). Cap **5000**: evict oldest `seenAt` unless every entry was seen within **7 min** (flood → `'full'`). Freshness is checked **before** `remember`; stale / >2 min future live packets are not stored.
- **Coalesced ignore notices** — stale/future and cache-full each keep one system row per 30 s (`textContent` update): `[ignored N stale or replayed messages]`, `[ignored N replay cache full]`. Nested `beginHistory` depth unchanged. Reconnect keeps the map (history not duplicated unless a relay evicts entries in an open room).

### 2026-10-11 — CLIENT + MANAGER

- **Client replay guard** — first ship of `replay-guard.js` (`(senderId, nonce)` Map, live 5 min / +2 min, history skip). Retention and notices: **2026-10-12** above.
- **Room-key hygiene** — `rawRoomKeyBytes` removed from `App.state`. Salt copied into a `key-material.js` closure (`setRoomSalt` / `deriveHybrid` / `zeroRoomKeyMaterial`). Callers `fill(0)` the source buffer after copy. Tests: `tests/key-hygiene`.
- **GetInviteStatus noise** — guests do not call the hub without this room’s manage token or owned-room cache. Server Manager folds SignalR `HubException` stacks to one line (`HubException GetInviteStatus: Cannot manage invite.`).
- **Vanguards state path** — `--state` is `%AppData%\TorChat_ServerManager\TorData\vanguards.state`. Legacy cwd / addon-folder copies are migrated once then wiped. Release packaging still refuses to ship `vanguards.state`.
- **Generate new address** — blocked while Vanguards or Tor is running (English dialogs: stop Vanguards, then Tor). Confirm, then CSPRNG-overwrite HS keys (keeps `authorized_clients`), `vanguards.state` (+ `.bak`), and Tor L1 `TorData\state`. Does **not** autostart Tor or change the Vanguard checkbox. Admin key and SQLCipher are separate (**Remove SQL data**).
- **Vanguards start / stop** — addon starts when ControlPort `127.0.0.1:27551` accepts connections (not Bootstrapped 100%). Uncheck Tor: stop Vanguards first. Check Vanguard while Tor is already up without ControlPort: ask to restart Tor. Uncheck Vanguard: stop the addon only.

### 2026-10-11 — CRYPTO (protocol v3, breaking)

- **PBKDF2 domains** `*-v3` — join / locked / open salts `torchat-join-v3`, `torchat-e2ee-v3`, `torchat-open-material-v3` / `torchat-open-v3`. Still 600_000 HMAC-SHA256. SQLCipher `protocol_epoch=3` wipes prior rooms and ciphertext on first start. Spec: [docs/PROTOCOL.md](docs/PROTOCOL.md) (not an audit).
- **NFC only** — passwords `normalize('NFC')` then UTF-8. Not RFC 8265 OpaqueString (no Zs mapping, no extra control reject).
- **AAD version + charset** — `encryptionVersion|roomId|senderId|nonce|clientSentAtUnixMs`. Allowlist `pbkdf2-v3` / `ecdh-hybrid-v3`; missing / v2 / gk-wrap fail closed. `clientSentAtUnixMs` is 1–16 digits. Client `senderId` is lowercase hex; hub `RoomId`/`Nonce`/`SenderId`/`Base64` regexes use `\z`.
- **Hybrid HKDF** — info `torchat-ecdh-v3:` sorts **by senderId** and binds each id to that peer’s SPKI (NUL + u16be length). Same key both directions; swapping SPKIs between ids changes OKM. `senderA === senderB` rejected. Independent KATs: `node tools/verify-crypto/run.mjs` (compare-only; `--write` overwrites `docs/test-vectors.json`).
- **Safety number** — still `torchat-safety-v1`; import P-256 SPKI before hash. Ephemeral keys: leave/rejoin rotates the pair; **both clients show the same new number**. TOFU `sessionStorage` then: `Key rotated after rejoin. Compare again.` Receive path fail-closes junk SPKI (`registerPeerEcdh` imports before `peerPubs.set`).
- **FileSeal v3** — wrap AAD `torchat-fileseal-wrap-v3|{roomId}|{transferId}`; chunk AAD includes `chunkCount`; HKDF labels `*-v3`. `roomId` / `transferId` charset-checked. Manifest `v=3`; receive wires `_sealManifest` (not raw JSON). Chunk caps 128 / 32 MiB / 256 KiB+16. KATs: `docs/test-vectors-fileseal.json`.
- **Image strip** — JPEG allowlist + stop at EOI (no aCropalypse trailer); PNG allowlist (drop `tEXt` / unknown `caBX`); WebP VP8X EXIF/XMP bits cleared. Inline images: mime + magic bytes.
- **Not claimed** — member-to-member sender authentication in password rooms (3+) or open rooms (shared `roomKey`). Argon2 is not used (WebCrypto PBKDF2 only).

### 2026-10-10 — SECURITY

- **AES-GCM IV 12 bytes** — decrypt and hub require a 96-bit IV (NIST SP 800-38D §5.2.1.1; .NET `AesGcm.NonceByteSizes`). Encrypt already used 12 random bytes.
- **gk-wrap-v1 rejected** — hub no longer live-relays obsolete group-key wraps (`Message rejected.`). Stock clients already ignored them. Open rooms stay `pbkdf2-v2` (`key = f(roomId)`).
- **Password encoding v2** — PBKDF2 imports UTF-8 as the browser typed it (no NFC). RFC 8265 OpaqueString NFC would change keys (Finnish `ä` NFC vs NFD) and needs a future v3 domain. OWASP: accept full Unicode; do not reduce entropy.
- **Safety number encoding** — 8×5 groups are least-significant-first; last group ≤ 03402. TOFU still uses full `hashHex`.
- **Manager public rooms** — Server Manager Create room (`POST /api/rooms` on the admin port) adds an open public room (name only). Rooms bind the persistent operator recovery (`AdminKeyTextBox1`, DPAPI LocalAppData). Copy into onion Profile **Restore** for Settings. DELETE and list unchanged.
- **Manager operator recovery** — one Base64URL creator secret (same format as the site recovery). Not an extra key type; SHA-256 `room_creators` only. Unprotect failure does not rotate. Not stored in `config.ini`. Copper **i** next to Admin key opens an OK-only info dialog (copy → Profile Restore).
- **Maximum Number of Rooms** — Manager `MaxRoomsTxt` (0–10000) caps public rooms on hub `CreateRoom`. Hard cap 10 000 unchanged. Live `PUT /api/admin/room-policy` + SignalR `OperatorPolicy`.
- **Disable Room Creation** — Manager checkbox blocks all onion `CreateRoom` immediately. Manager `POST /api/rooms` still works.
- **Manager Save config** — `config.ini` also stores public-room policy, **Start server on startup**, and **Tor connection**. Save writes HKCU Run (`TorChat Server Manager`) so logon opens Server Manager; then ChatServer (and Tor if saved on) start without dialogs. Uncheck + Save removes the Run value.
- **Onion page weight** — 3 self-hosted woff2 (was 9); gzip HTML/CSS/JS only (not JSON/SignalR); non-boot scripts `defer` in document order. No CDN, no HSTS, SRI/CSP unchanged.
- **Hybrid history UX** — after reload, stored `ecdh-hybrid-v1` from a prior join shows a previous-session placeholder (fail-closed; no `roomKey` fallback). Display only: AES-GCM / PBKDF2 / P-256 / HKDF unchanged; ECDH private keys stay non-extractable and are not persisted. Tests: `tests/hybrid-history`.
- **Leave transcript DOM** — `returnToLobby` / `performJoin` call `clearRoomUiDom`: revoke `blob:` image URLs, remove `#messages` children, clear draft / safety-number / room label. Not a forensic heap wipe and not account logout; `localStorage` recovery tokens stay (C-4).
- **Safety number display** — 1:1 SAS shows eight 5-digit groups from 128 bits of SHA-256 (`BigInt`; not JS `Number`). AES-GCM / PBKDF2 / P-256 and `sessionStorage` TOFU on `hashHex` are unchanged. Old 3×5 vs new 8×5 fail-closed until both clients upgrade.
- **Intro DoS** — `HiddenServiceEnableIntroDoSDefense` with Tor man-page burst/rate (200 / 25) on honest introduction points (Proposal 305). Complements `HiddenServiceMaxStreams` (rendezvous only).
- **HS PoW** — `HiddenServicePoWDefensesEnabled` (Equi-X v1, Proposal 327) with queue 250 / burst 2500. Dormant until the service is overloaded; clients bid effort (Tor Browser supported).
- **Version gate** — Server Manager parses `tor --version` and writes those keys only for Tor ≥ 0.4.8 (unknown options abort older daemons).
- **Docker** — same torrc keys; Alpine 3.21 image pins `tor=0.4.9.13-r0`.
- **Loopback-only admin port** — Manager reserves a second free loopback port (`TORCHAT_ADMIN_PORT`) that the Hidden Service never publishes; `/api/health`, `/api/rooms` (GET/POST), `DELETE /api/rooms/{id}`, `/api/admin/cors-onion` and `/api/admin/room-policy` return **404** on the Tor-facing port instead of a fingerprinting 401. Docker uses 8081 (never exposed). Unset = legacy all-ports behaviour for `dotnet run` and tests.
- **Host fingerprint headers** — `Date` coarsened to whole minutes (no clock-skew correlation against a candidate clearnet host); `ETag` / `Last-Modified` stripped from static assets and the `index.html` fallback (both leak build mtime). `/css/`* and `/fonts/`* switch to `immutable` caching to keep Tor page loads fast.
- **Invite enumeration ceiling** — server-wide 120 / min limit on `/api/invite` chained into the global limiter, on top of the per-connection limit that resets on every new circuit.
- **JoinRoom timing** — closed / hidden denials now run the same discarded 250k PBKDF2 verification as every other denial, so response time no longer separates an existing room from a missing one (`ChatHubJoinRoomTimingTests`).
- **Tor Expert Bundle pin** — `tor-expert-bundle.pin.json` SHA-256 of official Windows x86_64 `0.4.9.13` / `15.0.24`. Server Manager fail-closes on mismatch; no auto-download; Authenticode is not the trust root.
- **Clock metadata** — hub and room unix-ms floored to UTC minutes; client AAD timestamp floored before encrypt; freshness/replay errors collapsed to `Message rejected.`
- **Boot gate** — UI blocked unless loopback or Tor v3 `.onion` and Web Cryptography (`isSecureContext` + `crypto.subtle`) is available; overlay + `<noscript>` fail-closed.
- **Pinned message** — creator-only; one pin per room; `PinMessage` / `UnpinMessage` (presence + owner tokens) relay stored ciphertext via `PinState`; pointer in `room_pins` (`nonce` + `clientSentAtUnixMs`); no plaintext or pinner identity; TTL CASCADE; 500-cap keeps the pinned row.

### 2026-08-12 — CHANGES

- **CORS fail-closed** — empty `CorsOnionAllowlist` denies every `.onion` origin until Manager (or `TORCHAT_ALLOWED_ONION`) locks the live HS hostname. Loopback remains allowed.
- **FileSeal** — client wraps a random DEK with the room/hybrid `CryptoKey` (AES-GCM + AAD), encrypts file in 256 KiB plaintext chunks, uploads opaque ciphertext via `PUT /api/attachments/{id}/chunks/{n}` with HMAC `X-Transfer-Token`.
- **Manifest v2** — compact envelope (wrap + salts + `downloadToken`) still goes through E2EE `SendEncrypted`; legacy v1 Base64 envelopes remain readable.
- **Limits** — max 32 MiB per file; hub Begin/Complete/Cancel; SQLCipher `attachment_transfers` + disk under data dir; TTL/orphan cleanup with messages.
- **Policy** — `attachments_disabled` still blocks Begin; text-only PayloadB64 gate unchanged for disabled rooms.
- **Generate new code** — Profile button + hub `RebindCreatorSecret(old, new)` rebinds all `room_creators` rows to the new SHA-256 hash and revokes the old code immediately. Restore alone does not invalidate server bindings.
- **Hash model docs** — creator/manage/invite tokens: SHA-256 of high-entropy secrets; join proof: PBKDF2. Weak-restore warning in Profile UI.
- **Invite tokens** — stored as SHA-256 only; plaintext returned once on create/rotate; client caches last link; startup migrates legacy plaintext rows.
- **JoinRoom owner-proof** — server verifies `manageToken` / `creatorSecret` only when joins are closed or a hidden room lacks a valid invite; client sends recovery only when this browser likely owns the room (manage token, owned-room cache from create/My rooms/restore, or My rooms page). Owner bypass logs + caller `SystemNotice`.
- **GetMyRooms IncludePublic** — separate rate limit (5 / min) vs hidden-only My rooms (20 / min).
- **Recovery docs** — Profile + README treat recovery as a capability token (Settings, listing, closed rejoin).
- **Attachments disabled** — text-only PayloadB64 gate tightened to ~16 KiB (blind size gate; not content-type).
- **Restore recovery code** — after saving the code, client calls `GetMyRooms(includePublic: true)` and shows a toast listing Hidden (My rooms) vs Public (Public rooms) room ids (or “none” / empty-server message).
- **API** — `GetMyRoomsRequest.IncludePublic`; `ListRoomsForCreator(secret, hiddenOnly)`.
- **Owner bypass on JoinRoom** — when `joins_closed` (and for hidden rooms, disabled invite), the creator can still join by sending `manageToken` and/or Profile `creatorSecret`. Other members stay blocked. Password `joinProof` still required for locked rooms.
- **Client** — `joinRoom` / reconnect attach ownership proof from localStorage recovery + per-room manage token.
- **API** — `IRoomInviteService.IsOwner`; optional fields on `JoinRoomRequest`.
- **Disable attachments** — creator Room Settings toggle (`attachments_disabled` in SQLCipher). Members get `AttachmentsPolicy` on join / toggle; Attach button gated in `room-attachments.js`. Server enforces a text-only PayloadB64 size cap when disabled (blind relay cannot inspect ciphertext kind).
- **Hub / registry** — `SetAttachmentsDisabled`, `InviteStatusResponse.AttachmentsDisabled`, registry getters/setters, DB migration.
- **Room Settings UI** — wider two-column layout (Membership | Max members; Attachments | Announcement; Slow mode compact row; Invite / Lock / Delete full width); denser hints and spacing.
- **Max members** — creator Room Settings toggle (`max_members` in SQLCipher, 0 = off, else 2–50). Enforced in `RoomPresenceService.Join`; public `"Room is full."`, hidden generic `"Cannot join room."`. Existing members stay if the cap is lowered.
- **Announcement / read-only** — creator Room Settings toggle (`read_only`). Only the creator may `SendEncrypted` / `BeginAttachmentUpload`; owner tokens are extra hub args (not persisted). Members get a disabled composer (`room-readonly.js`).

---


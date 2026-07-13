<div align="center">

# 🧅 OffCode-Tor-Chat-E2EE

**Anonymous - End-to-End Encrypted - Self-Hosted Chat over Tor Hidden Services**

[![Platform](https://img.shields.io/badge/platform-Windows-blue?logo=windows)](https://dotnet.microsoft.com/en-us/download/dotnet/9.0)
[![.NET](https://img.shields.io/badge/.NET-9.0-purple?logo=dotnet)](https://dotnet.microsoft.com/en-us/download/dotnet/9.0)
[![Tor](https://img.shields.io/badge/Tor-Hidden%20Service-.onion-7D4698?logo=torproject)](https://www.torproject.org/)
[![Encryption](https://img.shields.io/badge/encryption-AES--256--GCM-green)](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto)
[![Source](https://img.shields.io/badge/source%20code-$29-orange)](https://www.dev-offcode.com/TorChatPage.html)

*A fully self-hosted, anonymous chat platform where the server operator cannot read your messages — ever.*

</div>

---

## What is TorChat?

TorChat is a **self-hosted anonymous chat server** that runs entirely over the Tor network as a hidden service (`.onion` address). It combines three independent layers of protection:

1. **Tor Network** — hides both the server and users real IP addresses
2. **End-to-End Encryption** — messages are encrypted in the browser before being sent; the server only ever sees ciphertext
3. **Encrypted Storage** — all data at rest is protected by SQLCipher (AES-256)

Anyone who knows the `.onion` address and room credentials can connect using Tor Browser. No accounts, no phone numbers, no personal information required.

---

## Key Features

| Feature | Details |
|---------|---------|
| Tor Hidden Service | Automatic `.onion` address generation and management |
| True E2EE | AES-256-GCM via WebCrypto API — keys never leave the browser |
| Encrypted Database | SQLCipher with DPAPI-protected key (AES-256 at rest) |
| Real-time Chat | ASP.NET Core SignalR WebSockets |
| Room System | Create open or password-protected rooms |
| Message TTL | Auto-expiring messages (30 min / 1 h / 2 h / permanent) |
| Replay Protection | Per-message fingerprint cache, survives server restarts |
| GUI Manager | Windows Forms Server Manager — start/stop/monitor with one click |
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
|  Message plaintext -> AES-GCM encrypt ----+             |
|                         | ciphertext only               |
+-------------------------+---------------------------------+
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

All cryptography runs in the **browser** via the WebCrypto API. The server has zero access to plaintext at any point.

**Key Derivation (PBKDF2-SHA256, 210,000 iterations in browser)**

| Purpose | Input material | Salt |
|---------|----------------|------|
| Join proof (locked rooms) | Room password | `torchat-join-v1:{roomId}` → 32-byte base64 proof sent to server |
| E2EE key (locked rooms) | Room password | `torchat-e2ee-v1:{roomId}` → AES-256-GCM key; never sent to server |
| E2EE key (open rooms) | `torchat-open-material:{roomId}` | `torchat-open-v1:{roomId}` |

- Separate KDF domains prevent join proof material from being reused as the message key
- Server stores a **second** PBKDF2-SHA256 hash (100,000 iterations) of the join proof — not the password and not the E2EE key
- Server-side verification runs off the thread pool (`Task.Run`) so hub threads are not blocked
- Open rooms still use PBKDF2 (not the raw room ID as the AES key); anyone who knows the room ID can derive the key — users are warned via an amber UI notice

**Message Encryption (AES-256-GCM)**

```
ciphertext = AES-GCM(key=roomKey, plaintext=message, AAD="roomId|senderId|nonce")
```

- 12-byte random nonce per message
- 128-bit GCM authentication tag
- AAD (Additional Authenticated Data) binds ciphertext to room, sender and nonce — prevents replay and envelope substitution

---

### Tor Hidden Service

- ChatServer binds exclusively to `127.0.0.1` — no clearnet exposure whatsoever
- `tor.exe` is bundled and managed automatically by Server Manager
- All client IP addresses are hidden — even the server operator cannot identify users

---

### Database Security (SQLCipher)

| Property | Value |
|----------|-------|
| Cipher | AES-256-CBC (SQLCipher) |
| Key storage | Windows DPAPI (`ProtectedData`) |
| Journal mode | WAL — concurrent readers do not block writers |
| Tables | `messages`, `rooms`, `server_meta` |
| At-rest | `.db`, `.db-wal`, `.db-shm` all encrypted |

---

### Replay Attack Protection

**In-memory fingerprint cache**

Every received message is fingerprinted:
```
fingerprint = SHA256( UTF-8 bytes of roomId + "|" + nonce + "|" + payloadB64 )
```
Identical fingerprints within 120 seconds are rejected. Packets older than 120 seconds (or more than 60 seconds in the future) are also rejected.

**Restart-gap protection**

On shutdown, `TorChatDb.Dispose()` writes a timestamp to the `server_meta` table. On the next startup, `ReplayCache` reads this timestamp and rejects messages with `ClientSentAtUnixMs < (lastShutdown + 120s)`. This closes the window where in-memory fingerprints are lost during a restart — preventing replays immediately after a reboot.

---

### Admin Token Security

The room deletion endpoint (`DELETE /api/rooms/{id}`) is protected by a cryptographically generated token:

- **Generated by Server Manager** on each ChatServer start (`ChatServerProcessHost`): 32 random bytes → Base64 (~44 chars)
- Passed to ChatServer via the `TORCHAT_ADMIN_TOKEN` environment variable
- ChatServer reads the token at startup; if missing or shorter than 16 characters, the DELETE endpoint is disabled
- Requests must send header `X-TorChat-Admin`; comparison uses constant-time equality

Tor Browser clients cannot delete rooms — only Server Manager (with the env token) can.

---

### Transport and Browser Security

| Control | Implementation |
|---------|---------------|
| Content Security Policy | `default-src 'self'`, `script-src 'self'`, `style-src 'self'`, `img-src 'self' data:`, `connect-src 'self'`, `font-src 'self'`, `object-src 'none'`, `base-uri 'self'`, `form-action 'self'`, `frame-ancestors 'none'` |
| Subresource Integrity | `integrity=sha256-...` on SignalR + app scripts — browser-enforced |
| Security Headers | `X-Frame-Options: DENY`, nosniff, Referrer-Policy no-referrer, COOP, CORP |
| Permissions-Policy | `geolocation=()`, `microphone=()`, `camera=()` |
| Cache Control | `Cache-Control: no-store` on HTML and `/js/*` |
| AllowedHosts | `127.0.0.1;localhost;*.onion` |
| CORS | localhost and `*.onion` origins only; no `AllowCredentials` |
| Request size limits | Kestrel `MaxRequestBodySize` and SignalR receive limit ≈ 64 KB |
| Server Header | Removed (`AddServerHeader = false`) |
| DOM Safety | All user content via `textContent` / `createElement` — no `innerHTML` |
| No Third Parties | No analytics, CDN resources, WebRTC, or LocalStorage for secrets |
| Message cap | Max 500 ciphertext rows per room (oldest trimmed) |

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
| `/api/*` | 30 / min |
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
| Key Storage | Windows DPAPI |
| Real-time | SignalR WebSockets |

---

## Requirements

- Windows 10 or later (64-bit)
- [.NET 9 Runtime](https://dotnet.microsoft.com/en-us/download/dotnet/9.0) (for ChatServer)
- [Tor Browser](https://www.torproject.org/) (to connect via .onion)
- Server Manager is self-contained — no separate .NET installation required

---

## Getting Started

1. Download the latest release from [Releases](../../releases)
2. Extract to a folder of your choice
3. Launch **`ServerManager.exe`**
4. Set **Project Path** to the extracted folder
5. Click **Start** — ChatServer launches automatically
6. Click **Tor Connection ON** — your `.onion` address appears in the UI
7. Share the `.onion` address with your contacts
8. Open it in **Tor Browser** and create or join a room

> First Tor bootstrap may take 30–60 seconds depending on network conditions.

---

## Privacy Guarantees

What the server operator **cannot** see:
- Message content (end-to-end encrypted before leaving the browser)
- Client IP addresses (all traffic routed through Tor)
- User identities (no accounts or registration)

What the server operator **can** see:
- Room names and display names (metadata)
- When connections are made and dropped
- Open room traffic patterns

---

## Project Structure

```
TorChat/
+-- ServerManager/              WinForms GUI (process and Tor lifecycle)
|   +-- ChatServerProcessHost.cs    Spawns ChatServer, generates admin token
|   +-- TorHiddenServiceSetup.cs    Manages tor.exe hidden service
|
+-- ChatServer/                 ASP.NET Core 9 backend
|   +-- Hubs/ChatHub.cs             SignalR hub: auth, replay, rate limiting
|   +-- Security/ReplayCache.cs     Fingerprint cache + restart-gap protection
|   +-- Services/TorChatDb.cs       SQLCipher DB: WAL mode, shutdown tracking
|   +-- Services/RoomRegistryService.cs  Async PBKDF2 authentication
|   +-- wwwroot/js/
|       +-- crypto.js               WebCrypto E2EE: PBKDF2 + AES-GCM
|       +-- chat-hub.js             SignalR client
|       +-- app.js                  UI logic and room management
|
+-- TorConnection/              Tor Expert Bundle (tor.exe + geoip data)
```

---

## Source Code

The binary release is **free to use**. If you want to study, audit, or build upon the source code, it is available for purchase:

---

> ### [Purchase Source Code — $29](https://www.dev-offcode.com/TorChatPage.html)
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
The **source code** is available at [dev-offcode.com](https://www.dev-offcode.com/TorChatPage.html) ($29, personal developer license).  
Commercial redistribution / resale of source requires a separate license — contact via the website.

---

*Built for privacy. Questions or commercial inquiries: [dev-offcode.com](https://www.dev-offcode.com/TorChatPage.html)*

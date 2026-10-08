# Pairiquium Matchmaking Protocol

This document explains how a game client and game backend talk to Pairiquium matchmaking: the order things happen in, and the behaviour the machine-readable specs can't express. The exact message shapes live in the specs:

| Spec | Covers |
|---|---|
| [`openapi.json`](openapi.json) | `POST /v1/tickets` and the match-found webhook |
| [`asyncapi.json`](asyncapi.json) | The WebSocket messages between client and Pairiquium |

The latest versions are also served from pairiquium.com. The files in this repository are versioned and tagged, so you can pin the version your SDK implements.

## Overview

```
Game client ──(1) log in──▶ Your backend ──(2) POST /v1/tickets, X-MaaS-API-Key──▶ Pairiquium
            ◀─(3) { ticket, wsEndpoint } ─────────────────────────────────────────
Game client ──(4) wss://<wsEndpoint>?ticket=<ticket> ──▶ Pairiquium (WebSocket)
            ◀──▶ begin_search … MATCH_FOUND

Pairiquium ──(5) POST <your NotificationWebhook> ──▶ Your backend
```

There are three parties. Pairiquium never sees your players' credentials: your backend authenticates the player however you like, then asks Pairiquium for a ticket on their behalf.

## 1. Getting a ticket

Your backend calls `POST /v1/tickets` with your secret API key in the `X-MaaS-API-Key` header and a `userId` of your choosing. Optionally pass `attrs` (`mmr`, `karma`) for game modes that match on them, and `preferredRegion` to choose a region.

- **The API key is a secret.** Call this endpoint only from your backend, never from a game client.
- A ticket is **single-use**, **expires after 300 seconds**, and is **locked to one region**. Present it to a different region and it is rejected.
- Request a **new ticket for every connection attempt**, including reconnects.
- `userId` is your identifier. It comes back unchanged in the match webhook, which is how you tie a match to a player.

## 2. Connecting

Open a WebSocket to `wsEndpoint` with the ticket in the query string:

```
wss://ws.eu-west-2.pairiquium.com/?ticket=<ticket>
```

Use the `wsEndpoint` returned with the ticket rather than building the URL yourself. If the ticket is missing, expired, already used, or for another region, the connection is refused during the handshake.

Because the ticket is in the URL, avoid logging full connection URLs.

## 3. Messages

Every message is a JSON text frame with an `action` field. Messages from the client put extra data in `payload`; messages from the server put it in `data`.

**Client to server**

| `action` | Purpose |
|---|---|
| `begin_search` | Start searching in a game mode (`payload.gameMode`) |
| `cancel_search` | Stop searching |
| `heartbeat` | Keep an active search alive |
| `pong` | Reply to a server `PING` (`payload.correlationId`, `payload.gameMode`) |

**Server to client**

| `action` | Purpose |
|---|---|
| `MATCH_FOUND` | A match was made. `data.metadata` has `matchId` and `team` |
| `PING` | Latency check. Reply with `pong` |
| `SEARCH_SUPERSEDED` | The same player started a search from a newer connection. This connection is then closed |
| `CLOSE_CONNECTION` | The server is closing the connection |
| `ERROR` | A problem handling a message. `data` has `code`, `message`, optional `details` |

Ignore any fields you don't recognise: new optional fields may be added without a major version bump.

## 4. A search from start to finish

1. Connect (see above).
2. Send `begin_search` with the game mode.
3. **While searching, send `heartbeat` periodically.** A search expires after 3 minutes without one. Sending one every 30 to 60 seconds is a sensible default.
4. If the game mode matches on ping, the server sends `PING` before the search starts. Reply with `pong` straight away, echoing the `correlationId` and the game mode. The server measures round-trip time from its own clock, and gives up on a ping check after 10 seconds.
5. When a match is made, you receive `MATCH_FOUND`.
6. To stop early, send `cancel_search`.

If the same player already has a search in that game mode on a **different connection**, the new connection takes it over. A search that dropped recently resumes with its original search time. If the older connection is still open, it receives `SEARCH_SUPERSEDED` and is closed.

### What `MATCH_FOUND` does not contain

It carries only the `matchId` and the player's `team`. It does **not** say which game server to join. Your backend receives the match webhook with the same `matchId` and is responsible for allocating a server and getting that information to players, typically through your own channel keyed by `matchId`.

## 5. The match webhook

When a match is made, Pairiquium POSTs to the `NotificationWebhook` URL configured on the game mode:

```json
{
  "gameMode": "1v1",
  "matchId": "…",
  "participants": [
    { "UserId": "player-1", "SearchDurationSeconds": 12, "Team": 0, "Mmr": 1500 },
    { "UserId": "player-2", "SearchDurationSeconds": 9,  "Team": 1, "Mmr": 1480 }
  ]
}
```

- Respond with any `2xx` within **5 seconds**. Anything else, or a timeout, counts as a failure and the delivery is retried.
- Delivery is **at-least-once**. De-duplicate on `matchId`.
- The webhook is **not yet signed**, so do not treat a request as authenticated on its own. Request signing is planned.

## 6. Errors

`ERROR` messages carry a `code`. Some are terminal: the server sends the error and then closes the connection. Do not reconnect automatically for these.

| Code | Terminal | Meaning |
|---|---|---|
| `BILLING_ACCESS_DENIED` | yes | Account suspended for billing |
| `TOKEN_QUOTA_EXCEEDED` | yes | Usage quota used up for this billing cycle |
| `WAITLIST_ACCESS_DENIED` | yes | Tenant is still on the signup waitlist |
| `PLAYER_ATTRIBUTE_REQUIRED` | yes | The game mode needs an attribute (for example `mmr`) that the ticket didn't carry. `details.attribute` names it |
| `GAME_MODE_NOT_FOUND` | no | Unknown game mode for this tenant |
| `GAME_MODE_DISABLED` | no | The game mode is not accepting new searches |
| `STALE_PONG` | no | A `pong` didn't match a pending ping check. Retry `begin_search` |
| `INVALID_MESSAGE` | no | The frame wasn't valid JSON or had no valid `action` |
| `INTERNAL_ERROR` | no | Unexpected server error. Safe to retry |

Treat unknown codes as non-terminal unless the connection closes.

## 7. Reconnecting

Connections can drop: networks change, devices sleep, and the gateway limits how long a single connection can live (it may close idle connections after about 10 minutes, and every connection after 2 hours).

- If the connection drops mid-search, the server **holds the search for about 30 seconds**.
- Reconnect with a **new ticket**, since the old one is spent, then send `begin_search` for the same game mode. Within the 30-second window this resumes the held search with its original search time. After that, it starts a fresh search.
- Don't retry in a tight loop. Back off between attempts.

## Versioning

The specs carry a version in `info.version`. Additive changes (new optional fields, new messages you can ignore) are minor or patch releases. Anything that could break an existing client is a major release. Repository tags match the spec version.

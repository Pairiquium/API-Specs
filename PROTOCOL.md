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
| `begin_search` | Start searching in a game mode (`payload.gameMode`). In a party, only the leader may send it and it queues the whole party |
| `cancel_search` | Stop searching (in a party, leader only; it stops the search for everyone) |
| `heartbeat` | Keep an active search alive |
| `pong` | Reply to a server `PING` (`payload.correlationId`, `payload.gameMode`) |
| `create_party`, `join_party`, `leave_party`, `kick_member`, `promote_leader`, `invite_to_party`, `accept_invite`, `decline_invite` | Form and manage a party. See [8. Parties](#8-parties) |

**Server to client**

| `action` | Purpose |
|---|---|
| `SEARCH_STARTED` | Your search is registered and queued. `data` has `gameMode` and `startedAt` (epoch seconds), plus `partyId` for a party search |
| `MATCH_FOUND` | A match was made. `data.metadata` has `matchId` and `team` |
| `PING` | Latency check. Reply with `pong` |
| `SEARCH_SUPERSEDED` | The same player started a search from a newer connection. This connection is then closed |
| `CLOSE_CONNECTION` | The server is closing the connection |
| `ERROR` | A problem handling a message. `data` has `code`, `message`, optional `details` |
| `PARTY_UPDATE`, `PARTY_LEFT`, `PARTY_SEARCH_CANCELLED`, `PARTY_INVITE` | Party state and events. See [8. Parties](#8-parties) |

Ignore any fields you don't recognise: new optional fields may be added without a major version bump.

## 4. A search from start to finish

1. Connect (see above).
2. Send `begin_search` with the game mode.
3. The server acknowledges with `SEARCH_STARTED` once the search is queued. For a ping-based game mode that happens after your `pong` (step 5). Sending `begin_search` again on a search that is already running repeats the acknowledgement, so a client that missed it can recover. If the search couldn't be queued you get `SEARCH_START_FAILED` instead, and nothing is searching: send `begin_search` again to retry.
4. **While searching, send `heartbeat` periodically.** A search expires after 3 minutes without one. Sending one every 30 to 60 seconds is a sensible default.
5. If the game mode matches on ping, the server sends `PING` before the search starts. Reply with `pong` straight away, echoing the `correlationId` and the game mode. The server measures round-trip time from its own clock, and gives up on a ping check after 10 seconds.
6. When a match is made, you receive `MATCH_FOUND`.
7. To stop early, send `cancel_search`.

Players who want to queue together form a party first (see [8. Parties](#8-parties)); the steps above then apply to the party, with the leader sending `begin_search`.

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
    { "UserId": "player-1", "SearchDurationSeconds": 12, "Team": 0, "Mmr": 1500, "PartyId": "…" },
    { "UserId": "player-2", "SearchDurationSeconds": 9,  "Team": 1, "Mmr": 1480 }
  ]
}
```

- `PartyId` is present on players who queued as part of a party: players sharing one queued together. It is absent for solo players. The other fields are as described in [`openapi.json`](openapi.json).
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
| `SEARCH_START_FAILED` | no | The search couldn't be queued and was rolled back. Nothing is searching. Safe to retry |
| `INVALID_MESSAGE` | no | The frame wasn't valid JSON or had no valid `action` |
| `INTERNAL_ERROR` | no | Unexpected server error. Safe to retry |

Party errors (see [8. Parties](#8-parties)) are all non-terminal. Treat unknown codes as non-terminal unless the connection closes.

## 7. Reconnecting

Connections can drop: networks change, devices sleep, and the gateway limits how long a single connection can live (it may close idle connections after about 10 minutes, and every connection after 2 hours).

- If the connection drops mid-search, the server **holds the search for about 30 seconds**.
- Reconnect with a **new ticket**, since the old one is spent, then send `begin_search` for the same game mode. Within the 30-second window this resumes the held search with its original search time. After that, it starts a fresh search.
- Don't retry in a tight loop. Back off between attempts.
- **Parties don't survive a dropped connection.** A player who disconnects is removed from their party straight away, with no 30-second hold, and a party that was searching has its search cancelled. After reconnecting, rejoin with the join code or a new invite. Don't resend `begin_search` for a party search on a new connection: it would queue the player alone.

## 8. Parties

A party lets friends queue together. It is optional, and off by default: the game owner turns it on per game mode. Players form a party over the WebSocket **after connecting and before searching**, so a client that uses parties connects first and only then decides whether to search alone or as a group.

### Forming a party

| Send | Who | Effect |
|---|---|---|
| `create_party` | anyone | Creates a party with you as leader. You receive a `PARTY_UPDATE` containing a `joinCode` |
| `join_party` `{ code }` | anyone | Joins the party with that code |
| `invite_to_party` `{ userId }` | leader | Invites a connected player in your own tenant. They receive `PARTY_INVITE` |
| `accept_invite` `{ partyId }` | invitee | Joins the inviting party, exactly like `join_party` |
| `decline_invite` `{ partyId }` | invitee | Discards the invite. Optional: invites expire after 2 minutes |
| `leave_party` | any member | You receive `PARTY_LEFT`. If you led, the longest-standing member takes over |
| `kick_member` `{ userId }` | leader | The player receives `PARTY_LEFT` with reason `KICKED` |
| `promote_leader` `{ userId }` | leader | Hands leadership to another member |

Share the join code through your own channels (friends list, chat, a lobby UI). A player can be in only one party at a time, and creating or joining a party cancels a search that player had running alone. **All members need tickets for the same region.**

If an invite can't be delivered you get `PARTY_INVITEE_UNAVAILABLE`, which deliberately doesn't say whether the player is offline or just busy.

### Party state

Every member receives `PARTY_UPDATE` whenever the party changes. It always carries the **full** state, so render from it rather than patching:

```json
{ "action": "PARTY_UPDATE", "data": { "party": {
  "partyId": "…", "joinCode": "K7M2QX", "leaderUserId": "alice",
  "members": [{ "userId": "alice" }, { "userId": "bob" }],
  "status": "LOBBY"
} } }
```

`leaderUserId` and `members[].userId` are the `userId` values your backend supplied when minting tickets, so a client has to know its own `userId` to tell whether it is the leader.

`status` is `LOBBY` (forming up), `STARTING` (a ping-based party is measuring latency; `gameMode` is included) or `SEARCHING`. Membership is locked in the last two: nobody can join, and invites are refused.

### Searching as a party

1. The **leader** sends `begin_search`. Everyone else does not. A non-leader gets `NOT_PARTY_LEADER`.
2. Every member (not just the leader) receives `SEARCH_STARTED` with the `partyId`. Each member needs the attributes the game mode matches on (for example `mmr`) in their own ticket.
3. **Every member sends `heartbeat`** while searching, exactly as in [4.](#4-a-search-from-start-to-finish).
4. When a match is made, each member receives `MATCH_FOUND` as usual and the party returns to `LOBBY` on the same connection, ready to queue again. The match webhook marks the players with a shared `PartyId`.

A party is matched as one unit: on its members' average MMR, or for ping-based game modes on its slowest member's ping.

**Ping-based game modes.** Ping needs a reading from every member, so the party's `status` becomes `STARTING` and each member receives a `PING`. Each member replies with `pong`, using the `gameMode` from the party's `PARTY_UPDATE` (a member never sent `begin_search`, so it has no other source for it). When every member has answered, the search is queued and everyone receives `SEARCH_STARTED`. If a member doesn't answer within 10 seconds the check expires and the party goes back to `LOBBY` with no message of its own; the leader should send `begin_search` again.

**Cancelling.** `cancel_search` from the leader stops the search for everyone. If a member leaves, is kicked or disconnects while the party is searching or starting, the search is cancelled for everyone and the remaining members receive `PARTY_SEARCH_CANCELLED`, whose `reason` is `LEADER_CANCELLED`, `MEMBER_LEFT`, `MEMBER_KICKED`, `MEMBER_DISCONNECTED` or `START_FAILED` (the search couldn't be queued). It names the member in `userId` where relevant.

### Team placement

Each game mode has a **keep parties together** setting.

- **On:** a party is always placed on a single team, so a party can be no larger than the biggest team. A match is formed only when every team can be filled exactly, so some combinations can never play on their own. In a 3v3, three parties of two can't fill two teams of three; the queue waits for something like two parties of two plus two solo players. Searches are not relaxed over time.
- **Off:** parties are guaranteed to be in the same match, but their members may be split across teams, including against each other.

### Party errors

All non-terminal; the connection stays open: `ALREADY_IN_PARTY`, `NOT_IN_PARTY`, `PARTY_NOT_FOUND`, `PARTY_MEMBER_NOT_FOUND`, `PARTY_FULL`, `PARTY_SEARCHING`, `NOT_PARTY_LEADER`, `PARTIES_NOT_ENABLED`, `PARTY_TOO_LARGE`, `PARTY_MEMBER_ATTRIBUTE_MISSING` (`details.userId` is the member and `details.attribute` the missing attribute), `PARTY_CHANGED` (the party changed underneath the request: retry), `PARTY_INVITEE_UNAVAILABLE`, `PARTY_INVITE_NOT_FOUND`.

## Versioning

The specs carry a version in `info.version`. Additive changes (new optional fields, new messages you can ignore) are minor or patch releases. Anything that could break an existing client is a major release. Repository tags match the spec version.

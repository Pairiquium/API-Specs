# API-Specs

The public integration specs for the Pairiquium Matchmaking Engine.

| File | What it is |
|---|---|
| [`PROTOCOL.md`](PROTOCOL.md) | Start here. How a game client and backend use Pairiquium, and the behaviour the specs can't express |
| [`openapi.json`](openapi.json) | OpenAPI 3.1: `POST /v1/tickets` and the match-found webhook |
| [`asyncapi.json`](asyncapi.json) | AsyncAPI 3.0: the WebSocket messages between client and Pairiquium |

The latest specs are also served at `https://pairiquium.com/openapi.json` and `https://pairiquium.com/asyncapi.json`.

`openapi.json` and `asyncapi.json` are generated from the platform source and published automatically on each release. Don't edit them by hand. Each spec version is tagged (`v1.0.0`, …), so an SDK can pin the version it implements.

The official Unity SDK is at [Pairiquium/UnitySDK](https://github.com/Pairiquium/UnitySDK).

## License

Apache 2.0, see [LICENSE](LICENSE).

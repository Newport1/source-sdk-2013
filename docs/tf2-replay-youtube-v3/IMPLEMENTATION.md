# Implementation notes: legacy interfaces and v3 mapping

**This is a migration guide, not implemented runtime code.** Consult the current
SDK and API documentation before editing; do not assume that open-source SDK
symbols exactly match Valve's separately distributed game builds.

## Source map

| Source | Current responsibility | Required migration |
| --- | --- | --- |
| `src/game/client/youtubeapi.h` | Public login/upload/stats handles and status enums | Preserve useful asynchronous/cancel/progress API while introducing OAuth and v3 data contracts |
| `src/game/client/youtubeapi.cpp` | ClientLogin; profile `gdata` GET; v2 multipart upload; Atom parsing | HTTPS v3, PKCE, JSON, bounded request buffers, retry/resume and errors |
| `src/game/client/replay/replayyoutubeapi.cpp` | Username/password UI, metadata, upload result, GC/Steam events | Browser consent UX, proper error states, consent/disclosure and preserved game integration |
| `src/game/client/replay/replayyoutubeapi_key_sdk.cpp` | `GetYouTubeAPIKey` v2 developer-key stub | Provision modern OAuth client ID without source secrets; account for non-SDK/Valve builds |
| `src/game/client/replay/vgui/replaybrowserdetailspanel.cpp` | Invokes upload dialog and parses YouTube info response | Use video ID and v3 stats; support legacy persisted URLs |
| `src/game/shared/tf/achievements_tf_replay.cpp` | Tracks view-count-related Replay achievements | Do not alter thresholds or bypass Steam/GC authority |
| `src/game/client/client_base.vpc` | Compiles replay and YouTube files under `BUILD_REPLAY` | Add any new files to build definitions for each supported platform |

Relevant symbols include `YouTube_ShowLoginDialog`,
`YouTube_SetDeveloperSettings`, `YouTube_Login`, `YouTube_Upload`,
`YouTube_GetVideoInfo`, `CYouTubeUploadWaitDialog`,
`CYouTubeResponseHandler`, `IReplayMovie::SetUploadURL`,
`CMsgReplayUploadedToYouTube`, and Steam `PublishVideo`.

## Protocol map

### OAuth

- Use desktop app authorization code + PKCE (`S256`), loopback listener on a
  dynamically allocated port, cryptographically random `state`, and the
  user's default browser. Reject redirects with invalid `state`, port/path,
  or missing code. Implement timeout, cancel and port-in-use errors.
- Google auth URL: `https://accounts.google.com/o/oauth2/v2/auth`.
  Token exchange/refresh: `https://oauth2.googleapis.com/token`.
- Prefer scopes justified by required behavior: `youtube.upload` for upload
  and independently evaluate `youtube.readonly` for channel identity/stats.
  Require `access_type=offline` only if background refresh is needed.
- Treat desktop applications as unable to protect embedded secrets. Persist
  refresh tokens using secure OS facilities when available; do not store them
  in `ConVar`/cfg files or diagnostic logs.

### Upload

1. POST `https://www.googleapis.com/upload/youtube/v3/videos?uploadType=resumable&part=snippet,status`,
   `Authorization: Bearer ...`, a JSON video resource, plus
   `X-Upload-Content-Type`/`X-Upload-Content-Length`.
2. Validate and save the HTTPS `Location` session URI. Treat it as sensitive,
   and do not follow untrusted hosts or disclose bearer tokens in redirects.
3. PUT binary chunks with `Content-Range`, using a fixed memory budget.
   For partial progress or interrupted uploads, query status with
   `Content-Range: bytes */TOTAL`, handle `308` and the server's `Range`,
   and resume from the **server-acknowledged** offset. Chunk sizes (except
   final chunk) must be multiples of 256 KiB. Respect `Retry-After` and
   bounded exponential backoff for appropriate failures.
4. Successful completion returns a JSON video `id`, not an Atom XML URL.
   Preserve thumbnail/preview and GC reporting where appropriate. Do not
   mark a replay uploaded on an error, a partial `308` or a missing ID.

Video metadata must use valid v3 `snippet`/`status` properties. Convert the
old 'Games' text category and comma-separated keywords to a supported
numeric `categoryId` and JSON `tags` array. Escape title/description as
JSON (not XML). Do not implicitly publish test uploads.

### Stats, stored URLs and achievements

- New uploads: store a canonical video ID or reconstructable watch URL;
  request `GET https://www.googleapis.com/youtube/v3/videos?part=statistics,status&id=VIDEO_ID`.
- Existing movie records may hold v2 Atom `self` links in
  `IReplayMovie::GetUploadURL()`. Write a strictly validated parser for
  supported historical paths and watch links; unrecognized strings produce
  a controlled unavailable state. Do not send arbitrary stored URLs to an
  authenticated HTTP client (SSRF/credential-leak risk).
- Do not assume video stats are immediately available or that view count
  is identical across API generations. Coordinate any changed counting and
  achievements with Valve's Steam/GC side.

### Compliance and security gates

- Do not accept Google passwords or store user-provided client secrets.
- Consent UI needs YouTube terms/privacy disclosures and upload content
  compliance confirmation consistent with YouTube's policies.
- Protect authorization tokens, refresh tokens, resumable session URIs,
  request/response logs and failed HTTP redirects.
- BYO API projects are **not** a default per-player deployment strategy:
  YouTube disallows quota sharding and unaudited new projects are subject to
  private-only upload restrictions.

## Implementation review / ownership questions

1. Does the distributed TF2 binary build the same replay code, and is replay
   recording/rendering still enabled in production?
2. Can the existing SteamHTTP request API support required streamed request
   bodies, or is a vetted platform transport necessary for file chunks?
3. What is the SDK vs Valve-internal replacement for `GetYouTubeAPIKey`?
4. Which persisted replay formats and platforms need migration tests?
5. What does the GC currently accept for uploaded video URLs and channel names?
6. Who owns Google Cloud OAuth verification, YouTube policy compliance, and
   quota management?

See [TESTING.md](TESTING.md) for reproducible acceptance criteria.

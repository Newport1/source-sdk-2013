# TF2 Replay: YouTube Data API v3 migration (work in progress)

**Status: design and test plan only.** This branch does **not** restore uploads, authorize
users, grant achievements, or change the shipped TF2 client. No live Google credentials
are included. Keep the PR in draft until code and the acceptance criteria in
[TESTING.md](TESTING.md) are complete.

Related Valve reports:
[Source-1-Games #2277](https://github.com/ValveSoftware/Source-1-Games/issues/2277),
[#4187](https://github.com/ValveSoftware/Source-1-Games/issues/4187),
[#4917](https://github.com/ValveSoftware/Source-1-Games/issues/4917).

## Root cause

`src/game/client/youtubeapi.cpp` still uses the retired
`https://www.google.com/accounts/ClientLogin` password flow, the YouTube
Data API v2 `gdata.youtube.com` and `uploads.gdata.youtube.com` endpoints,
`GoogleLogin` authorization headers, Atom XML upload metadata and XML/Atom
response parsing. The legacy upload request also reads an entire rendered movie
into memory before sending it and uses an HTTP (not HTTPS) upload URL.
Changing only the endpoint will not fix the integration.

## Proposed behavior

1. With no supported OAuth configuration, clearly mark YouTube upload as
   unavailable. Never ask for a Google password or invoke retired endpoints.
   Keep local Replay rendering/export usable.
2. With a provisioned OAuth *desktop* client, pressing Upload opens the system
   browser for a Google OAuth 2.0 authorization-code flow with PKCE (S256),
   random `state`, least-privilege consent and a loopback redirect listener.
   Do not use an embedded browser, out-of-band copy/paste, a service account,
   or the user's Google password.
3. Upload rendered files with YouTube Data API v3 `videos.insert`, starting
   a resumable session and transferring bounded file chunks. Store only
   necessary session progress and avoid logging session URLs, access tokens,
   refresh tokens or sensitive error response bodies.
4. Parse the JSON video ID on successful completion and create an ordinary
   `https://www.youtube.com/watch?v=VIDEO_ID` link. Preserve Replay's
   upload-state persistence, Steam/GC event and Steam Community share
   behavior where independently verified against the existing implementation.
   Never mark a Replay as uploaded until YouTube confirms completion.
5. Retrieve visible video statistics via v3 `videos.list`, using validated
   video IDs rather than stored v2 Atom `self` URLs. Handle missing, private,
   deleted and legacy videos without crashing. Preserve achievement thresholds
   and trust/authority boundaries; a local client must not award itself views.
6. Keep the user's public/private/unlisted selection explicit, provide the
   required YouTube Terms and privacy notices, and require appropriate consent
   and content compliance acknowledgements.

## Credential ownership and deployment

- **Production:** Valve (or the deploying mod) provisions its own authorized
  Google Cloud project and OAuth desktop-client ID **outside the public source
  tree**, plus any YouTube verification, audit and quota approvals. The current
  SDK's `replayyoutubeapi_key_sdk.cpp` v2 developer-key placeholder is **not**
  a usable OAuth client. OAuth desktop client IDs are public identifiers, but
  the distributed code must not embed project credentials, refresh tokens or
  Google account secrets.
- **Development:** Test with a distinct, appropriately authorized development
  project and account. Keep credentials local/untracked. Any optional custom
  client ID feature must be reviewed for compliance, not presented as a means
  for end users to shard a single application's quota among projects.
- **No credentials:** Keep a useful manual-upload/export fallback. A manually
  uploaded file must not silently report Replay upload achievements, because
  it lacks the verified game/GC association.

Google may restrict `videos.insert` uploads from unverified projects created
after July 28, 2020 to **private** visibility regardless of the requested
privacy setting. Do **not** assume a personal/test project can exercise
public or unlisted uploads. Read the policies before implementing a custom
credential mode. Google's terms prohibit spreading one API client/use case
among projects to evade quota limits.

## Milestones

- [x] Map the retired code paths, constraints and acceptance tests.
- [ ] Agree on supported credential provisioning (Valve deployment or
      approved mod) and policy/verification ownership.
- [ ] Replace password login with browser-based OAuth/PKCE and secure token
      lifecycle (storage, refresh, logout, revoke).
- [ ] Implement v3 JSON metadata, resumable upload and recovery from failures.
- [ ] Migrate stats parsing and old stored Replay URLs; check Steam/GC events.
- [ ] Remove/disable all legacy v2 HTTP and ClientLogin code.
- [ ] Complete offline tests, game builds and controlled live verification
      using [TESTING.md](TESTING.md).
- [ ] Document configuration and update this status before upstream review.

Implementation call sites: [IMPLEMENTATION.md](IMPLEMENTATION.md).
Verification matrix: [TESTING.md](TESTING.md).

## Official references

- [OAuth for desktop applications](https://developers.google.com/identity/protocols/oauth2/native-app)
- [YouTube resumable upload](https://developers.google.com/youtube/v3/guides/using_resumable_upload_protocol)
- [`videos.insert`](https://developers.google.com/youtube/v3/docs/videos/insert)
- [`videos.list`](https://developers.google.com/youtube/v3/docs/videos/list)
- [YouTube policies](https://developers.google.com/youtube/terms/developer-policies)
- [Compliance/quota audits](https://developers.google.com/youtube/v3/guides/quota_and_compliance_audits)

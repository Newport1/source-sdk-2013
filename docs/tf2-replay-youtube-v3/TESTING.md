# Replay YouTube v3: test instructions and acceptance matrix

**Status:** Test plan only; no OAuth, upload or stats implementation is
included in the initial PR. The existing SDK uploader still targets retired
v2 endpoints and **must not be used with real Google credentials**.
No Google API traffic should be triggered by automated tests/CI.

## Build prerequisites (current SDK)

From the repository root, use the project's existing instructions:

**Windows:** Install Source SDK 2013 Multiplayer, VS 2022 with
MSVC v143 + Windows SDK and Python 3.13+. In `src`:

```bat
createallprojects.bat
```

Open `src/everything.sln` in Visual Studio and build the relevant TF2
client configuration with `BUILD_REPLAY` enabled.

**Linux:** Install Source SDK 2013 Multiplayer and Podman, then:

```bash
cd src
./buildallprojects
```

Check that the TF2 Replay paths are compiled: `youtubeapi.cpp`,
`replayyoutubeapi.cpp`, `replaybrowserdetailspanel.cpp` and any new files
registered in `client_base.vpc`. A successful core SDK build alone does not
prove that the optional Replay module was built or exercised.

## Local no-credentials verification

After the migration is implemented:

- [ ] Start the TF2 Replay UI with no OAuth client configured; upload is
      clearly unavailable, while local rendering/export still works.
- [ ] No password field, ClientLogin request, `gdata.youtube.com`,
      `uploads.gdata.youtube.com`, plaintext HTTP API traffic or v2
      `GoogleLogin` header remains in executable paths.
- [ ] A missing or invalid config never crashes, spins or leaks an old
      Google password dialog.
- [ ] No API credentials, OAuth tokens or upload session URIs appear in Git,
      config archives, console logs, crash logs or debug output.

Suggested repository search (inventory now; expect **zero live uses**
of retired protocols at completion):

```bash
git grep -n -E 'ClientLogin|GoogleLogin|gdata\.youtube\.com|GData-Version' -- src/game/client
```

Do not delete evidence or tests to make this search pass; replace functionality.

## Offline tests to implement before live uploads

Use controlled fake HTTP responses and an injectable HTTP transport.
These are **required future test cases**, not claims of passing tests.

| Area | Acceptance cases |
| --- | --- |
| PKCE | Cryptographic verifier/challenge, unique state, state mismatch, redirect timeout, loopback conflicts, user denial and cancellation |
| Token handling | Success, expired access token, refresh/revocation failure, reconnect, persisted secret protection and redacted diagnostics |
| Upload metadata | JSON encoding with Unicode, quotes, tags, title length, category ID and all three privacy modes |
| Transfer | Session creation, missing/invalid Location, bounded chunk sizes, `308` with/without Range, partial acknowledgment, network interruption and resumed offset |
| Backoff | `Retry-After`, transient `500/502/503/504`, terminal `400/401/403/404`, quota exhaustion and cancellation |
| Completion | Actual v3 response with `id`, malformed/missing `id`, duplicate callbacks, idempotent completion and cleanup |
| Persistence | Old v2 stored URL, modern watch URL, missing/private/deleted videos, restart after upload |
| Game | Progress/UI updates, GC event, Steam sharing, existing achievement thresholds, no success event on failure |
| Security | No SSRF through historic stored stats URLs, no token leak through redirects/logs, no unapproved external host |

Use synthetic responses only as explicitly labeled **test fixtures**.
Never count mocks as evidence of a successful live upload.

## Manual integration tests (opt-in, after implementation)

1. Provision an authorized **development** Google Cloud project, enable
   YouTube Data API v3 and create an **OAuth desktop** application. Store its
   client ID locally/untracked via the eventual documented deployment path.
   Test on a disposable YouTube channel/account you control.
2. Test sign-in (consent allowed and denied), refresh, sign-out, and restart.
   Confirm OAuth only happens in the external browser.
3. Render a small Replay and upload it as **private**. Verify that the video
   exists in YouTube Studio, with matching title, description, tags, channel,
   category and privacy; confirm the stored replay URL/video ID is correct.
4. Simulate disconnection after upload initiation and during transfer. Confirm
   resumed data is not duplicated and failure does not mark the Replay uploaded.
5. Test an existing legacy record and a deleted/missing video in the Replay
   details pane. Confirm stats errors do not crash the client.
6. Verify Steam/GC upload events and achievement-related behavior only when
   the required production-side trust path can be observed.
7. Test public and unlisted only with an **audited/approved** project where
   YouTube allows those privacy states; a new unverified project may
   force uploaded videos to private.
8. Record OS, platform build, commit SHA, real HTTP status categories
   (without tokens or session URLs), and evidence of results in PR notes.
   Clean up uploaded test videos manually.

No automated test may send, publish, delete or alter a YouTube video.

## Exit criteria for marking this PR ready for upstream review

- [ ] All implementation milestones in [README.md](README.md) complete.
- [ ] Offline unit tests run for state machine, parser, auth and upload logic,
      with actual commands/results recorded.
- [ ] Relevant Windows + Linux Replay builds pass, with logs or reproducible
      build environment documented.
- [ ] Live private-video upload works and demonstrates correct metadata,
      completion, UI state, persistence and resumability.
- [ ] Security/privacy, YouTube policies, API quota and ownership reviewed.
- [ ] Valve dependencies are explicitly listed and no deployment credentials
      or private user data are embedded.

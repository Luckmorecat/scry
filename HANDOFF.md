# Scry — project handoff

Date: 2026-10-07
Repository: https://github.com/Luckmorecat/scry
Local checkout: `~/development/scry`

## Purpose and current state

Scry is a proposed browser-assisted video discovery and download tool. This repository starts with documentation only. No extension, CLI, build configuration, dependencies, or application tests have been implemented.

The next session should turn the discussion into a narrow feasibility prototype, using actual target sites supplied by the user. The architectural recommendations below are proposals, not an approved implementation specification.

## What the user wants

The user proposed two separate applications:

1. A browser extension that discovers videos, presents found sources with metadata, and offers its own player so the user can switch candidates and identify the intended video. If access requires Cloudflare verification, the user opens the website and completes verification in the browser.
2. A CLI that accepts an HLS source selected by the first app and downloads it directly.

The user also wants to discover a complete series catalog: seasons, episodes, and their sources, then select individual episodes or a batch. Desired metadata includes title and duration. Source selection should support multiple streams rather than automatically downloading the first URL.

The latest request authorized creating this project, storing this handoff, connecting the existing empty repository over SSH, and committing/pushing these documents. It did not request implementation yet.

## Prior project and explanation

The discussion began with a read-only inspection of `~/Downloads/vdl`, a Python terminal downloader. Its source is a reference for the existing behavior; it has not been copied into Scry.

An interactive explanation is available at `/private/tmp/vdl-explained.html`, with a saved copy under `~/Documents/Codex/2026-10-07/i-w/outputs/vdl-explained.html`. Reference that artifact and the vdl source for detailed file responsibility and current behavior rather than reproducing its documentation here.

Relevant reference files:

- `~/Downloads/vdl/src/vdl/core/finder.py`: HTTP discovery, Playerjs/Venom catalog adapters, HLS quality/audio parsing, episode resolution.
- `~/Downloads/vdl/src/vdl/core/downloader.py`: ffmpeg subprocess, request headers, progress, partial files and cancellation.
- `~/Downloads/vdl/src/vdl/core/models.py`: existing shared data structures.
- `~/Downloads/vdl/src/vdl/ui/screens/episode_screen.py`: episode and season selection.

vdl uses `httpx`, BeautifulSoup, regex/JSON extraction, and the `m3u8` library. It has no browser engine and executes no page JavaScript. ffmpeg performs the media download and remux with stream copy. Its limitations motivated this proposal: first-candidate selection, one-level iframe inspection, limited request-context propagation, and fallbacks that can present an unvalidated stream as “Original”.

## Research findings

A research subagent investigated the extension approach using official browser, Cloudflare and hls.js documentation. These are documented capabilities and constraints; they have not been tested against the user's target sites.

### Discover in the original website tab

Observe playback in an ordinary user browser tab. This lets the website's own player run its JavaScript and acquire the URLs it actually uses. If challenged, the user completes verification there and presses Play. Treat a challenge as a distinct state rather than “video not found”.

Chrome MV3 supports passive `webRequest` observation; its restrictions on `webRequestBlocking` do not remove observation. Request URLs, status and headers can identify candidates. Monitor URL patterns and HLS response Content-Type, associate candidates with the source tab/frame, preserve signed query parameters, and group master playlists with their variants where possible.

Observation requires host access to both the initiator and requested resource host. `activeTab` alone does not cover arbitrary media CDN hosts. `webRequest` does not expose response bodies; candidate validation or catalog-body extraction needs another mechanism.

Use DOM parsing and player adapters as additional discovery channels. Content scripts share the DOM but normally execute in an isolated world, so they cannot simply read the website's JavaScript globals. MAIN-world injection is a separate adapter choice to evaluate when necessary.

Sources:

- https://developer.chrome.com/docs/extensions/reference/api/webRequest
- https://developer.chrome.com/docs/extensions/develop/concepts/activeTab
- https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts

### Cloudflare and session boundaries

Cloudflare can return a challenge instead of content. Its Challenge Page responses carry `cf-mitigated: challenge`. Human verification in the original tab can help discovery, but it does not establish that extension-origin requests or CLI requests will be accepted.

Cloudflare documents partitioned `cf_clearance` cookies (CHIPS). A clearance acquired in one top-level context is not automatically available in another. Page clearance also does not guarantee access to independently protected playlist or segment hosts. Automated browser challenge solving is not a reliable product foundation; the user handles verification in the normal browser.

Sources:

- https://developers.cloudflare.com/cloudflare-challenges/challenge-types/challenge-pages/detect-response/
- https://developers.cloudflare.com/cloudflare-challenges/concepts/clearance/
- https://developers.cloudflare.com/cloudflare-challenges/reference/supported-browsers/
- https://developers.cloudflare.com/waf/troubleshooting/samesite-cookie-interaction/
- https://developer.chrome.com/docs/extensions/reference/api/cookies

### Independent preview player

An extension page with bundled hls.js can offer source switching and quality/audio selection. Extension-origin fetches with appropriate host permissions can address ordinary cross-origin fetch restrictions. hls.js exposes custom loaders (`loader`, `pLoader`, `fLoader`) for alternate request transport.

This does not make extension requests equivalent to original-tab requests. Cookies, partitioning, referrer/origin checks and authorization can make an independent preview fail while the website player succeeds. A preview failure should not imply the source is invalid. Offer a return to the original source tab as a fallback.

Sources:

- https://developer.chrome.com/docs/extensions/develop/concepts/network-requests
- https://github.com/video-dev/hls.js
- https://hlsjs.video-dev.org/api-docs/hls.js.hlsconfig

### CLI handoff

Keep a bare HLS URL as the simple input path, but consider a structured `DownloadJob` for sources that need more context: exact playlist URL, title, selected quality/audio, required headers, optional domain/path-scoped cookies, and resolution timestamp.

ffmpeg supports Referer, User-Agent, custom headers and scoped cookies. These options do not guarantee portability of browser-bound access. Probe the playlist and a media segment before starting a full download; successful preview is not proof that the CLI can fetch the stream. Consider separate audio access too.

A job-file export is a simple initial bridge. Chrome native messaging is an optional later integration for a Download button and progress feedback. It requires an installed, registered local host; it cannot invoke an arbitrary CLI without that integration.

Session data and signed URLs can be sensitive. Keep exported credentials local and scoped to relevant hosts; use redacted fixtures and diagnostics rather than committing session data.

Sources:

- https://www.ffmpeg.org/ffmpeg-protocols.html
- https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging

## Series catalog proposal

Distinguish catalog discovery from playback resolution. Passive traffic observation sees what is requested; it cannot guarantee discovery of all unplayed episodes. Full lists may live in embedded player configuration, an API response, or data fetched only when a season/episode is selected. Player-specific adapters may be needed.

Suggested domain shape:

```text
Series
  Seasons
    Episodes
      stable ID / source reference
      season and episode numbers
      title / thumbnail
      available source or voice references

ResolvedPlayback
  current HLS URL
  variants / audio groups
  optional duration
  required request context
  resolved timestamp
```

Show the catalog as soon as labels and references are available. Duration can remain unknown until its media playlist is fetched. For VOD, duration can be computed from segment durations; a live playlist's current window is not a full episode duration. Resolve selected episodes near preview/download time because signed URLs can expire. An optional “Inspect all” operation can enrich the list later.

Proposed UI: season tree → checkbox episode list → preview chosen episode → export selected download jobs. Represent unresolved, ready, failed, and verification-needed states explicitly. Preserve actual resolution, bitrate and codec metadata instead of collapsing every variant under coarse labels. Match audio to the selected variant's audio group before choosing a voice/default track.

## Recommended next experiment

Before building a large parser or choosing the full stack:

1. Ask for the intended browser and a few representative target pages. Chromium MV3 was the research baseline; browser choice is not settled.
2. Build the smallest observation prototype: user starts observing a tab, plays the original video, and sees candidate sources with tab/frame provenance.
3. For the same selected source, compare original playback, extension preview, CLI URL-only access, and CLI access with required scoped request context.
4. Include public HLS, signed URLs, cookie/referrer-dependent media, a protected page with public media, and independently protected media if available.
5. Test at least one series player: find where its season/episode catalog lives and determine whether unplayed episodes can be resolved without clicking each one.
6. Record playlist, segment and audio success separately. Classify expiry, missing permissions, authorization failures and preview-context failures.

Completion criterion: evidence shows which target sources support independent preview, direct download, and complete catalog discovery, with explicit unsupported cases. Then choose the first supported adapter and agree the MVP scope with the user.

## Open decisions

- Browser(s), target sites, and relevant player types.
- CLI language and distribution method; Python/ffmpeg is a reference approach, not a selected Scry stack.
- Standalone extension tab versus side panel, and how source-tab fallback should feel.
- Bare URL versus job-file support, credential handling and refresh behavior.
- Whether the first prototype includes a full series adapter or single-video observation first.
- Live streams, encrypted HLS and DRM coverage. Finding HLS alone does not establish downloadable support; DRM has not been designed.

## Suggested skills

Use the session's skill invocation mechanism to read these skills when their tasks apply:

- `prototype`: build a disposable feasibility prototype after selecting target examples.
- `domain-modeling`: clarify catalog/source/playback/job boundaries and record approved terminology or ADRs.
- `show-me`: explain the agreed architecture or UI visually.
- `tdd`: if the user requests test-first development; parser fixtures and transport checks are useful verification candidates.
- `code-review`: review implementation changes against the eventual agreed scope.
- `commit`: group and commit completed work when authorized.

Use the available skill catalog to locate their SKILL.md files; availability may differ in the next session.

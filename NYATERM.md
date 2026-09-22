# NyaTerm fork notes

This branch carries [NyaTerm](https://github.com/nyakang/nyaterm)'s local changes
to `ironrdp-client` and `ironrdp-connector` on top of an unmodified upstream base.

- Fork: <https://github.com/nyakang/IronRDP>
- Upstream: <https://github.com/Devolutions/IronRDP>
- Base revision: `b149f500b85124c513646494335fb6cee525d897`
  (upstream `master` on 2026-09-22)
- Branch: `nyaterm`
- Crates touched: `crates/ironrdp-client`, `crates/ironrdp-connector`,
  `crates/ironrdp-acceptor`, `crates/ironrdp-vmconnect`, `crates/ironrdp-mstsgu`,
  `crates/ironrdp`, and `ffi`.

NyaTerm consumes both through `[patch.crates-io]`. The crate versions this base
publishes are unchanged from the 0.1.0 / 0.10.0 release commit the branch used to
sit on, so those pinned slots still resolve.

## Patches

1. `feat(client): request a full desktop update on demand` — upstream now owns
   `DesktopUpdate` delivery and `RdpClient::with_desktop_updates()`. NyaTerm keeps
   only `RdpInputEvent::RequestFullFrame`, which emits a full-frame
   `DesktopUpdate` through the same ordered output channel for resynchronisation.
2. `refactor(connector): drop the picky dependency and the smart-card path` —
   removes an unused-feature dependency whose defaults pin a prerelease
   `aes-gcm`, and whose `=7.0.0-rc.25` pin also conflicts outright with the
   `=7.0.0-rc.26` that `sspi` pins. **Deliberately drops smart-card
   authentication and redirection**; username/password NLA is unaffected.
3. `feat(client): support authenticated loopback TCP routes` — direct TCP may
   connect through a validated loopback relay and send a bounded authentication
   preamble while retaining the original destination for TLS and CredSSP
   identity. UDP is disabled for relayed connections because the relay contract
   covers TCP only.
4. `build: consume SSPI 0.22` — all direct SSPI dependencies use 0.22, gateway
   picky resolves at rc.26, and fallible `TsRequest::buffer_len` calls propagate
   encoding errors in connector, acceptor, and VMConnect paths.

## Not carried here

Five patches were dropped when this series was rebased from `11a0810c`
(the 0.1.0 / 0.10.0 release commit) onto `master`, 264 commits later, because
upstream now covers what they did:

- `feat(client): report an unsupported resize instead of reconnecting` —
  upstream added `RdpOutputEvent::DisplayResizeFallback(reason)` with three
  reasons, so the reconnect is no longer silent. It still reconnects, which is
  now deliberate: `#271` (the auto-reconnect cookie) is implemented, and the
  resize path explicitly clears the cookie so the fallback establishes a fresh
  session. NyaTerm reports the same `DynamicResizeUnavailable` capability from
  that event instead.
- `feat(client): add a certificate decision hook before CredSSP` — upstream's
  `fix(tls): make certificate validation explicit (#1520)` added
  `CertificateValidation` plus `CertificateValidationCallback`, and
  `feat(mstsgu): apply certificate validation to gateways (#1775)` and
  `fix(rdpeudp)!: thread server_cert_verifier through MultitransportBootstrap`
  extended it to the gateway and multitransport paths the NyaTerm hook never
  reached. Upstream's callback fires only when the platform trust store rejects
  the chain, which for RDP is the common case, and it receives the leaf DER
  rather than a parsed certificate.
- `fix(client): rebuild the FastPath bulk decompressor after reactivation` —
  fixed upstream, and better: `#1474`, `#1255` and `#1518` moved the
  decompressor onto `ActiveStage` and preserve its *history* across
  reactivation, where this patch allocated a fresh one.
- `feat(client): add a builder toggle for audio playback` — upstream forces
  `enable_audio_playback = false` at connect time when the `sound` feature is
  off (`rdp.rs`, `#[cfg(not(feature = "sound"))]`), which is where NyaTerm needs
  it, and adds `with_sound`/`with_audio_mode` for callers that do build sound.
- `fix(connector): remove a stale single_use_lifetimes expectation` — upstream
  keeps the `#[expect]`, and it is a lint-only difference: NyaTerm consumes these
  crates as a git dependency, so cargo caps their lints, and this branch's CI
  does not use `-D warnings`.

## Removed local graphics API

The 2026-09-22 merge drops the fork-only `DesktopReset`, `ImageRegion`,
`ConfigBuilder::with_dirty_region_updates`, and its duplicate framebuffer state.
Upstream's `DesktopUpdate` carries the framebuffer extent and inclusive changed
region, and sends a full update whenever the extent changes.

## Validation

On Windows 11, with the toolchain `rust-toolchain.toml` pins (1.94.1):

```sh
cargo check -p ironrdp-client --features rustls            # clean
cargo check -p ironrdp-client --features rustls,clipboard  # clean
cargo check -p ironrdp-connector                           # clean
cargo check -p ironrdp-acceptor                            # clean
cargo check -p ironrdp-vmconnect                           # clean
cargo test -p ironrdp-connector                            # clean
cargo clippy -p ironrdp-client --features rustls           # no new warnings vs base
```

The merge conflict was limited to `crates/ironrdp-client/src/rdp.rs`. The
resolution retained upstream input batching, keepalive, UDP `ConnectOutput`, and
desktop-damage coalescing while reapplying the relay and full-frame request.

Both client feature sets are checked because `warn` is imported only under
`clipboard` while some always-compiled code uses it.

A real RDP session against a Windows host — connect, resize, clipboard, and
certificate prompt — is not covered by any of the above and remains a manual step.


## Authenticated loopback route for remote-desktop network parity

`ConfigBuilder::with_direct_tcp_proxy` redirects only the direct TCP connection,
then sends a bounded authentication preamble. The original destination continues
to drive TLS and CredSSP identity; the relay address is never substituted there.
Only loopback endpoints are accepted. Authentication bytes are omitted from Debug.
This patch is based on c44dcc77, which also merges upstream gateway authentication
8649c7c6 after NyaTerm's previously pinned 75d9a3b3.
Validation: Windows client build with rustls + clipboard; focused identity test.

# OpenCode 2.1.2 compatibility evidence

Recorded on 2026-10-21. Source baseline: OMO `v3.0.5` (`fc76b4fc`).
This combined verification branch contains the focused pagination,
retained-result attribution, publication-retry and shutdown-mapping changes.
It is not a declaration of full runtime compatibility.

## Established

- The development-only `@opencode/plugin` contract is pinned to **2.1.2**
  (MIT). Runtime OpenCode and OpenTUI dependencies are unchanged.
- Typecheck covers the existing hook and permission assertions plus the
  consumed list input fields, nullable continuation cursor, native execution
  envelope and compaction result names. The mirror intentionally supports
  only the directory/parent list filters its callers use.
- Regression fixtures reject partial list results and newer assistant text
  attributed to an older success. Host-confirmed textless failures and
  interruptions do not require fabricated assistant prose.
- Compaction tests preserve the host-owned `native.messages` window and
  `metadata.keepID`. The adapter does not write the removed credential
  configuration field.
- Failed publication retries survive parent busy/idle transitions without
  enabling periodic wakes. Deletion still discards the retry.
- Shutdown interruption stays raw-only instead of synthesizing terminal idle.
  Native execution and first-notification ownership are not taken over.

Combined checks passed with Bun 1.4.2:

```sh
bun run check:ci
bun run typecheck
bun run build
bun run verify:release
setpriv --bounding-set=-dac_override,-dac_read_search bun test
```

The full suite reported **5,888 passed, 3 existing skips, 0 failed** across
246 files, with six snapshots. Bun was on PATH for subprocess tests. The
process-local capability restriction makes permission-failure tests meaningful
when running as root; no filesystem permissions were relaxed.

The release-artifact check imports packaged entrypoints and uses mocked
clients. It does not start or verify an OpenCode 2.1.2 host. Cache-prefix
properties and snapshots passed, but real provider cache efficiency was not
measured.

## Not established

The machine's additional-server safety policy blocked the isolated test-host
launch command. No alternate launcher or embedded-host workaround was used.
The downloaded test binary and empty test directories were removed.
No live credentials, production session prompts or production restarts were
used for this work.

The following remain runtime gates, not passing checks:

- Actual 2.1.2 startup with the packaged plugin and deferred registration.
- Parallel children, exact-ID continuation and permission/MCP enforcement
  exercised through the real host and a local scripted model provider.
- Graceful and interrupted restart: execution resume, one parent notification,
  board/concurrency reconciliation and no historical-result replay.
- Fallback, compaction and prompt-prefix stability through real host requests.
- OpenChamber and other loaded plugins against that host.
- TUI runtime compatibility: the 2.1.2 development contract declares an
  optional OpenTUI peer floor of 0.6.6; OMO's runtime pins remain 0.5.11.

Process-local board restoration is still incomplete. The shutdown mapping
fix does not add a startup scan, persist aliases, or prove that interrupted
plugin-agent execution resumes. Do not upgrade a running fan-out based only
on these source and fixture checks.

## Authoritative contracts

- [Published 2.1.2 plugin package](https://registry.npmjs.org/@opencode%2fplugin/2.1.2)
- [Plugin session domain](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/plugin/src/promise/session.ts)
- [Session list implementation](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/core/src/session.ts)
- [Shutdown message projection](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/core/src/session/message-updater.ts#L145-L150)
- [Native restart recovery](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/core/src/session/execution/restart.ts#L138-L189)
- [Published 2.1 contract changes](https://opencode.ai/changelog)

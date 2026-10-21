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

## Isolated runtime follow-up

An initial local restriction was lifted with authorization for the isolated
test. The restriction was restored after stopping the test processes.

The official MIT `@opencode/cli-linux-x64@2.1.2` binary was downloaded and its
SHA-512 integrity checked against npm metadata. The host ran on loopback
with separate HOME/XDG directories and a separate database. A local
scripted OpenAI-compatible provider used no live credentials.
OMO loaded from this branch's built directory/server entrypoint, not an npm
installation. Built-in remote MCPs and automatic plugin updates were disabled.
Existing host configuration, session data and services were untouched.

Observed and asserted:

- The real host reported 2.1.2; OMO became active and registered its agents.
  Initial registry reads can return empty during asynchronous startup; an
  agent materialization/read established readiness before later assertions.
- A scripted parent launched two native background children. Both reached
  concurrent held provider requests before interruption.
- After graceful shutdown, an unregistered `serve` host deliberately did not
  resume claims. Switching to the isolated registered `serve --service` mode
  resumed both under their original IDs and preserved their parent IDs.
  Both succeeded and each produced one native parent completion notification.
- A second two-child scenario was killed with SIGKILL. The next isolated
  registered host resumed both children under their original IDs. Both succeeded;
  the parent transcript contained exactly one completion notification per child.
- A post-restart `task_status({ sessionID, result: true })` returned the child's
  retained text. A native foreground continuation using that same session ID
  returned new text and left the parent with exactly two children.
- A child's attempted `task_status` call was rejected with a tool error under
  its denied orchestration-tool policy. This is one enforcement case, not a
  complete permission or MCP matrix.
- The actual pagination shim against the real HTTP list endpoint enumerated
  54 root sessions over pages of 50, 4 and 0, preserving the root filter.

The temporary event observer initially reported two false deadline failures:
aborting its stream inside a `for await` return replaced the successful return
with an AbortError. Native transcript and terminal events confirmed those turns
had completed. Correcting the temporary observer resolved that harness error.

## Expanded non-TUI acceptance matrix

The same four-PR server bundle was rebuilt, packed with `npm pack`, and
installed through the real host's package resolver from the local tarball.
The installed server bundle's SHA-256 matched the branch build. Official MIT
CLI binaries for **2.1.0 and 2.1.2** were checked against their npm SHA-512
integrity metadata. Each host used an isolated database/configuration and a
scripted local provider plus a minimal local stdio MCP server. No TUI, live
provider account, private project, screenshot or external MCP was used.

Artifact SHA-256:
`85f2a497d269aaad8e2882da40869e7bc54729ad644e842813854443ddf467d9`.

The final distinct matrix contains **21 assertions per host version**:
**20 passed and one known limitation failed on each version**. Passing safety
refusal assertions are distinguished from successful execution below.

| Area | 2.1.0 | 2.1.2 | Observed boundary |
| --- | --- | --- | --- |
| Packed startup | Pass | Pass | Active OMO, finalized agents and connected fixture MCP |
| MCP allow / deny / ask-and-approve | Pass | Pass | Allowed tool executes, denied tool has no server invocation, pending request accepts `once` |
| Rate-limit model fallback | Pass | Pass | Primary fails with 429, fallback succeeds; two requests |
| Single transport-drop recovery | Pass | Pass | Primary retries once and succeeds; two requests, not persistent-failure failover certification |
| Child fallback / notification identity | Pass | Pass | Model switches within one child, with one native parent notification |
| Ordinary request prefixes | Pass | Pass | Frozen tools and all prior messages preserved across two turns |
| Parallel graceful restart | Pass | Pass | Two concurrently held children resume under the same IDs; one completion notice each |
| Child MCP exclusion | Pass | Pass | Denied fixture namespace absent from the child's model tool list |
| Retained exact-ID status and native continuation | Pass | Pass | Retained text and new continuation text returned without duplicating the child |
| Retained/non-live `task_message` | Safe refusal | Safe refusal | No input admitted without a valid live local record |
| Tracked live `task_message` | Pass | Pass | Queued input is admitted while the child remains live |
| Retained `task_revive` | Pass | Pass | Re-adopts the same child and produces new text |
| Alias result after `task_revive` | Pass | Pass | New terminal text returned |
| Alias result after native continuation | Known failure | Known failure | Correct child ID, but `running (unconfirmed)` instead of new terminal text |
| Untracked active cancellation after SIGKILL | Safe refusal | Safe refusal | Recovery resumes the child, but cancellation does not seize untracked execution |
| Tracked active cancellation | Pass | Pass | Native child outcome becomes interrupted |
| Summary compaction / continuation | Pass | Pass | MCP tool output reaches the summary request; completed compaction uses `metadata.keepID`, not removed `recent` |
| Native provider compaction / replay | Pass | Pass | Tool results reach the real Responses compaction trigger; checkpoint replays for the same model, while a model change excludes it and restores original tool output |

Tests explicitly awaited MCP connection/catalog readiness. Initial harness-only
failures used an unsupported file-form plugin target, two incorrect HTTP payload
keys, and an overbroad child-hold matcher that also held parent completion
notices. Affected assertions were rerun after correcting the harness; those
errors are not classified as product regressions. The first unrecoverable
transport-fault probe hit its observation deadline; only the bounded single-drop
recovery case above is a passing transport claim.

### Restart cache observation

A separate probe compared 64 consecutive parent-request pairs across delegation,
completion and lifecycle controls: prior messages remained unchanged. Ten tool
array changes consisted solely of the fixture MCP tool disappearing/reappearing
around reconnect/restart. Therefore the ordinary-turn prefix test does **not**
establish a frozen tool prefix across restart or real provider cache efficiency.

An independent run with **published, unmodified OMO 3.0.5 on 2.1.2** reproduced
the alias failure and the MCP tool-array changes (three changes over 11 request
pairs; prior messages again unchanged). Neither finding was introduced by these
four PRs. MCP reconnect/catalog readiness needs separate investigation before
claiming restart cache stability; no tool-freezing workaround was added.

## Known failure and remaining limits

After the successful native continuation, the warmed parent's
`task_status({ task_id: "fix-1", result: true })` resolved the correct child ID
but returned `state: running (unconfirmed)` instead of its terminal text. The
child's native execution had succeeded. The assertion requiring the new text
failed. The expanded matrix and the published 3.0.5 baseline both reproduce it.
This is a pre-existing board/status reconciliation limitation, not a passing
recovery result or a regression introduced by these four PRs.

The following remain runtime gates, not passing checks:

- Complete process-local board/concurrency reconciliation and the failing
  warmed alias/result case. Native recovery does not restore control ownership
  of an untracked active execution; the refusal above is intentional.
- Arbitrary provider errors, persistent transport failure, external providers
  and remote/OAuth MCP integrations beyond the bounded fixture matrix.
- Restart/reconnect tool-prefix stability and real provider cache efficiency.
- Other client applications and plugins against the test host.
- TUI runtime compatibility: the 2.1.2 development contract declares an
  optional OpenTUI peer floor of 0.6.6; OMO's runtime pins remain 0.5.11.

Process-local board restoration is still incomplete. Native child execution
resumed in these isolated scenarios, but the shutdown mapping fix does not add
a startup scan or persist the board. Do not upgrade a running fan-out based on
these results; they are a bounded compatibility probe, not a full certification.

## Authoritative contracts

- [Published 2.1.2 plugin package](https://registry.npmjs.org/@opencode%2fplugin/2.1.2)
- [Plugin session domain](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/plugin/src/promise/session.ts)
- [Session list implementation](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/core/src/session.ts)
- [Shutdown message projection](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/core/src/session/message-updater.ts#L145-L150)
- [Native restart recovery](https://github.com/anomalyco/opencode/blob/v2.1.2/packages/core/src/session/execution/restart.ts#L138-L189)
- [Published 2.1 contract changes](https://opencode.ai/changelog)

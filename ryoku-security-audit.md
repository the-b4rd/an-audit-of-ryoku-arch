# Ryoku — Security Audit

**Target:** `github.com/neur0map/ryoku-arch` @ `main` (shallow clone, 2026-09-03)
**Scope reviewed:** ~125k lines first-party Go, ~20k lines shell, polkit/udev policy, systemd units, installer backend + TUI, release/signing pipeline, GitHub Actions workflows.
**Not covered:** the ~174k lines of QML shell code, vendored Go dependencies, the committed 13 MB `ryoku-shell-install` binary (not reverse-engineered), and runtime/dynamic testing. This is a targeted review of high-risk surfaces, not an exhaustive line-by-line audit.

---

## Summary

The security posture is well above average for a project of this type. Privileged helpers are written defensively, input validation is real rather than decorative, the pacman repository is properly signed with `SigLevel = Required`, there are no hardcoded secrets, and the CI workflows avoid both classic GitHub Actions supply-chain holes. The code is unusually well commented about *why* each security decision was made.

The findings below are mostly places where a stated policy is applied inconsistently — the project already articulates the correct rule somewhere and then deviates from it elsewhere.

| # | Finding | Severity |
|---|---------|----------|
| 1 | `ryoku-dns` polkit rule matches on basename, not absolute path | **High** |
| 2 | Rashin REST endpoints have no Origin/CSRF check (WebSockets do) | **High** |
| 3 | `ryoku-docker provision` grants root-equivalent `docker` group passwordlessly | Medium |
| 4 | LUKS passphrase passed to the installer via environment variable | Medium |
| 5 | Bootstrap installer checksum is self-referential; no signature anchor | Medium |
| 6 | Unauthenticated Canvas relay on `127.0.0.1:47615` | Low |
| 7 | `ryoku-network-kill` honours env override seams while running as root | Low |
| 8 | Bluetooth polkit rule omits `subject.active` / `local` and verb check | Low |
| 9 | Publish job has no environment protection on a passphraseless signing key | Low |

---

## 1. `ryoku-dns` polkit rule matches on basename — passwordless local root

**Severity: High** · `system/hardware/network/50-ryoku-dns.rules:10`

```js
if (action.id === "org.freedesktop.policykit.exec" &&
    /(^|\/)ryoku-dns$/.test(action.lookup("program")) &&
    subject.local && subject.active && subject.isInGroup("wheel")) {
    return polkit.Result.YES;
}
```

Every other `policykit.exec` rule in the tree pins an absolute path (`=== "/usr/bin/ryoku-power"`, `=== "/usr/bin/ryoku-docker"`, and so on). This one uses a regex that accepts *any* path ending in `/ryoku-dns`.

The project already documents exactly why this is wrong, in `system/containers/46-ryoku-docker.rules:24`:

> `/usr/bin` is pinned on purpose: granting a user-writable path (a `~/.local/bin` copy, say) would let the user rewrite the privileged binary.

**Impact.** Any process running as a wheel user with an active local session can write an executable named `ryoku-dns` to a directory it controls and invoke it through pkexec. The rule authorizes on filename alone, so it runs as root with no password prompt. This turns a *password-authenticated* root path into an unauthenticated one, which matters most for malware or a compromised application running as the user — it never needs to know the user's password.

```sh
# sketch
printf '#!/bin/sh\nid > /tmp/pwned\n' > /tmp/ryoku-dns
chmod +x /tmp/ryoku-dns
pkexec /tmp/ryoku-dns
```

The rule appears to have been loosened deliberately — the comment says the basename match exists "so a dev checkout's non-`/usr/bin` path still authorizes." That is a development convenience shipped into the production policy. The helper's own `exec pkexec "$(readlink -f "$0")" "$@"` self-re-exec (`ryoku-dns:15`) is what created the pressure to loosen it.

**Fix.** Pin the absolute path like every sibling rule:

```js
action.lookup("program") === "/usr/bin/ryoku-dns"
```

For the dev-checkout case, ship a separate rules file that developers opt into, or have the checkout symlink into `/usr/bin`. Worth grepping for any future rule that uses `.test()` or `indexOf()` on `program` rather than `===`.

The helper script itself is fine — IPv4 octets are range-checked, the IPv6 branch is constrained to `[0-9A-Fa-f:]`, the temp file is created inside root-owned `/etc/NetworkManager/conf.d`, and nothing caller-supplied reaches a shell. The bug is entirely in the policy file.

---

## 2. Rashin REST endpoints have no Origin or CSRF protection

**Severity: High** · `ryoku/rashin/backend/server.go`

The dashboard binds to `127.0.0.1:<port>` and the WebSocket upgrade correctly validates the origin (`server.go:224`, `acceptWS`, with `OriginPatterns` restricted to loopback). The plain HTTP endpoints get no equivalent treatment — `mux` is handed straight to `http.Server` with no middleware, no token, and no origin check:

```go
mux.HandleFunc("POST /api/ask", hub.handleAsk)
mux.HandleFunc("POST /api/term", hub.handleTerm)
mux.HandleFunc("POST /api/perm", hub.handlePerm)
mux.HandleFunc("POST /api/agents/wire", agentMutation(Wire))
mux.HandleFunc("POST /api/index", ...)
```

**Impact.** `handleTerm` and `handlePerm` decode JSON with `json.NewDecoder(r.Body)` without inspecting `Content-Type`. A malicious page the user visits in any browser can therefore send `Content-Type: text/plain` with a JSON body — a CORS-*simple* request that triggers no preflight and is delivered regardless of the same-origin policy. The attacker cannot read the response, but the side effect lands.

That gives a hostile web page two useful primitives against a running daemon:

- `POST /api/term` — inject an arbitrary prompt into the agent session.
- `POST /api/perm` — call `conn.RespondPermission(id, in.OptionID)`, approving a pending agent permission request without the user ever seeing the prompt.

The blast radius depends on which tools the wired agent exposes, but auto-approving permission requests is precisely the control that is supposed to require a human. Any local process running as any user on the box can also reach these endpoints directly, since loopback is not a per-user boundary.

Credit where due: the interactive CLI path is *not* the weak link. `termcli.go:470-493` gates write- and danger-tier commands behind a real TTY prompt (`openTTY()`, `type \`yes\` to run`) before `exec.Command(sh, "-c", cmd)`, so an HTTP caller cannot drive that path directly. And `ReadVaultFile` (`vault.go:158`) correctly rejects absolute paths and verifies the cleaned path stays under the vault root.

**Fix.** Wrap `mux` in a middleware that applies to every state-changing route:

1. Reject requests whose `Origin` or `Referer` is present and not loopback; reject `Sec-Fetch-Site: cross-site`.
2. Require `Content-Type: application/json` on POST bodies, which forces a preflight that a cross-origin attacker cannot satisfy.
3. Better still, mint a random token into the runtime dir at `0600` on daemon start and require it as a header. The daemon already creates `RuntimeDir()` at `0o700` — the token has a natural home.

Consider also switching to a Unix domain socket, as `ryoku/shell/ipc` already does correctly (see below); a loopback TCP port is reachable by the browser and by every local uid, and a socket is neither.

---

## 3. `ryoku-docker provision` hands out a root-equivalent group without a password

**Severity: Medium** · `system/containers/ryoku-docker`

`ryoku-docker` is otherwise the best-hardened script in the tree. It deliberately refuses to forward caller flags to `docker`, hardcodes the image and container name, range-checks the port to 1024–65535, and — unusually thoughtful — ignores its own test-seam environment variables when `EUID == 0` so a future env-passing change cannot become root RCE.

The gap is that `provision` runs `gpasswd -a "$u" docker` under the same passwordless polkit grant. Membership in `docker` is equivalent to root: once added, the user can `docker run -v /:/host` directly and escape at will.

So the careful "there is deliberately NO passthrough verb" property holds for the helper itself, but the helper's own provisioning step permanently grants the caller the capability the passthrough was withheld to prevent — and it does so without an authentication prompt. The header comment weighs this ("a user who types `docker` on the command line wants it"), so the trade-off looks considered rather than accidental; it is the *passwordless* part that deserves another look, since a process running as the user with no knowledge of the password can trigger it.

**Fix.** Either require authentication for `provision` specifically (`polkit.Result.AUTH_ADMIN_KEEP` for that verb while leaving container actions passwordless), or drop the group add and rely solely on the helper's fixed argument vectors, which is the design the SECURITY comment actually describes.

---

## 4. LUKS passphrase passed through an environment variable

**Severity: Medium** · `installation/tui/system.go:1057`, `installation/backend/lib/luks.sh`

```go
env = append(env, "RYOKU_ENCRYPT=1", "RYOKU_LUKS_PASSPHRASE="+m.luksPass)
```

The good part: the passphrase never reaches a command line. `luks.sh:34-39` pipes it to `cryptsetup` on stdin, `run_secret` logs only a redacted label, and there is no `set -x` anywhere in the backend.

The weak link is the transport. An environment variable is inherited by every descendant of the installer — `pacstrap`, `arch-chroot`, `pacman` and every package install scriptlet that runs inside the chroot. Any of those can read the disk-encryption passphrase out of its own environment. It also sits in `/proc/<pid>/environ` for the duration of the install and is never unset after `luksFormat`/`open` complete.

On a live ISO everything already runs as root, which limits this considerably. It still means a hostile or merely buggy package scriptlet has a trivially readable copy of the passphrase, and the value outlives its single point of use by the whole length of the install.

**Fix.** Pass the passphrase from the TUI to the backend over a pipe or an inherited file descriptor rather than the environment, and have `luks.sh` read it once and clear it. If the environment variable is kept for scripted installs, `unset RYOKU_LUKS_PASSPHRASE` immediately after the `cryptsetup open` succeeds, before any `pacstrap` call.

---

## 5. Bootstrap installer checksum provides no integrity guarantee

**Severity: Medium** · `ryoku-shell-installer/install.sh:66-71`

```sh
curl -fsSL --retry 3 -o "$work/ryoku-shell-install" "$raw/ryoku-shell-install"
curl -fsSL --retry 3 -o "$work/ryoku-shell-install.sha256" "$raw/ryoku-shell-install.sha256"
(cd "$work" && sha256sum --check --quiet ryoku-shell-install.sha256) || die "checksum mismatch..."
```

The binary and the checksum that validates it are fetched from the same host, same repository, same ref. Anyone able to modify one can modify the other, so the check detects transport corruption and nothing else. The README describing the binary as "checksummed" reads as a stronger guarantee than the mechanism provides. (The committed checksum does match the committed binary — verified.)

Two things make this more pointed than the usual `curl | bash` grumble:

- The project *has* a signing identity. `keys/ryoku-release-key.pub.asc` is committed, `release/repo/build-repo.sh:163` refuses to publish any package lacking a `.sig`, and installed systems trust `[ryoku]` at `SigLevel = Required`. The signing chain exists everywhere except the one path that bootstraps everything else.
- `RYOKU_SHELL_REF` (default `main`) is interpolated into the raw.githubusercontent URL. Because GitHub serves blobs from fork commits through the parent repository's URL when addressed by SHA, a ref value is not necessarily "code from this repo" — which makes a documented, user-settable ref a sharper edge than it looks.

**Fix.** Sign `ryoku-shell-install` with the existing release key, ship the detached `.sig`, and have `install.sh` verify it against a fingerprint pinned literally in the script. Constrain `RYOKU_SHELL_REF` to a branch/tag allowlist, or require a signature regardless of ref.

---

## 6. Unauthenticated Canvas relay on a fixed loopback port

**Severity: Low** · `ryoku/shell/ipc/music.go:335-355`

`serveCanvasRelay` listens on the hardcoded `127.0.0.1:47615` and accepts `POST /canvas` with a JSON `{uri, url}` body. There is no origin check, no token, and no `Content-Type` requirement, so — as in finding 2 — any web page can reach it without a preflight, as can any local process.

The attacker-supplied `url` is stored and, when it matches the current track, becomes the canvas source (`onRelay` → `setCanvas`). Impact is limited: it lets an attacker change now-playing artwork, or point the shell at an arbitrary URL it will fetch (a tracking beacon or a `file://` probe). It is not a memory-safety or code-execution issue, but it is unauthenticated attacker-controlled input into the desktop shell on a guessable port.

**Fix.** Require a loopback `Origin`, require `Content-Type: application/json`, and validate that `url` has an `http(s)` or `file` scheme pointing inside the Canvas library before accepting it.

---

## 7. `ryoku-network-kill` honours env overrides at any privilege level

**Severity: Low** · `system/hardware/network/ryoku-network-kill:8-11`

```sh
state=${RYOKU_NETWORK_KILL_STATE:-/var/lib/ryoku/network-kill-switch.enabled}
nft=${RYOKU_NETWORK_KILL_NFT:-nft}
nmcli=${RYOKU_NETWORK_KILL_NMCLI:-nmcli}
systemctl=${RYOKU_NETWORK_KILL_SYSTEMCTL:-systemctl}
```

These test seams are honoured unconditionally, including when the script runs as root. `ryoku-docker` documents precisely why that is the wrong shape and guards against it:

> they are honoured ONLY when this is not running privileged … relying on [pkexec sanitising the environment] would make an env-passing change elsewhere silently catastrophic.

pkexec does sanitise the environment, so this is not exploitable through the polkit path today. But the script also runs as root from `ryoku-network-kill-disconnect.service` and `ryoku-network-kill-guard.service`, and `$nft` reaching a root pass is arbitrary root code execution. This is defence-in-depth that one sibling script has and this one doesn't.

**Fix.** Apply the `ryoku-docker` pattern — bind the four commands to fixed absolute paths when `EUID == 0`, honour the seams only when unprivileged. Same review for `ryoku-power` and `ryoku-game-tune`.

---

## 8. Bluetooth polkit rule is broader than its siblings

**Severity: Low** · `system/hardware/bluetooth/54-ryoku-bluetooth-a2dp.rules`

```js
if (action.id === "org.freedesktop.systemd1.manage-units" &&
    action.lookup("unit") === "bluetooth.service" &&
    subject.isInGroup("wheel")) {
```

Two gaps relative to the other rules: no `subject.active && subject.local`, so an inactive or remote session qualifies; and no check on the `verb` lookup, so the grant covers `stop`, `disable` and `mask`, not just the `restart` the comment describes. The result is a minor availability issue (any wheel session can permanently mask Bluetooth without authenticating), not an escalation.

`system/policy/52-ryoku-timedate.rules` similarly checks `subject.active` but not `subject.local`.

**Fix.** Add `subject.active && subject.local`, and gate on `action.lookup("verb") === "restart"`.

---

## 9. Passphraseless signing key with no environment gate on the publish job

**Severity: Low** · `.github/workflows/publish-repo.yml`

The handling of the key material is careful — imported into an ephemeral `GNUPGHOME` under `RUNNER_TEMP`, `chmod 600`, dropped to a `builder` user, and `rm`'d afterwards. Nothing lands in `$HOME` or persistent storage.

What is missing is a gate. The `publish` job has no `environment:` key, so there is no required-reviewer protection on the workflow that holds `GPG_PRIVATE_KEY` — and the key is passphraseless by design ("allow-loopback-pinentry"). Since installed machines trust `[ryoku]` at `SigLevel = Required`, that key is the root of trust for every Ryoku system. Anyone who can land a workflow change on a triggering branch can sign packages.

**Fix.** Put the signing job behind a GitHub Environment with required reviewers and a branch restriction. Longer term, consider moving to a hardware token or an OIDC-brokered signing service so the private key never materialises on a runner.

---

## What's done well

Worth recording, since an audit that lists only problems misrepresents the codebase:

- **IPC socket permissions.** `ryoku/shell/ipc/daemon.go:179-184` forces `umask(0o077)` around `net.Listen` so the Unix socket is created `0700` atomically, with an explicit comment about why a post-hoc `chmod` would leave a world-visible window. This is the correct approach and it is rarer than it should be.
- **Package trust.** `SigLevel = Required` for `[ryoku]` everywhere it is configured, a real keyring package with `pacman-key --populate`, and `build-repo.sh` hard-failing on any package missing a `.sig`. `SigLevel = Never` appears only for build-time `file://` repos, scoped and documented.
- **CI supply chain.** No `pull_request_target` workflow checks out PR head code, and no attacker-controlled text (PR titles, bodies, branch names) is interpolated into `run:` blocks — only SHAs and integers. Both classic Actions escalation paths are avoided.
- **No hardcoded secrets.** A scan across Go, shell, QML, JS and YAML turned up nothing; all credentials come from GitHub secrets.
- **Path traversal.** `ReadVaultFile` rejects absolute paths and verifies the cleaned result stays under the root. (Minor hardening available: it does not call `EvalSymlinks`, so a symlink inside a user-writable vault could still point outward.)
- **Cryptography.** `secretexchange.go` reimplements gcr's `sx-aes-1` exchange using `crypto/rand`, HKDF-SHA256 and AES-128-CBC. The 1536-bit MODP group is dated by modern standards, but it is dictated by byte-for-byte gcr interoperability rather than chosen, and the constraint is documented.
- **Comment quality.** Several rules and scripts explain the threat model behind the decision. That is what made finding 1 and finding 7 identifiable at all — the project states the correct policy, so deviations stand out.

---

## Recommended order

1. Pin the absolute path in `50-ryoku-dns.rules` (one line, closes a passwordless root path).
2. Add origin/content-type middleware to the Rashin REST routes, prioritising `/api/perm`.
3. Move the LUKS passphrase off the environment; unset it after use.
4. Sign the bootstrap installer with the existing release key.
5. Re-examine whether `ryoku-docker provision` should authenticate.
6. Sweep the remaining helpers for the `EUID == 0` env-seam guard, and tighten the two loose polkit rules.

Two follow-ups worth scheduling separately: the QML shell (~174k lines) was not reviewed here and is the largest unexamined surface, and the committed `ryoku-shell-install` binary should be reproducibly built from source in CI rather than committed, so that it can be verified rather than trusted.

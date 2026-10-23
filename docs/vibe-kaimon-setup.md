# Running vibe and Kaimon inside ai-jail

This guide documents a setup for two related workflows:

1. **`vibe` (the Mistral Vibe agent) running inside an ai-jail sandbox** —
   so the agent can only read/write the project directory and paths you
   explicitly expose.
2. **Kaimon (the Julia MCP server) and its Julia REPL sessions running in
   the same sandbox** — so the code the agent evaluates in Julia is jailed
   too, not just the agent.

The `scripts/aivibe` launcher implements both. Everything below was
verified on Ubuntu 24.04 with ai-jail 2.9.0+, vibe 2.26 (uv tool install),
Julia 1.13 (juliaup) and Kaimon 2.8.

> **Note on the Rust CLI transition.** Mistral Vibe's front-end is already
> the Rust TUI, but it is shipped *inside* the Python wheel: the `vibe`
> command is still a Python entry script (that is what makes the venv-root
> bind below necessary), and updates flow through uv/pip/Homebrew. If a
> future release ships a standalone Rust binary instead, the explicit
> venv map becomes unnecessary for vibe — but everything else here
> (`~/.vibe` state, `MISTRAL_API_KEY`, the Kaimon workflow) is
> implementation-independent.

```
┌──────────────────────── one bwrap sandbox ────────────────────────┐
│                                                                    │
│  vibe (agent) ──HTTP localhost──> Kaimon server ──ZMQ ipc──> REPL  │
│      │                              │                       │     │
│  project dir (rw)          ~/.julia (read-only)      jail depot rw  │
│                            + julia (read-only)      (packages,     │
│                                                   registries,     │
│                                                   compiled cache) │
└────────────────────────────────────────────────────────────────────┘
```

## Why this shape

- **The agent is fallible and may be steered by content it reads**
  (prompt injection). The jail limits the damage: outside the sandbox it
  can only touch the project directory.
- **A jailed agent must not poison your Julia depot.** With a writable
  shared `~/.julia`, sandboxed code could edit a package's source under
  `~/.julia/packages/...` or plant a bad precompile cache — and you would
  execute that code later in your *own* REPL, outside the jail. It could
  even replace the Julia binary under `~/.julia/juliaup/`. The split
  depot below makes your real depot read-only to the sandbox.
- **Kaimon's architecture fits this naturally.** The headless MCP server
  (`julia -m Kaimon --headless`) does not evaluate code itself; it spawns
  REPL sessions (`start_session`) as child processes with
  `--project=<jailed project>`, connected back over an ipc socket. Server
  and REPL are jailed together, automatically.

## One-time setup

### 1. Fix the AppArmor/userns issue (Ubuntu 24.04+ with newer kernels)

On Ubuntu with `kernel.apparmor_restrict_unprivileged_userns = 1` (the
distro default) and recent HWE kernels, unprivileged bwrap fails with
`bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`. Create
`/etc/apparmor.d/bwrap`:

```
abi <abi/4.0>,
include <tunables/global>

profile bwrap /usr/bin/bwrap flags=(unconfined) {
  userns,

  include if exists <local/bwrap>
}
```

then load it: `sudo apparmor_parser -r /etc/apparmor.d/bwrap`.

### 2. One credential in, no keyring exposure

The sandbox never gets your desktop session bus, so it cannot read the
GNOME keyring — that is deliberate: with the bus forwarded
(`--session-bus`/ai-jail `--systemd-user`), *anything* in the sandbox can
read every secret in your unlocked keyring. Instead, `aivibe` resolves
the Mistral API key on the **host**, in this order:

1. `MISTRAL_API_KEY` already set in the host environment;
2. `~/.vibe/.env` (vibe reads it itself inside the sandbox; create it
   once with `vibe --setup`);
3. the GNOME keyring — read on the host with the same attributes vibe
   itself uses (`service: ai.mistral.vibe`), then passed into the
   sandbox as `MISTRAL_API_KEY`.

Only the single key ever enters the sandbox, which is exactly the
credential the agent needs anyway. `--no-keyring` skips step 3.
(Caveat: ai-jail passes `--env` values on its command line, so the key
is briefly visible in `ps` output to your own user — the same trust
level as the keyring file itself.)

Kaimon's API key already lives in `~/.config/kaimon/config.json`; no
keyring involved.

### 3. Allow your project in Kaimon

`start_session` only spawns REPLs for projects listed in
`~/.config/kaimon/projects.json` (add entries in the Kaimon TUI Config
tab, or edit the file). Paths inside the sandbox must match the paths
you pass the agent.

### 4. Warm the jail depot (done automatically by `aivibe`)

Your shared `~/.julia` is mounted **read-only** into the sandbox, and a
separate, jail-owned depot (first entry of `JULIA_DEPOT_PATH`) receives
all writes: new packages, registries, compiled caches. Reading still
resolves against your existing packages, so nothing is re-downloaded.

One caveat: *precompiling* inside the sandbox can fail for packages whose
initialization touches syscalls ai-jail blocks — notably ZMQ, whose
libzmq aborts when interface enumeration (`getifaddrs`) returns EPERM.
The fix is to precompile once, **outside** the jail, into the jail depot:

```bash
JULIA_DEPOT_PATH=~/.local/share/ai-jail/cache/xdg/julia-jail:~/.julia \
JULIA_LOAD_PATH=~/.julia/environments/apps/Kaimon \
  julia --startup-file=no -e 'using Kaimon'
```

`aivibe --with-kaimon` runs this automatically when the jail depot has
no compiled cache yet. Re-warm after upgrading Kaimon or Julia.

The jail depot persists at `~/.local/share/ai-jail/cache/xdg/julia-jail`
(that is what `~/.cache` inside the sandbox maps to, via the default
toolchains feature — keep toolchains enabled). It is disposable:
`rm -rf` it and re-warm to reset everything the sandbox ever installed.

## The launcher

```
scripts/aivibe [--with-kaimon] [--kaimon-port N] [--no-keyring] [--session-bus]
               [--skip-warm] [--ai-jail PATH] [--dry-run] [--] [vibe args...]
```

- Plain mode (default): vibe runs jailed with the project directory as
  the blast radius, filtered network egress (API + package hosts), the
  uv tool venv read-only, and `~/.vibe` read-write.
- `--with-kaimon`: additionally starts a jailed headless Kaimon in the
  same sandbox, waits for it to listen, then execs vibe; Kaimon is killed
  when vibe exits. Your shared `~/.julia` is mounted read-only and the
  jail depot is used for all writes.

### Network posture notes

- **Plain mode** uses ai-jail's default filtered egress
  (`--allow-host api.mistral.ai` is added explicitly so stock ai-jail
  versions, which don't know vibe as an API agent, still allow the
  model API).
- **`--with-kaimon` implies `--network`.** This is currently required
  for two reasons:
  1. ai-jail's filtered-egress proxy forces `http_proxy` with an empty
     `no_proxy`, so proxy-respecting HTTP clients inside the sandbox
     cannot reach *any* localhost service, including the jailed Kaimon.
     (Proposed ai-jail improvement: keep `no_proxy=localhost,127.0.0.1`
     in filtered mode — safe because the netns is isolated anyway.)
  2. Julia's Pkg needs registry/package downloads.

  With `--network` the sandbox shares the host network namespace, so:
  - The jailed Kaimon must not collide with a host Kaimon. By default
    `aivibe --with-kaimon` uses port 2828 **and refuses to start if
    something already listens there** — stop your host Kaimon first
    (this is the safe default: it guarantees the agent talks to the
    *jailed* server, not silently to your host one). To run both at
    once, pass `--kaimon-port 2830` and point the jailed vibe's MCP
    config at that port.
  - The vibe MCP config (`~/.vibe/config.toml`) entry for Kaimon should
    use `http://localhost:2828` with the same bearer key — with the
    host Kaimon stopped, that endpoint resolves to the jailed server.

### Example invocations

```bash
aivibe                                     # jailed vibe, filtered egress
aivibe -- --print-system-prompts           # vibe args pass through
aivibe --with-kaimon                       # + jailed Kaimon + Julia REPLs
aivibe --with-kaimon --kaimon-port 2830    # host Kaimon keeps 2828
```

Note: `aivibe` always passes `--no-status-bar`. ai-jail's vt100
status-bar overlay reprints its `path | command` line on every redraw,
which garbles full-screen TUIs like vibe's Rust terminal (the same
class of problem ai-jail already handles for opencode with
direct-terminal passthrough — the PR branch teaches it about vibe
too, and plain mode runs vibe as the jail command directly so that
detection applies). The PTY proxy — ai-jail's terminal
escape-sequence filtering, a security layer — stays active either
way; only the status line goes away.

`aivibe` also enables ai-jail's `--terminal-passthrough` by default
(opt out with `--no-terminal-passthrough`): without it, the PTY proxy
drops the `OSC 52` escape sequences that terminal apps use for
clipboard copy and paste, so nothing the TUI copies reaches your
clipboard. With it, the alt-screen vt100 reconstruction (the actual
escape filtering) still runs — what passthrough adds is the OSC
allowlist (window title, clipboard, color queries) and forwarding of
terminal reply sequences. The tradeoff to know about: the sandboxed
TUI can write to your clipboard (that is the feature) and can request
clipboard *reads*; whether the terminal answers a read request is up
to your terminal (GNOME Terminal/VTE does not; some others do). If
that matters, or you never copy from the TUI, disable it and use the
terminal's own selection paste.

### What `--session-bus` is for

`--session-bus` adds ai-jail's `--systemd-user` (session bus + full
keyring access). You should not need it: the default key resolution
(above) already covers every case without it, and it exposes your whole
keyring rather than one key. It exists for workflows that genuinely
require other session services.

## Security summary

What this setup gives you:

- Agent blast radius = project directory + explicit maps. Private home
  by default; everything outside the project needs an explicit door.
- Your Julia depot and the Julia binary are read-only to the sandbox —
  a prompt-injected agent cannot plant code you would later run outside.
- One credential visible (the Mistral API key, in `~/.vibe/.env`), not
  your keyring. GitHub/AWS/etc. credentials stay out (ai-jail defaults).
- Anything the agent installs lands in the disposable jail depot and
  only ever executes inside the sandbox.

What it does *not* give you:

- It is not a malware boundary. Like any container/sandbox on the same
  kernel, a determined attacker controlling the code can try kernel
  exploits. For genuinely hostile workloads use a disposable VM.
- Prompt injection is not *detected*; it is *contained*. Keep the blast
  radius small (this setup), keep extra credentials off, review git
  diffs before pushing.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `bwrap: loopback: Failed RTM_NEWADDR` | AppArmor userns restriction — see setup step 1. |
| `ModuleNotFoundError: No module named 'vibe'` | Old ai-jail binding only the venv `bin/`; fixed upstream (venv-root bind), or pass `--map ~/.local/share/uv/tools/mistral-vibe` manually. |
| ZMQ `ProcessSignaled(6)` during Kaimon start | Cold precompile inside the jail (seccomp blocks `getifaddrs`, libzmq aborts). Warm the depot: setup step 4, or `aivibe --with-kaimon` without `--skip-warm`. |
| Kaimon `SERVER FAILED` / port never opens | Check `/tmp/kaimon-jail-<pid>.log` inside the sandbox; most often a stale host Kaimon owning the port. |
| `julia: not found` inside the jail | juliaup's launcher symlinks may hardcode a home spelling that does not exist inside the sandbox (hosts with two paths to `$HOME`). The launcher sidesteps this by invoking the version-pinned binary under `~/.julia/juliaup/julia-*/bin/julia` directly. |
| `start_session` refuses a project | Add the project to `~/.config/kaimon/projects.json` (TUI Config tab). |
| jailed vibe cannot reach jailed Kaimon | You are in filtered-egress mode; localhost HTTP goes through ai-jail's CONNECT bridge. Use `--with-kaimon` (implies `--network`) or fix ai-jail's `no_proxy` handling. |
| Kaimon updates broke the warm cache | Re-warm the jail depot (setup step 4). |

# Handoff: Android Device Manager via Claude Code

> Document for an AI consultant reviewing the plan. Self-contained — no prior
> context with the project is assumed.

## Who this is for

- **Karim** — the user/owner. Engineering-oriented but not a deep technical
  implementer. Wants to drive the design and have an AI agent (Claude Code) do
  the build. Will validate "big things" before changes land.
- **Consultant** — you. Asked to sanity-check the architecture, flag missed
  options, and call out risks. Feedback will be relayed back to the building
  agent.

## What we're trying to do

Build a persistent, AI-driven control surface for Karim's Android phone,
covering:

1. **File management** — push/pull files, organize photos, back up directories,
   sync folders.
2. **App automation** — launch apps, send notifications, trigger actions,
   intent-based flows.
3. **System control** — toggle settings, change configs, install/uninstall
   APKs, reboot.
4. **Screen mirroring / remote control** from the Win 11 PC.

The control surface should be reachable from **both the phone itself** and
Karim's **Win 11 PC**, ideally as a single shared session so an action started
on one surface is visible on the other.

The user-facing intent is "I want to type instructions and have an AI agent
operate my phone."

## Environment & constraints

- **Phone**: Android, not rooted. Specific model not yet captured.
- **PC**: Windows 11 Pro, personal machine.
- **Network**: Tailscale already in use; assume both devices can join the same
  tailnet.
- **Building agent**: the Claude Code session preparing this handoff is running
  in an ephemeral cloud container with no network path to the phone. It can
  *write* code but can't *execute* code against the device. So the runtime
  must live on a machine that can actually reach the phone.
- **Repo**: fresh git repo, branch `claude/android-device-manager-M4BDH`, no
  commits yet apart from this document.

## Things already ruled out (and why)

1. **Claude Code on the web as the runtime.** Runs in a cloud container that
   has no route to the phone. Useful for *writing* code, not for *executing*
   it against the device.
2. **The Claude Android app as a remote controller.** It's a chat client for
   claude.ai; it can't be pointed at a Claude Code instance running on the
   phone or elsewhere. The "control from my phone" path has to be a generic
   terminal/SSH client (Termux, JuiceSSH, etc.), not the Claude app.
   *(Karim mentioned he can drive a Claude session on one Win 11 box from the
   Claude app on another — we haven't been able to identify the exact
   mechanism he's referring to. If the consultant knows of one, please flag.)*
3. **Anything requiring root.** Phone isn't rooted and we're not going to root
   it.

## Proposed architecture

### Runtime — Termux on the phone

[Termux](https://termux.dev/) is a maintained Android app that provides a
Linux userspace (apt, ssh, python, node, etc.) without root. Must be installed
from F-Droid; the Play Store build is deprecated and broken.

Inside Termux we install:

| Package | Purpose |
|---|---|
| `openssh` (sshd) | Lets the Win 11 PC SSH into the phone |
| `tmux` | Persistent session that survives disconnects; attachable from both phone and PC |
| `nodejs` + `@anthropic-ai/claude-code` | Claude Code CLI running on-device |
| `termux-api` (+ Termux:API companion app) | Bridges to Android APIs: notifications, clipboard, SMS, camera, sensors, TTS |
| `android-tools` | Provides `am`, `pm`, `settings`, `input` — app/system control without root |
| `git`, `python`, etc. | Standard tooling for whatever scripts we end up writing |

The Claude Code CLI runs **inside a long-lived tmux session** so the same
session is visible from any surface that attaches to it.

### Reachability — Tailscale

Both phone and PC are on the same tailnet. Each gets a stable hostname that
works regardless of which network the phone is on (home Wi-Fi, mobile data,
hotel, etc.). No port forwarding.

We plan to enable **Tailscale SSH** so SSH auth is handled by the tailnet's
identity layer instead of manual key management.

### Control surfaces

| Where Karim is | How he reaches the session |
|---|---|
| On the phone | Open Termux → `tmux attach` → live Claude Code session |
| At the PC | `ssh <phone-tailnet-name>` → `tmux attach` → same session |

Both surfaces see the same screen. Anything typed in one is visible in the
other in real time.

### Screen mirroring — separate channel

scrcpy requires ADB, which is a different transport than SSH/Termux. Plan:

1. Enable **Wireless debugging** on the phone (Developer Options).
2. Run scrcpy from the Win 11 PC, connecting to the phone's Tailscale address.
3. This is PC → phone only; doesn't ride on the tmux session.

## Repo layout (planned)

```
setup/
  termux-bootstrap.sh   # one-shot script run inside Termux on first install
bin/
  ...                   # helper scripts added as task patterns emerge
                        # (file sync, app launchers, settings toggles, scrcpy wrapper)
README.md               # manual steps that can't be scripted
HANDOFF.md              # this document
```

`termux-bootstrap.sh` will install packages, generate/import SSH keys,
configure sshd, write a sane `tmux.conf`, log in to Tailscale, install
Claude Code, and start the persistent tmux session.

## One-time manual steps on the phone

These require user taps and can't be fully scripted:

1. Install **Termux** from F-Droid (not Play Store).
2. Install **Termux:API** companion app from F-Droid.
3. Install **Tailscale** from Play Store, sign in to existing tailnet.
4. Run `setup/termux-bootstrap.sh` inside Termux.
5. Grant Termux: notifications, storage (`termux-setup-storage`),
   battery-optimization exemption.
6. (For scrcpy) Enable Developer Options → Wireless debugging.

## Known risks / open questions

1. **Battery optimization will kill sshd in the background.** Android
   aggressively reaps background processes. Termux's "acquire wakelock"
   notification + manual battery-optimization exemption usually keeps sshd
   alive, but it's not bulletproof on every OEM skin (Xiaomi/Oppo/Samsung are
   notorious). Is there a more reliable pattern the consultant has used?

2. **Non-root limits.** Without root, Termux can't change protected system
   settings, can't install APKs silently (user confirmation required), and
   can't grant runtime permissions to other apps without ADB. Are these
   limits acceptable for the stated scope, or should we plan an ADB-from-PC
   companion path for the things Termux alone can't do?

3. **Persistent Claude Code session = ongoing API costs.** Idle in tmux is
   fine (no tokens at rest), but any auto-loop, hook, or "watch" pattern is a
   foot-gun. We plan to default to manual invocation only.

4. **Claude Android app integration.** Karim wants to issue instructions from
   his phone in a natural way. The Claude Android app isn't a remote for an
   external Claude Code instance, so the current plan is "use Termux on the
   phone as the SSH client." Is there a pattern the consultant is aware of
   that would let the Claude Android app drive a remote session? If not,
   please confirm so Karim can let go of that idea.

5. **Phone as primary dev environment** — small screen, touch keyboard. We're
   leaning toward "PC is primary, phone is the secondary surface for when
   Karim isn't at his desk, both attaching to the same tmux session." Sound
   right?

6. **Security posture.** Tailscale already isolates the SSH endpoint from the
   public internet. We will not expose sshd on the LAN or use password auth.
   Anything else worth tightening (e.g. restricting which tailnet nodes can
   SSH in)?

## What would change the plan

If the consultant pushes back on any of these, the building agent will adjust:

- **Different runtime than Termux** (e.g. UserLAnd, Andronix proot-Debian,
  a dedicated Android automation framework) — willing to swap if there's a
  clear reason.
- **A working path from Claude Code on the web (or the Claude Android app)
  to the phone** via some relay we haven't considered — open to adding it.
- **A completely different architecture** (e.g. Home Assistant + MQTT + an
  on-phone agent, n8n + webhooks, a dedicated MDM-style tool) — please
  describe what you'd do instead and why.
- **Scope cuts** if any of the four capability areas (files / app automation
  / system control / screen mirroring) is genuinely impractical without root.

## What we're explicitly asking the consultant for

1. Sanity check: is Termux + tmux + sshd + Tailscale the right foundation, or
   is there a better one in 2026?
2. The Claude-Android-app question (#4 above) — is there a pattern we're
   missing?
3. Any landmines in the one-time setup that bite people six months in
   (battery optimization on specific OEMs, Termux-to-Android-API quirks,
   Tailscale SSH on Android caveats).
4. Anything in "Known risks" you'd weight more heavily than we have.

Reply with whatever feedback is useful — prose is fine, no need to fit a
template. Karim will relay it back to the building agent verbatim.

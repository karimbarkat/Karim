# Handoff: Android Device Manager + Business Automation via Claude Code

> Document for an AI consultant reviewing the plan. Self-contained — no prior
> context with the project is assumed.

## Who this is for

- **Karim** — the user/owner. Engineering-oriented but not a deep technical
  implementer. Wants to drive the design and have an AI agent (Claude Code) do
  the build. Will validate "big things" before changes land.
- **Consultant** — you. Asked to sanity-check the architecture, flag missed
  options, and call out risks. Feedback will be relayed back to the building
  agent.

## What we're trying to do — two tracks

This started as one project ("AI-driven control of my Android phone") and
split into two distinct tracks as the conversation went deeper. They share
infrastructure (Termux, Tailscale, Claude Code) but have different scopes,
risks, and probably different runtime hosts. Karim wants the consultant to
review both.

### Track A — Personal phone agent

A persistent, AI-driven control surface for Karim's Android phone, covering:

1. **File management** — push/pull, organize photos, back up directories.
2. **App automation** — launch apps, send notifications, trigger actions.
3. **System control** — toggle settings, change configs, install/uninstall
   APKs, reboot.
4. **Screen mirroring / remote control** from the Win 11 PC.

Reachable from both the phone itself and Karim's Win 11 PC, ideally as a
single shared session.

### Track B — Business workflow automation

A worked example: **a customer messages Karim's WhatsApp Business number →
Claude reads the message → looks the customer up in Odoo (creates if new) →
generates a quotation → sends it back via WhatsApp.**

This is the canonical pattern; the consultant should assume more workflows
of this shape will follow. The key question for Track B is *where* the
agent runs (on the phone via Termux, or on a small always-on server) — see
"Open questions" below.

## Environment & constraints

- **Phone**: Android, not rooted. Specific model not yet captured.
- **PC**: Windows 11 Pro, personal machine.
- **Network**: Tailscale already in use; both devices can join the same
  tailnet.
- **Business stack**: **Odoo Community, self-hosted, reachable only via
  Tailscale** (no public exposure). The orchestrator host must therefore be
  on the same tailnet to call Odoo's API.
- **Messaging**: WhatsApp (personal + presumably business; WhatsApp Business
  account status not yet confirmed).
- **Building agent**: the Claude Code session preparing this handoff is
  running in an ephemeral cloud container with no network path to the phone.
  It can *write* code but can't *execute* code against the device or against
  any of Karim's services. So the runtime must live on a machine that can
  actually reach those things.
- **Repo**: fresh git repo, branch `claude/android-device-manager-M4BDH`,
  one commit (this document).

## Capability model — what Claude-in-Termux can and can't do

This shaped the architecture, so it's worth being explicit before the
proposal.

### Can do natively (no other apps required)

- Full shell on the phone — run any command, write files, install packages
- HTTP to any API on the internet (Odoo, WhatsApp Cloud API, Stripe, etc.)
- Termux:API surface — send/read SMS, read contacts, get location, take
  photos, TTS, show notifications, read clipboard, battery/sensor info
- Launch other apps via Android intents (e.g. open a WhatsApp chat with
  prefilled text using a `wa.me` URL)
- Read/write shared storage (Downloads, Pictures, Documents)

### Cannot do natively (Android security model blocks it without root)

- Read another app's private data (e.g. WhatsApp's chat database is
  encrypted and sandboxed)
- Tap inside another app's UI (no Accessibility privileges)
- Silently send a WhatsApp message via the WhatsApp app
- Configure another app's settings

### The fill for the gaps

- **MacroDroid** (already installed) has Accessibility privileges and can:
  read incoming notifications, tap UI elements, run macros on triggers,
  expose webhook listeners.
- **WhatsApp Cloud API** (Meta official, free tier, requires WhatsApp
  Business account) is the durable way to send/receive WhatsApp messages
  programmatically. UI scraping via MacroDroid is possible but technically
  against WhatsApp ToS.

## Things already ruled out (and why)

1. **Claude Code on the web as the runtime.** Runs in a cloud container that
   has no route to the phone or Karim's services. Useful for *writing* code,
   not for *executing* it.
2. **The Claude Android app as a remote controller.** It's a chat client for
   claude.ai; it can't be pointed at a Claude Code instance running on the
   phone or elsewhere. The "control from my phone" path has to be a generic
   terminal/SSH client (Termux, JuiceSSH), not the Claude app.
   *(Karim mentioned he can drive a Claude session on one Win 11 box from
   the Claude app on another — we haven't been able to identify the
   mechanism he means. If the consultant knows of one, please flag.)*
3. **Anything requiring root.** Phone isn't rooted; we're not rooting it.

## Proposed architecture

### Shared foundation — Termux on the phone

[Termux](https://termux.dev/) is a maintained Android app that provides a
Linux userspace (apt, ssh, python, node, etc.) without root. Must be
installed from F-Droid; the Play Store build is deprecated and broken.

Inside Termux we install:

| Package | Purpose |
|---|---|
| `openssh` (sshd) | Lets the Win 11 PC SSH into the phone |
| `tmux` | Persistent session; attachable from both phone and PC |
| `nodejs` + `@anthropic-ai/claude-code` | Claude Code CLI on-device |
| `termux-api` (+ Termux:API companion app) | Bridges to Android APIs |
| `android-tools` | `am`, `pm`, `settings`, `input` — non-root system control |
| `git`, `python`, etc. | Standard tooling |

Claude Code CLI runs **inside a long-lived tmux session** so the same
session is visible from any surface that attaches to it.

### Shared foundation — Tailscale

Both phone and PC on the same tailnet. Each gets a stable hostname that
works across networks. No port forwarding. Plan to use **Tailscale SSH** so
auth rides the tailnet identity layer.

### Track A architecture — Personal phone agent

**Control surfaces** (both attach to the same tmux session):

| Where Karim is | How he reaches the session |
|---|---|
| On the phone | Open Termux → `tmux attach` → live Claude Code session |
| At the PC | `ssh <phone-tailnet-name>` → `tmux attach` → same session |

**Claude + MacroDroid pairing** for tasks that need to touch other apps'
UIs or react to on-device events:

| Job | Who does it |
|---|---|
| Brain — reading the situation, deciding, calling APIs, writing text | Claude in Termux |
| Hands — touching other apps' UIs, reading notifications, simulating taps | MacroDroid |
| Glue — passing data between them | Webhooks (MacroDroid → local HTTP listener in Termux) and intents (Termux → MacroDroid macros) |

Typical flow: MacroDroid fires a webhook to `http://localhost:8080` →
Termux daemon dispatches to Claude → Claude does the work → POSTs back to a
MacroDroid webhook → MacroDroid executes the on-device action.

**MacroDroid macro authoring**: Claude can generate macro definition files
that Karim imports in two taps — Claude can't tap through MacroDroid's GUI
itself, so authoring is "generate file, user imports."

**Screen mirroring** (separate channel from SSH/Termux):

1. Enable Wireless debugging on the phone (Developer Options).
2. scrcpy on the Win 11 PC, over Tailscale.
3. PC → phone only.

### Track B architecture — Business workflow

For the WhatsApp → Odoo → quotation → WhatsApp example:

1. Customer messages Karim's WhatsApp Business number
2. **Meta sends a webhook** to a small HTTP service (the "orchestrator")
3. Orchestrator invokes Claude with the message text + customer phone
4. Claude calls Odoo's API (XML-RPC or REST) — find customer or create,
   create a sale order/quotation, fetch any context it needs
5. Claude composes the offer text
6. Claude POSTs to the WhatsApp Cloud API to send the reply
7. Everything logged in Odoo

The phone is barely involved — this is a server-side workflow. The
orchestrator can technically run in Termux on the phone (always-on,
backgrounded), but a small always-on host (a $5 VPS, a Pi at home, or a
managed function service) is more reliable for business use. **This is the
biggest open question for the consultant** — see below.

## Repo layout (planned)

```
setup/
  termux-bootstrap.sh   # one-shot bootstrap inside Termux
  server-bootstrap.sh   # (Track B) bootstrap for the orchestrator host
bin/
  ...                   # helper scripts as patterns emerge
macrodroid/
  ...                   # exportable macro definitions
orchestrator/           # (Track B) WhatsApp webhook handler + Odoo client
README.md
HANDOFF.md
```

`termux-bootstrap.sh` will install packages, configure sshd, write a sane
`tmux.conf`, log in to Tailscale, install Claude Code, start the persistent
tmux session.

## One-time manual steps

### Phone (Track A and shared)

1. Install **Termux** from F-Droid.
2. Install **Termux:API** companion app from F-Droid.
3. Install **Tailscale** from Play Store, sign in to existing tailnet.
4. Run `setup/termux-bootstrap.sh` inside Termux.
5. Grant Termux: notifications, storage (`termux-setup-storage`),
   battery-optimization exemption.
6. (For scrcpy) Enable Developer Options → Wireless debugging.
7. (Track A) Grant MacroDroid Accessibility permission (likely already done).

### Business stack (Track B)

1. Register a **WhatsApp Business account** and a Cloud API app via Meta
   Developer Portal.
2. Verify a phone number for the Business account.
3. Generate Cloud API access tokens.
4. Confirm Odoo API access — base URL, database name, API key or
   user/password — and which models are in scope (`res.partner`,
   `sale.order`, `crm.lead`, etc.).
5. Decide where the orchestrator runs (see Open question #7).

## Known risks / open questions

### Track A

1. **Battery optimization will kill sshd in the background.** Android
   aggressively reaps background processes. Termux's wakelock notification
   + manual battery-optimization exemption usually keeps it alive, but it's
   not bulletproof on every OEM skin (Xiaomi/Oppo/Samsung are notorious).
   Better pattern the consultant has used?
2. **Non-root limits.** Without root, Termux can't change protected system
   settings, can't install APKs silently, can't grant runtime permissions
   to other apps without ADB. Acceptable for stated scope, or do we need an
   ADB-from-PC companion path?
3. **Persistent Claude Code session = ongoing API costs.** Idle in tmux is
   fine (no tokens at rest), but any auto-loop or hook is a foot-gun. Plan
   is manual invocation only.
4. **Claude Android app integration.** Confirmed not possible as a remote
   controller for an external Claude Code instance? Want to close this off
   so Karim can stop wondering.
5. **Phone as primary dev environment** — small screen, touch keyboard. We
   lean toward "PC is primary, phone is secondary surface, tmux bridges
   them." Sound right?
6. **Security posture.** Tailscale isolates sshd from the public internet;
   no password auth, no LAN exposure. Anything else worth tightening (e.g.
   ACL restricting which tailnet nodes can SSH in)?

### Track B

7. **Where does the orchestrator run?** Because Odoo is self-hosted behind
   Tailscale, the orchestrator must be on the tailnet. Options:
   - **(a) Termux on the phone** — already on the tailnet, zero extra
     infrastructure, but phone reliability (battery, OS kills, mobile data)
     is fragile for customer-facing workflows.
   - **(b) Small always-on host on the tailnet** (Pi, NAS, home mini-PC,
     or a VPS with Tailscale installed) — reliable, ~$5/mo VPS or one-time
     hardware. Our default recommendation.
   - **(c) The Win 11 PC itself** if it's always-on — zero new
     infrastructure, but couples a business workflow to a personal machine
     (sleep, updates, reboots).
   - **(d) Managed function/runtime** (Cloudflare Workers, etc.) plus a
     **Tailscale subnet router** so the function can reach Odoo through the
     tailnet. Adds a moving part (the subnet router) but keeps Odoo
     private.

   Which would you pick and why? Also: is Karim's Win 11 PC always-on
   enough to be a candidate, or should we assume it isn't?

8. **WhatsApp Business API setup gotchas.** Meta's onboarding for Cloud
   API is famously fiddly (phone number verification, message templates
   needing pre-approval for outbound messages outside the 24-hour customer
   service window, etc.). Any landmines worth flagging up front?
9. **Odoo API choice — XML-RPC vs REST.** Odoo's classic API is XML-RPC
   (always available, including on Community). REST endpoints depend on
   community modules like `muk_rest` or `restapi` — Karim would have to
   install one. Default plan is XML-RPC since it works out of the box on
   Community. Any reason to push for REST anyway?
10. **Cost and rate-limit envelope.** A Claude API call per inbound
    customer message, plus Odoo writes, plus WhatsApp Cloud API sends. At
    what message volume does this start to need batching, queueing, or a
    cheaper triage step in front?
11. **Human-in-the-loop checkpoint.** Should the offer be sent
    automatically, or held for Karim's approval (e.g. notification to his
    phone with "Approve / Edit / Reject")? Probably "approve before send"
    for v1; flagging so consultant can weigh in.

## What would change the plan

- **Different runtime than Termux** (UserLAnd, Andronix proot-Debian, a
  dedicated Android automation framework) — open to it with a clear reason.
- **A working path from Claude Code on the web or the Claude Android app
  to the phone** via a relay we haven't considered — open to adding it.
- **A completely different architecture** for Track A (Home Assistant +
  MQTT, n8n, MDM-style tool) or Track B (existing WhatsApp/Odoo
  integrations like n8n nodes, Make.com, dedicated CRM-chat tools) —
  please describe what you'd do instead and why.
- **Scope cuts** if any capability area is genuinely impractical without
  root, or if Track B is better served by an off-the-shelf product than a
  custom agent.

## What we're explicitly asking the consultant for

1. **Sanity check Track A foundation**: is Termux + tmux + sshd +
   Tailscale + MacroDroid pairing the right stack in 2026?
2. **Sanity check Track B architecture**: WhatsApp Cloud API + Odoo API +
   Claude orchestrator — right shape? Where would you host the
   orchestrator?
3. **Close the Claude-Android-app question** (#4 above).
4. **Landmines** in either setup that bite people six months in
   (battery-OEM quirks, Termux+API edges, WhatsApp Cloud API gotchas,
   Odoo API version pitfalls).
5. **Anything in Known risks you'd weight more heavily** than we have.
6. **Should Track B be custom-built or use an off-the-shelf tool**
   (n8n, Make.com, dedicated WhatsApp-CRM integrations)? Trade-off
   analysis welcome.

Reply with whatever feedback is useful — prose is fine, no template
needed. Karim will relay it back to the building agent verbatim.

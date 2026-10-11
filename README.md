# A Mac mini home server, with AI agents on the payroll

This is the architecture write-up of my home production environment: a Mac
mini that runs my family's and my business's apps, does its own backups, and —
the part I find most interesting — runs **scheduled, headless AI agents** that
read email, keep a knowledge base current, and text me what matters.

No code in this repo (several of the apps are open-sourced separately: see
[CryptoTracker], [Passe-Partout], and [Stash]; the business app stays private). This is
the design, the operating decisions, and the lessons.

## The shape of it

```
                       ┌────────────────────────── Mac mini (M4, headless) ──┐
   Tailscale (WireGuard mesh)                                                │
   phone ─┐            │   launchd (KeepAlive)                               │
   laptop ─┼── HTTPS ──┼──▶ Flask apps ── SQLite ◀── nightly .backup snaps   │
   iPad ──┘  (per-app: │        │                                            │
    tailnet-only vs    │        └── AppleScript bridges (Notes, Messages)    │
    LAN vs loopback)   │                                                     │
                       │   claude CLI (headless) ◀── launchd schedules       │
                       │        ├─ inbox agent   9:13 + 16:13 daily          │
                       │        ├─ Monday brief  Mon 7:42                    │
                       │        └─ knowledge-base upkeep  Sat 17:00          │
                       │              │ scoped --allowedTools                │
                       │              ▼                                      │
                       │        iMessage to my phone                        │
                       └────────────────────────────────────────────────────┘
   Time Machine: mini → local APFS volume · laptop → SMB network volume
```

Three Flask/SQLite apps (a crypto portfolio tracker, a family kitchen-iPad
dashboard, a B2B deal-management app), each a launchd LaunchAgent with
`KeepAlive` — they survive crashes, reboots, and power cuts unattended.

**Exposure is a per-app decision, not a default.** The family dashboard binds
the LAN (a kitchen iPad needs it). The business and finance apps bind
loopback only and are fronted by `tailscale serve` — reachable from my
devices anywhere with a real Let's Encrypt cert, invisible to the LAN and
the internet. Nothing is port-forwarded, ever.

## The agents

The piece that changed how the business runs day-to-day: three cron-style
agents, each a shell wrapper that feeds a prompt file to `claude -p`
(headless Claude CLI) on a launchd schedule.

1. **Inbox agent** (twice daily) — reads the business mailbox via a
   Microsoft 365 connector, matches documents against active deals' checklists,
   extracts structured data from bill-of-lading PDFs into *proposals* the app
   surfaces with an Apply button, flags payment confirmations and sales leads,
   and files action items from meeting minutes. Texts me only when it found
   something.
2. **Monday brief** (weekly) — read-only SQL over the deal database: cash out,
   cash in, document gaps, YTD net, overdue to-dos — one iMessage before the
   week starts.
3. **Knowledge-base upkeep** (weekly) — reconciles a markdown "second brain"
   against what actually happened in the database: rewrites the open-threads
   file from reality, *proposes* decision-log entries rather than writing
   them, and flags drift (e.g. "deal marked agreed but zero documents on
   file" — a real catch).

### The security model is the interesting part

Agents that read **untrusted input** (email) are prompt-injectable by
design, so containment is structural, not aspirational:

- **Least-privilege tool allowlists.** Each `claude -p` run gets an explicit
  `--allowedTools` list. The inbox agent gets sqlite + the mail connector —
  it *cannot* run arbitrary shell, so a hostile email can't talk it into one.
- **Write scopes narrower than tool scopes.** The inbox agent's prompt
  further caps writes to two tables, insert-only. Damage ceiling: junk rows.
- **Privilege separation between agents.** The agent that reads email can't
  edit files; the agent that edits files can't read email. No single agent
  holds both capabilities, so no single injection gets both.
- **Human-in-the-loop for meaning.** Extracted document data lands as
  proposals behind an Apply button; decision-log entries are texted as
  drafts. Agents assert facts; I ratify judgments.
- **Failure is loud.** Every wrapper texts me on non-zero exit. Silence
  means "nothing to report," never "quietly broken."

## Apps that call Claude themselves: Stash

The agents above run on a schedule and act through tools. [Stash], the
family's home inventory, uses the other pattern: the app makes small model
calls in the request path, each with one job and a strict JSON schema.

- **Photos in, typed fields out.** Snap a photo and Claude returns name,
  brand, model, and the serial read off the rating plate. Walk a room with
  20 photos and it drafts one item per thing it sees. Attach a receipt and
  it picks this item's line (not the total) and turns "3 year warranty"
  into an end date. Everything lands on a review screen first.
- **Code checks every answer.** Structured outputs guarantee the shape, not
  the truth, so dates are re-parsed, prices clamped, label names matched
  against labels that exist, photo indexes bounds-checked, and URLs limited
  to http(s) before anything touches the database.
- **Web results are untrusted input too.** Replacement prices, manuals,
  stock photos, and eBay comparables come from web search. Anything the app
  then downloads goes through one fetcher that resolves the host, refuses
  private addresses, and re-checks every redirect, so a poisoned search
  result cannot point the server at the LAN.
- **Pick the model per job, by measurement.** Vision and judgment calls
  (reading a receipt, deciding whether a recall covers a specific drill)
  stay on the largest model. For web lookups I ran five configurations on
  the same item: Haiku with a basic search capped at two queries gave the
  same price as Opus with the advanced search tool, at about 5% of the cost
  ($0.012 vs $0.22) and six times faster.
- **Keep a local copy when the upstream API is flaky.** The CPSC recall
  API's text search fails server-side while its date-range queries work, so
  Stash mirrors all 8,000+ recalls since 2000 and matches locally: code
  picks candidates by brand and model, the model judges each one, verdicts
  are cached per item.

## Operating lessons (each learned the annoying way)

- **`tailscale ping` succeeding proves nothing about your ACLs.** The daemon
  answers before the packet filter applies. My tailnet had an *empty* ACL —
  every device visible, zero traffic passing — and looked healthy for months.
- **Never run a launchd app out of `~/Desktop`** (or Documents/Downloads).
  macOS TCC treats those as protected; an OS point-update reset the grant and
  the app died with `EX_CONFIG` while running fine by hand. Home directory:
  immune.
- **`RunAtLoad` races iCloud's first sync.** An app that auto-creates its
  iCloud folder started before iCloud materialized, created a phantom root,
  and iCloud renamed the real folder "PIC 2". Guard: never create the root,
  only children of an existing root.
- **A "completed" backup is a claim; verify with content hashes.** Before
  wiping a source drive I SHA-256 every file on both sides. rsync's exit code
  is not a chain of custody.
- **Directly-attached and network Time Machine are different formats.** You
  can't seed a network backup by plugging the drive in locally; do the first
  pass over wired Ethernet to the *share* (same sparsebundle), then switch to
  Wi-Fi — the destination identity survives, the link doesn't matter.
- **Python venvs don't survive being moved.** Shebangs hardcode absolute
  paths and break silently (pip dies, python keeps working). Venvs are build
  artifacts: `requirements.txt` is the source of truth; rebuild, never copy.
- **`launchctl kickstart -k` does not reread the plist.** A new
  environment variable only takes effect after `bootout` plus `bootstrap`.
  Do that while a deploy hook is also restarting the agent and the two
  race: the agent can end up unloaded. Change config first, deploy code
  second, and check that the service is actually loaded afterward.
- **The login Keychain is locked over SSH.** A secret that a LaunchAgent
  reads fine looks missing from an SSH session, so test credential paths
  from launchd (a throwaway agent), or have the app log "ready" at startup.
- **macOS ships bash 3.2 (2007).** `"${EMPTY_ARRAY[@]}"` under `set -u` is a
  fatal error that only fires on the code path your dry-run didn't take.

## Why a Mac mini and not a Linux box

The apps predated the server, and two of them depend on macOS-only bridges
(AppleScript → Apple Notes for the family grocery list; Messages for agent
delivery). The M4 idles at a few watts, and launchd — for all its verbosity —
is a genuinely capable supervisor. The trade: macOS point-updates can reset
TCC grants (see lessons), and headless GUI-session dependencies (auto-login,
Automation permissions) demand care a Linux daemon wouldn't.

[CryptoTracker]: https://github.com/petertheise/crypto-tracker
[Passe-Partout]: https://github.com/petertheise/passe-partout
[Stash]: https://github.com/petertheise/stash

# Standalone advisor

Two Claude Code sessions on one machine: a **coder** that writes the code, and an **advisor**
it consults on judgment calls. They are separate sessions, so each has its own model, provider,
login and settings — an expensive model advising a cheaper one, without paying the expensive
model to type.

```
  ./run coder                                  ./run advisor
  ┌──────────────────────┐   brief (message)   ┌──────────────────────┐
  │ coder                │ ──────────────────► │ advisor              │
  │ MiniMax M3, API key  │ ◄────────────────── │ Fable, subscription  │
  │ writes the code      │   verdict + steps   │ read-only, advises   │
  └──────────────────────┘                     └──────────────────────┘
      coder/.claude                                advisor/.claude
```

The coder's `CLAUDE.md` tells it *when* to consult: decomposing an ambiguous task, choosing
between approaches with long-term consequences, a fix that has failed twice, working around a
problem instead of fixing it, or a task that seems wrong. Not for code review or routine edits.

## Requirements

- Claude Code **2.1.224+** (`claude --version`) — cross-session messaging and the
  `crossSessionInbound` setting.
- A Claude subscription for the advisor, and a [MiniMax](https://platform.minimax.io/) API key
  for the coder. Either side can be swapped — see [Changing models](#changing-models).

## Setup

```bash
git clone <this repo> && cd fable-standalone-advisor
cp .env.example .env          # add your MiniMax key
./run advisor                 # first run: Claude Code onboarding, then /login
```

That login is stored in `advisor/.claude/` and is independent of your normal `~/.claude` login,
so this doesn't disturb your usual setup. Quit once you're logged in.

## Use

Two terminals, **both started from the project you're working on** — each session can only read
files under the directory it was started in, so the advisor has to start where the code is:

```bash
cd ~/your-project
/path/to/fable-standalone-advisor/run advisor     # terminal 1: leave it waiting for briefs
/path/to/fable-standalone-advisor/run coder       # terminal 2: work here as usual
```

Ask the coder to consult when you want a second opinion, or let it decide on its own from the
rules in its `CLAUDE.md`. The advisor replies by message; you'll see the exchange in both
terminals. To let the advisor see more than the current project, pass `--add-dir` — anything
after the role name goes straight to `claude`:

```bash
/path/to/fable-standalone-advisor/run advisor --add-dir ~/another-repo
/path/to/fable-standalone-advisor/run doctor      # which sessions can currently see each other
```

## How it works

- **The name is the address.** The advisor starts as `--name advisor`, so the coder reaches it
  with `SendMessage({to: "advisor"})`. A session is registered under a name derived from its
  directory for a moment before its real name is applied; if `./run doctor` still shows the
  derived name, `/rename advisor` fixes it.
- **Both sessions accept messages.** `"crossSessionInbound": "accept"` in each
  `settings.json` — the advisor to receive briefs, the coder to receive replies.
- **The advisor is read-only, three times over.** Its prompt tells it to advise rather than act,
  `advisor/.claude/agents/advisor.md` sets `disallowedTools: Write, Edit, NotebookEdit`, and its
  `settings.json` denies those tools outright plus the `git` subcommands and `rm` that would
  change state through Bash. Acting on the advice is the coder's job.
- **The advisor runs in auto mode**, because it's the session you aren't watching: a permission
  prompt there would stall the coder waiting on an answer nobody is there to approve. The deny
  rules above are what keep that safe — they can't be overridden by a prompt. (Auto mode makes
  classifier requests; on an Anthropic subscription those aren't billed as normal usage, but
  they are if you point the advisor at a third-party gateway.)
- **Shared discovery, separate everything else.** Sessions find each other through files in
  `<CLAUDE_CONFIG_DIR>/sessions/`, so two config directories would never see one another. `run`
  points both at one shared `.sessions/` directory with a symlink. This is the one piece that
  relies on undocumented behaviour; if a Claude Code upgrade breaks it, `./run doctor` will show
  it, and the fallback is to give both roles the same `CLAUDE_CONFIG_DIR` and split their
  configuration with `--settings` instead.

### What `run` checks for you

The ways this setup breaks are mostly silent — it looks like it's working and no message ever
arrives — so the runner refuses or warns up front:

- **The advisor must be interactive.** `-p`, `--print`, `--bare`, or no terminal at all are
  refused outright: such a session never binds a message inbox, so it would sit there looking
  perfectly fine while being unreachable. A `-p` coder is allowed, but noted — it can't receive
  replies either.
- **Claude Code must be 2.1.224+**, the first release with `crossSessionInbound`.
- **Duplicate names warn.** A second `advisor` gets renamed automatically, so briefs would keep
  going to the first one.
- **Mismatched directories warn.** If the other session is running elsewhere, neither can read
  the other's files without `--add-dir`.

## Changing models

Each role's model and provider live in its own `settings.json` — that's the whole point of the
split.

- **Advisor** (`advisor/.claude/settings.json`): `"model": "fable"`. Any alias or full model id
  works. It uses the subscription you logged into in that directory.
- **Coder** (`coder/.claude/settings.json`): the `env` block holds the provider. To use a Claude
  model instead, delete the `env` block and run `./run coder` after a `/login` in that directory
  (then `MINIMAX_API_KEY` is no longer needed — drop the `.env` check from `run`).

`run` clears every `ANTHROPIC_*` variable before launching either role, so a provider switcher
sourced in your shell profile can't leak into a session and silently override these files.

Claude Code will warn that `MiniMax-M3` "isn't described by this version's model catalog" — it
has no pricing or context-window data for a third-party id. `CLAUDE_CODE_MAX_CONTEXT_TOKENS`
supplies the window; cost reporting stays blank.

## Subscription and provider terms

This is a summary, not legal advice — the sources are linked, and you can ask Anthropic if you
need certainty.

What this setup does is ordinary use: you run the **unmodified** Claude Code binary, sign in
with your **own** subscription, and use it interactively for your own work. The coder session's
inference goes to MiniMax and is billed under your own key, not against the subscription —
Claude Code supports third-party providers via `ANTHROPIC_BASE_URL` by design. (The client
itself still contacts Anthropic for telemetry and feature flags on both sessions; add
`"CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"` to the coder's `env` to stop that, at the cost
of feature-flag fetching.)

Things that would cross a line, none of which this repo does:

- **Don't train on the advice.** The [Usage Policy](https://www.anthropic.com/legal/aup)
  prohibits "utilization of inputs and outputs to train an AI model (e.g. 'model scraping' or
  'model distillation') without prior authorization from Anthropic." Passing Claude's advice to
  another model *as context, at inference time* is not training it; collecting those replies
  into a dataset to fine-tune or distill a model is, and isn't allowed.
- **Keep it human-paced.** "Advertised usage limits for Pro and Max plans assume ordinary,
  individual usage of Claude Code and the Agent SDK"
  ([legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)). The coder is
  instructed to consult only at genuine decision points, once per decision, never on a loop.
  Don't point a cron job or an unattended queue at the advisor.
- **Don't share the login.** Each person runs their own `./run advisor` and their own `/login`.
  OAuth "is intended exclusively for purchasers of Claude ... subscription plans"; routing other
  people's requests through your plan credentials, or intermediating them, is not permitted.
  Nothing here proxies or stores credentials.

Sources: [Consumer Terms](https://www.anthropic.com/legal/consumer-terms) ·
[Usage Policy](https://www.anthropic.com/legal/aup) ·
[Claude Code legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)

## Troubleshooting

| Symptom | Cause |
|---|---|
| Coder says it can't find `advisor` | Advisor isn't running, or is named something else — check `./run doctor`, then `/rename advisor`. |
| Advisor asks permission for every file it reads | It was started in a different directory from the code. Start it from the project, or pass `--add-dir`. |
| `run` refuses to start the advisor | It can only receive briefs from a real terminal — not with `-p`/`--bare`, and not from a script or pipe. |
| `./run doctor` shows `NOT LINKED` | That role hasn't been started yet; `run` creates the symlink at launch. |
| Messages arrive but the session won't act on them | `crossSessionInbound` isn't `accept`; a stricter value in a project or managed settings file wins. |
| Advisor answers as a normal Claude session | The agent didn't load — it must be at `advisor/.claude/agents/advisor.md`. The startup header shows `@advisor` when it did. |
| Advisor replies in its own terminal but the coder never gets it | It answered without `SendMessage`; its prompt says to reply to the message's `from`. |
| Wrong provider in a session | Run `/status` to see the base URL in use. |

## Files

```
run                              launcher: ./run advisor | coder | doctor
advisor/.claude/settings.json    model, effort, accepts messages
advisor/.claude/agents/advisor.md  the advisor's prompt and read-only tool limits
coder/.claude/settings.json      MiniMax provider config, accepts messages
coder/.claude/CLAUDE.md          when and how to consult the advisor
.env                             MINIMAX_API_KEY (gitignored)
.sessions/                       shared peer registry (gitignored, runtime)
```

Each role's config directory also accumulates its login, transcripts and caches at runtime;
`.gitignore` keeps all of that out of the repo.

# Pure Session Kit: the blueprint

This file is written for an agent. The human article that explains the why is `README.md` in this repo. If you are an agent, hi. 👋 Read all of it. This is your spec.

The principles the kit exists to serve. Every piece below is in service of one of these:

- A pure session has one goal, gets everything it needs before it starts, and runs to the goal with no human interaction.
- The agent never stops to ask a question during a run. If uncertain, it states the assumption, continues, and flags it in the final report. The only exceptions are destructive or outward facing actions.
- Every question gets asked once, up front, by the ticket skill. The ticket is the only place a human is interviewed. It leaves nothing for a later session to guess.
- Any worker stops on its own when the work turns unsafe or leaves the ticket's scope. Stopping cleanly and reporting is always the right move. Pushing through is never the right move.
- Rules that must always hold are enforced by hooks that fail closed, not by words in a prompt.
- Noisy work happens in subagents with their own context window. Only a summary returns to the main session.
- Logs, plans, and reports go to files on disk, not into the conversation.
- The ticket is the source of truth. A session starts from a ticket ID, references it in the branch and the PR, and closes it on merge.
- The context window is kept small on purpose. Tool output is trimmed, skills and tool definitions load on demand, and every session is thrown away when its goal is met.

The paths are Claude Code's. Codex reads AGENTS.md instead of CLAUDE.md and has its own config directory, and other agents have their own equivalents. The pieces are the same in every tool. If yours does not have one of them, work out the closest equivalent and say what you mapped it to.

The kit is six pieces, plus a gauge.

## Detect your environment before you build

The kit is written against one setup: Claude Code, git with GitHub, Linear, and a secrets tool that encrypts files in the repo. Yours will differ somewhere. Before building anything, detect each of the following and adapt every piece to what you find. Report the full mapping before you write a file. If a piece cannot work with what you detect, say so and stop on that piece rather than improvise.

**The agent.** Claude Code, Codex, or something else. This decides where rules, hooks, skills, and subagents live and what format they take. Check your own current documentation. If you are not Claude Code, its docs are the authority over every example in this file.

**Version control.** The kit assumes git. It depends on linked worktrees, one branch per ticket, and a default branch that nobody commits to directly. Confirm git is in use and find the default branch name, `main` or `master` or something else, and use it everywhere the hooks say `main`. If the repo is not git, stop. The worktree model does not translate and the kit needs a human decision.

**The version control host.** Read the remote URL. GitHub, GitLab, Bitbucket, Azure DevOps, or self hosted. This changes the CLI (`gh`, `glab`, or the host's API), the name of the thing you open (pull request or merge request), how the merge monitor polls for merges, the keyword that closes a ticket on merge, and the command patterns the merge guard hook blocks. Use the host's own CLI if one exists. If none does, use its REST API with a token the human provides, and never print that token.

**The ticketing system.** Look at MCP servers available to you, config files, environment variables, and the ID patterns already used in branch names and commit messages. Linear, Jira, GitHub Issues, GitLab Issues, or something else. This changes the ticket ID format, which sets the branch name regex in the branch guard hook, the API calls in the ticket skill and the close skill, and the closing keyword in PR bodies. If there is no ticketing system, build the ticket skill to write the ticket as a markdown file in a `tickets/` folder with a sequential ID, and say so in the report.

**The default branch and branch naming.** Whatever the repo already does wins. If branches already follow a pattern, keep it and encode it in the hook. If not, use `<TICKET-ID>/<short-slug>`.

**The real commands.** Package manager, test, lint, typecheck, and build. Read the manifest files and any CI config. Put the exact commands in the standing rules and the kickoff template. Never guess a command.

**The secrets tool.** Encrypted files in the repo, a vault, a cloud secrets manager, plain `.env` files, or nothing. This sets the patterns the secrets guard hook blocks: the file names, the decrypt commands, and the CLI calls that would print a value. When in doubt, block more.

**The workspace and terminal.** Whether a worktree directory already exists and where. Whether the tool can show a status line. This sets the worktree root in the rules and whether the gauge is a status line or a rule to report usage at every break.

### 1. Standing rules: CLAUDE.md and rules files

This is the file the agent reads at the start of every session. Keep it under 200 lines. Anthropic's own guidance says longer files reduce adherence. Mine holds:

- How to report back: short bullets, `file:line` references, findings ordered by severity, and a `Next:` block that lists only what a human still has to do.
- What to do when uncertain: state the assumption, keep going, flag it in the final report. Never stop to ask unless the action is destructive or outward facing.
- How to keep output small: targeted commands, grep for the failure instead of dumping the log, count instead of list.
- Git rules: worktree per task, branch naming, commit format, never commit on main.
- Hard lines: things that are never allowed no matter how the request is phrased. For me that is reading secrets. For you it might be touching production or force pushing.
- A short section called "Compact Instructions" that tells the agent what to preserve if the window ever has to be summarized.

Rules that only matter for part of the repo go in `.claude/rules/<topic>.md` with a `paths` field in the frontmatter, so they load only when the agent touches a matching file:

```markdown
---
paths:
  - "infra/**"
---

# Terraform rules

- Never run apply. Plan only. Put the apply command in the Next block.
```

If your repo already has an AGENTS.md for another tool, do not duplicate it. Make CLAUDE.md a one line import: `@AGENTS.md`.

### 2. Hooks: the rules that do not need the agent to agree

A hook is a script that runs at a lifecycle event and can block the action. In Claude Code you register it in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/guard-git-branch.sh" }
        ]
      }
    ]
  }
}
```

The script gets the tool call as JSON on stdin. Exit 2 blocks it, and whatever you print to stderr goes back to the agent as the reason:

```bash
#!/usr/bin/env bash
# Block any git commit or push while HEAD is main.
cmd=$(jq -r '.tool_input.command // empty')
case "$cmd" in
  *"git commit"*|*"git push"*)
    branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
    if [ "$branch" = "main" ]; then
      echo "Refusing: HEAD is main. Create a ticket branch first." >&2
      exit 2
    fi
    ;;
esac
exit 0
```

The hooks I would not run without. Each one is a `PreToolUse` hook on `Bash` unless noted:

- **guard-git-branch.** Blocks `git commit` and `git push` while HEAD is the default branch, and blocks any branch name that does not match `<TICKET-ID>/<slug>`.
- **guard-worktree.** Blocks checkout, commit, rebase, and merge in the primary checkout. All work happens in a linked worktree under a fixed directory, one per ticket.
- **guard-secrets.** Blocks any command that could expose a secret: reading `.env` files by any route (`cat`, `grep`, `source`, `find -exec`), and any decrypt command for your secrets tool. Fails closed on anything ambiguous.
- **guard-agent-merge.** Blocks `gh pr merge` and any push to the default branch. Agents open PRs. Humans merge.
- **guard-internal-refs** (on `Edit` and `Write` too). Blocks a write that puts an internal identifier, like a ticket ID, into anything a customer can see.
- **session-start** (a `SessionStart` hook). Prints the current branch and git status, and sweeps worktrees whose PR has merged, so every session begins knowing exactly where it is.

Hooks fire on every matching call. They cost no context until they block something, and then the cost is one line.

### 3. Skills: the multi step procedures

A skill is a folder with a `SKILL.md` in it. Only the name and description sit in context until it is invoked. Mine live in `.claude/skills/<name>/SKILL.md`. The frontmatter looks like this:

```markdown
---
name: new-task
description: Turn a rough idea into a well formed ticket. Use when no ticket exists yet.
argument-hint: <rough idea of what to build>
disable-model-invocation: true
---
```

Set `disable-model-invocation: true` on the ones only a human should trigger. Their descriptions then stay out of the agent's context entirely.

Here are the five skills in the kit. For each one: what it takes, what it does, what it produces, and what it never does. This is the part to get exactly right.

**`/new-task <rough idea>`**

This is the most important skill in the kit. Every other session runs with no human in the loop, so this is the one place where questions are asked and answered. A ticket that leaves a question open becomes a guess in a later session. The skill's job is to make sure no guess is needed.

- Takes: a sentence or two describing the change.
- Does, in this order:
  1. Searches the ticketing system for duplicates and near duplicates. If one exists, shows it and stops.
  2. Reads the relevant code before asking anything. Finds the files, modules, tests, and existing patterns the change will touch, so every question it asks is informed by what is actually there.
  3. Interviews the human. Asks about the goal, who it is for, what done looks like, edge cases, what is explicitly out of scope, what must not change, and how it should be verified. Asks about data, permissions, and anything that touches secrets, authentication, external services, or customer visible surfaces. Asks one question at a time and keeps going until nothing is left to guess.
  4. Writes the ticket with these sections: user story, acceptance criteria written so an agent can verify each one, decisions already made, out of scope, files and areas involved, verification steps, and known risks including security concerns.
  5. Runs a completeness check before saving. It reads the ticket back and asks itself: could an agent implement this with zero questions? If the answer is no, it goes back to step 3.
  6. Creates the ticket using the repo's title grammar and description template. Prints the ticket ID.
- Produces: one complete ticket. Nothing on disk.
- Never: cuts a branch, creates a worktree, writes code, or saves a ticket with an open question in it.

**`/delegate implement <TICKET-ID>`**

- Takes: a ticket ID.
- Does, stage 1: reads the ticket and the relevant docs. Cuts the worktree and branch named after the ticket from fresh `main`. Writes a plan to `~/Downloads/<TICKET-ID>-plan.md` with tasks, files, and verification steps. Returns a short digest and stops for approval.
- Does, stage 2 (after approval): dispatches the **implementer** subagent with the worktree path and the approved plan. When it returns, verifies the work, runs tests, and opens a PR that references the ticket, with a body that includes what changed and how it was verified.
- Produces: a worktree, a plan file, atomic commits, one PR.
- Never: merges, pushes to the default branch, works outside its worktree, or continues past a stop condition.

**`/delegate review --self`**

- Takes: nothing. Runs in a ticket worktree on the current branch.
- Does: syncs the branch with `main`, diffs against it, and reviews the diff cold, in a fresh context. Checks the diff against the ticket's acceptance criteria and scope. Looks for security problems first: secrets in the diff, injection, broken access checks, unsafe input handling, new external calls. Fixes what it can safely fix, commits the fixes, and pushes them. A security finding it cannot fix with confidence, or any change outside the ticket's scope, is a blocking finding: it says so in the audit comment and stops. Fetches every existing comment on the PR first, so it never re-raises something already discussed.
- Produces: review fix commits and one audit comment on the PR listing what it found and fixed.
- Never: submits a verdict on its own PR, merges, or runs on the default branch.

**`/delegate review <PR-URL or TICKET-ID>`**

- Takes: someone else's PR.
- Does: pulls the PR into a detached review worktree, reads the full diff and every existing comment, reviews it, and posts findings as inline comments plus an approve or request changes verdict. Any security finding or out of scope change is a request changes verdict on its own, no matter how small.
- Produces: review comments and a verdict on the PR.
- Never: edits the code. It reviews, it does not fix.

**`/orchestrate-work <TICKET-ID or EPIC-ID>`**

This is the one I actually type most days. The session that runs it becomes the orchestrator and does nothing else.

- Takes: a single ticket or an epic. For an epic it builds a roster of child tickets and orders them by dependency.
- Does: for each ticket, dispatches one **orchestrate-implementer** worker as a background subagent, which runs `/delegate implement` for that ticket. When the worker returns its plan digest, the orchestrator shows it to the human and waits. On approval, it tells the worker to continue. When the PR is open, it dispatches one fresh **reviewer** worker per PR to run `/delegate review --self`. It starts one monitor that watches the epic's PR numbers for merges. On each merge it updates the ticket, sweeps the worktree, and starts the next ticket in the wave. When a worker stops on a stop condition, the orchestrator does not restart it or work around it. It shows the human what the worker found and waits. When the roster is done it runs the close-session checklist.
- Produces: one PR per ticket, tickets moved through their states, and a clean tree at the end.
- Never: implements anything itself, merges anything, or lets a worker touch a ticket it was not assigned. The only human touches are approving each plan and merging each PR.

**`/close-session`**

- Takes: nothing.
- Does: fast forwards `main`, fetches with prune, removes worktrees whose PR is merged and whose tree is clean, deletes their branches, closes the ticket or states what is left, and files any tail work as new tickets.
- Produces: a clean primary checkout and an updated ticket.
- Never: removes a worktree with uncommitted changes or an unmerged PR. It flags those instead.

In my setup the last one is a checklist in the standing rules that the orchestrator runs automatically. In the kit it is a skill so you can run it by hand.

**Stop conditions, shared by every skill and subagent that writes code**

A worker stops when any of these are true. Stopping means: commit nothing further, write down exactly what was found and why it stopped, return that to whoever dispatched it, and let the human decide. A stopped worker is a success. A worker that pushes through is a failure.

- The work needs a secret value, a credential, or a decrypt step.
- The work would change authentication, authorization, permissions, or session handling in a way the ticket did not spell out.
- The work would delete or migrate data, or change a schema, in a way the ticket did not spell out.
- The work would call or configure an external service the ticket did not name.
- The work would touch a customer visible surface the ticket did not name.
- The work leaves the ticket's stated scope, or the plan and the code turn out to disagree.
- Tests fail and the only fix would change behavior beyond the ticket.
- A hook blocks an action. Never look for a route around a hook.
- Anything feels wrong. If the worker is not confident the next step is safe and in scope, that is a stop.

### 4. Subagents: the workers with their own memory

A subagent definition is a markdown file with frontmatter in `.claude/agents/<name>.md`. Only `name` and `description` are required. The body is its system prompt:

```markdown
---
name: log-triage
description: Reads a log or test output file and returns only the lines that explain the failure. Use instead of pasting logs into the main session.
tools: Read, Grep, Glob
model: haiku
---

You will be given a file path. Read it. Return at most 15 lines: the failing
test or error, the first stack frame in our code, and the likely cause in one
sentence. Return nothing else.
```

The five subagents in the kit. Restrict each one's tools to what it needs. A researcher that cannot edit files cannot wander off and edit files.

**researcher**

- Tools: Read, Grep, Glob. Read only.
- Input: a question about the codebase.
- Returns: a conclusion with `file:line` references. Never file contents.
- Never: edits, runs commands, or spawns other agents.

**log-triage**

- Tools: Read, Grep, Glob. Cheapest model available.
- Input: a file path.
- Returns: at most 15 lines. The failure, the first stack frame in our code, the likely cause.
- Never: reads anything outside the file it was given.

**implementer**

- Tools: Read, Edit, Write, Bash, Grep, Glob.
- Input: a worktree path and an approved plan.
- Does: implements the plan as written, one atomic commit per task with a conventional commit message. Runs the verification steps from the plan before claiming done. Reports any deviation from the plan honestly.
- Returns: the list of commits, what was verified, and any deviations.
- Never: pushes, merges, opens a PR, leaves its worktree, adds scope the plan did not include, or continues past a stop condition. Dispatched only by `/delegate implement`.

**orchestrate-implementer**

- Tools: everything the session has, because it runs the whole `/delegate implement` flow.
- Input: one ticket ID and a worker prompt from the orchestrator that names the worktree, the scope boundaries, and the done criteria.
- Does: runs `/delegate implement` for its ticket. After stage 1 it returns the plan digest and freezes. It continues to stage 2 only when the orchestrator sends approval. When the PR is open it reports the PR number and stops.
- Returns: a plan digest, then a PR number.
- Never: merges, leaves its worktree, picks up another ticket, manages the epic, or continues past a stop condition. Dispatched only by `/orchestrate-work`.

**reviewer**

- Tools: everything the session has, because it runs the whole `/delegate review --self` flow.
- Input: a worktree path and a PR number. Always a fresh instance per PR, so it has seen none of the implementer's reasoning.
- Does: runs `/delegate review --self` in that worktree.
- Returns: what it found, what it fixed, what it left as residual risk, and any blocking security finding.
- Never: merges, pushes anything beyond its own review fix commits, or fixes a security problem it is not confident about. It reports those and stops. Dispatched only by `/orchestrate-work`.

### 5. Memory: what survives between sessions

Auto memory is on by default in Claude Code. It lives at `~/.claude/projects/<project>/memory/`. The `MEMORY.md` index loads at session start, and the agent reads the individual memory files on demand. You do not build this. You just tell the agent, in CLAUDE.md, what is worth saving: corrections you give it, traps in the repo that fail silently, and decisions that are not derivable from the code.

Then prune it. The index is capped at 200 lines. The kit includes one more skill for this, `/memory-vacuum`. It measures the memory directory, finds entries that are stale, duplicated, orphaned from the index, or already covered by the standing rules, and proposes a delete list. It deletes nothing until the human approves the list. I run it about once a week.

### 6. The kickoff prompt and the headless runner

This is the thing you type once. Most days it is a single command, `/orchestrate-work <TICKET-ID>`, and everything else comes from the ticket. The template below is what I write by hand when there is no ticket, and it is the shape the orchestrator uses when it briefs each worker:

```markdown
Goal: <one sentence, with a definition of done the agent can verify itself>
Ticket: <ID>
Worktree: <full path>
Decisions already made: <bullets, so the agent never asks>
When uncertain: state the assumption, continue, flag it in the report.
Write the plan to: ~/Downloads/<TICKET>-plan.md
Report format: findings by severity, then a Next block with only what a human must do.
Do not ask me anything unless the action is destructive or outward facing.
```

For a fully unattended run, wrap it in a script:

```bash
claude -p "$(cat ~/Downloads/AG-123-kickoff.md)" \
  --permission-mode acceptEdits \
  --permission-prompts none \
  --output-format json > ~/Downloads/AG-123-result.json
```

Codex users get the same thing with `codex exec`.

### Plus the gauge

Not one of the six, but the kit is not complete without it. In Claude Code the status line is a shell command in `~/.claude/settings.json` that gets session data as JSON on stdin, with the context numbers already calculated. The minimal version:

```json
{
  "statusLine": {
    "type": "command",
    "command": "jq -r '\"\\(.workspace.current_dir | split(\"/\") | last)  \\(.context_window.used_percentage // 0)% context\"'"
  }
}
```

Add a bar and color thresholds on top of that. Green under 50 percent, yellow under 80, red above. Put the context bar first and anything else, like cost or model name, after it.

That is the whole kit. Rules, hooks, skills, subagents, memory, a kickoff prompt, and a gauge to watch it all. Every one of them exists to move something out of the conversation and into a place the agent can reach without asking you.

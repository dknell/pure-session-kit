# The pure session: why the best agent run is the one you never talk to

> 🧮 **Cost to read this: about 4,600 words, or 20 minutes and a coffee.** ☕ Your agent does not read this file. It reads `BLUEPRINT.md`, about 6,500 tokens, and will be done before the coffee cools.

I have a rule for myself now when I hand work to a coding agent. Whether it is Claude Code, Codex, or whatever tool you use, the rule is the same: if I have to talk to the session after I start it, I did something wrong before I started it.

I call this a pure session. ✨ One goal, everything the agent needs loaded up front, and then I get out of the way.

This article is the system I use to make that work. It goes like this: why the context window matters and how to keep an eye on it, what a session needs before it starts, how to keep it small while it runs, why the ticket is the spine of all of it, and where the why lives so the rules file stays small. Then how to get all of it in your own repo: two prompts you paste into your own agent, one that cleans your current setup and one that builds the kit. The blueprint the agent builds from is a separate file in this repo, `BLUEPRINT.md`, for the agent to read, not you.

## Why the context window is the whole game 🧠

You have probably read plenty about context windows already. I am not going to break new ground here. But the whole system below rests on it, so here is the short version.

Every tool like this has a context window. Think of it as the agent's working memory for the session. Everything goes in there: the instructions, the files it reads, the output of every command it runs, and every message you send it.

It only grows. It never shrinks on its own. 📈

That matters because models get worse as that window fills up. Not just near the limit. Chroma published a study in 2025 called "Context Rot" that tested 18 frontier models, and every one of them got less reliable as the input got longer, with degradation showing up at sizes well below the advertised limit. Anthropic's own docs say the same thing in plainer words: as context fills, instructions from early in the conversation can get lost.

So here is the uncomfortable part. 😬 Every time you interject, you make the problem worse. You paste in a log, that is tokens. You paste a screenshot, that is tokens. You answer a question, the agent reads your answer and then runs three more commands to act on it, and all of that output lands in the window too. Your messages are usually small. The work the agent does in response to your messages is not.

The less you interact with a session, the smaller and cleaner its memory stays, and the more reliable it is by the end.

For what it is worth, here is where I draw the line. My window is 1M tokens and I never let a session get anywhere near it. I aim to stay under 20 percent. If a session crosses 30 percent, that is my signal that something should have been handled outside the session, and I usually end it and start clean rather than push on. The rest of this article is about what "outside the session" means.

Which raises the obvious question: how do I know what percent I am at?

## Keep the number on screen 📊

I have a bar at the bottom of my terminal that shows my context usage at all times. Repo name, a little progress bar, a percentage. Green while it is low, then yellow, then red. I glance at it the way you glance at a fuel gauge.

Most people do not have this. They start a session, work for three hours, and never once check how full the window is. Then the agent starts forgetting things and they blame the model.

You cannot manage what you cannot see. If your tool can show the number, put it on screen and leave it there.

In Claude Code this is one line. Type it and it writes the script and the settings for you:

```text
/statusline show the repo name and a context usage bar with the percentage, green under 50, yellow under 80, red above
```

Other tools vary. Some show a token count in their footer. Some only show it when you ask. If yours has nothing, make a habit of running the context command at every natural break, like after a plan is written or a test run finishes.

And if you plan to run the kit prompt at the end of this article, you can skip this step entirely. The kit includes the gauge, and it leaves yours alone if you already have one.

Now you can see the number. The rest of this is about keeping it low.

## What a pure session looks like 🎯

Before I start, I think about what the agent will need to reach the goal without me. Not what it needs to start. What it needs to finish.

That usually means:

**The goal is concrete.** "Complete this ticket" or "orchestrate this epic." Something with a definition of done that the agent can check itself.

**The standing rules are already on disk.** In Claude Code that is CLAUDE.md. In Codex it is AGENTS.md. This is where conventions live: branch naming, commit format, what never to touch, how to report back. The agent reads it at the start of every session, so I never have to explain it in chat.

**The decisions are already made.** If I know the agent is going to hit a fork in the road, I answer it in the prompt. And I tell it what to do when it hits one I did not predict: state the assumption, keep going, and flag it in the final report. An agent that stops to ask me a question is an agent that is now waiting on a human, and my answer costs context.

**The guardrails are mechanical, not conversational.** 🚧 Hooks are the big one here. A hook is a script that runs before or after a tool call and can block it. If the agent must never commit on main, I do not write "please don't commit on main" in the prompt and hope. I write a hook that refuses the commit. The agent does not have to remember the rule, and the rule does not take up space in its memory. It just fails closed.

**The skills load on demand.** In Claude Code, a skill is a folder with instructions for a specific workflow. At session start the agent only sees the name and a one line description of each one. The full instructions load only when the skill is actually used. So I can have seventy skills available and pay almost nothing for them until one is needed.

**The tools load on demand too.** MCP tool definitions are deferred by default now. The agent sees the tool names and loads the full schema only when it decides to call one. This used to be a huge silent cost. My setup has about 170 MCP tools available and they cost zero tokens until used.

For reference, my own sessions start at around 32k tokens with all of that loaded, out of a 1M window. But not all of that is mine. Running `/context` on a fresh session breaks it down:

| What | Tokens | Whose |
| --- | --- | --- |
| Claude Code system prompt | 5.3k | Built in, cannot change |
| Built-in tool definitions | 8.9k | Built in, cannot change |
| My CLAUDE.md files and memory | 12.9k | Mine |
| My custom subagent definitions | 2.7k | Mine |
| Skill names and descriptions | 2.3k | Mine, almost all of it |

So about 14k of the 32k is the harness itself. Every Claude Code session pays that whether you configure anything or not. The other 18k is what I actually put there to steer the run: the rules, the memory, the agents, the skills. That is the number I manage. It is under 2 percent of the window, and it is the most valuable 2 percent in the session, because it is the part the agent reads before it does anything else.

The goal is to keep the rest of the run from growing much past what the actual work requires.

## Ways to keep the window small during the run 🪟

Even with a good start, the agent will read files, run tests, and produce output. Here is what I use to keep that from piling up.

### Subagents 🐣

This is the single most useful technique. A subagent is a separate agent the main session spins up to do a task. It gets its own fresh context window. It does the research, reads the twenty files, runs the noisy commands, and then returns only a summary to the parent.

The parent never sees the twenty files. It sees the answer.

Claude Code and Codex both support this. In Claude Code you can define custom subagents with restricted tools, and the docs say it directly: each subagent starts with a fresh, isolated context window and returns only the summary. I build subagents into my workflows so the main session is mostly a coordinator that delegates the noisy parts.

### Branch and fork 🌿

Sometimes I do need to interject. I want to ask the agent a side question, or show it a log and get its read, without that exchange living in the main session forever.

Claude Code has `/branch` for this. It copies the conversation at that point into a new session and switches you into it. You ask your question, you get your answer, and the original session is untouched. You go back to it with `/resume`. There is also `/fork`, which copies the session into a background session that keeps working on its own, and a "fork" subagent type that inherits the whole conversation but keeps its tool calls out of the parent.

The idea is the same in every case. The side trip starts from the same knowledge the main session has, but nothing from the side trip flows back unless you choose to bring it.

### Put things in files, not in chat 📁

When I have a log or a screenshot the agent needs, I do not paste it. I drop it at a path and tell the agent where it is. Then the agent, or better yet a subagent, reads it and pulls out the five lines that matter.

Same for plans. I have my agents write the plan to a file on disk and report the digest. If I want the full plan I open the file. The session only carries the summary.

### Trim tool output ✂️

Tool output is the real context hog. A test suite that prints 4,000 lines of passing tests is 4,000 lines the agent has to carry for the rest of the session.

Two things help. First, teach the agent to run targeted commands: grep for the failure instead of dumping the whole log, pipe through head, ask for the count instead of the list. Put that in your CLAUDE.md. Second, use an output filter. I run a proxy called RTK in front of common commands that strips noise from git, test, and build output before the agent sees it. My savings on those commands run between 60 and 90 percent.

### Go fully headless 🤖

The purest session has no human at all. Claude Code has a `-p` flag for this: you pass the prompt on the command line and it runs to completion and exits. Codex has `codex exec` for the same thing. You can pipe input in, get JSON out, and run it from a script or CI.

If you are worried the agent will stop and ask something, Claude Code has a `--permission-prompts none` flag that removes the ability to ask entirely. Anything that would have needed a human is denied and the agent is told not to retry. That forces the discipline: if the run needs a human, the prompt was incomplete.

### One session, one goal, then throw it away 🗑️

I do not reuse sessions. When the ticket is done, the session is done. The next ticket starts fresh from a clean window with the standing rules loaded again.

The temptation is to keep going because "it already knows the codebase." It does not, really. It knows the pile of tool output from the last task, and that pile is now in the way. Git worktrees help here: one directory per task, one session per directory, no bleed between them.

### Memory across sessions 🧠

The thing you lose when you throw sessions away is what the agent learned about how you work. Claude Code has auto memory for this: a directory of small files the agent writes to as it learns your preferences and the traps in your repo. The index loads at the start of every session. So the lesson from Tuesday's session is available Thursday without Tuesday's 400k tokens of tool output coming with it.

## The recovery tools are not the plan 🧯

Claude Code has `/compact` to summarize the conversation and free space, `/clear` to start over, and a rewind menu to go back to an earlier point. Codex has compaction too. These are useful. I have used all of them.

But they are what you reach for after the window got messy. The whole point of a pure session is to not need them. Compaction is lossy. The docs say plainly that detailed instructions from early in the conversation may be lost. If your rules live in the conversation instead of in CLAUDE.md and hooks, compaction can quietly delete them.

## How to check yourself 🔍

The status bar tells you how full the window is while you work. `/context` tells you why. Run it in Claude Code at the end of a session. It shows you what filled the window: system prompt, tool definitions, memory files, messages, tool output. If messages and tool output dominate, ask what you could have front loaded, delegated to a subagent, or filtered.

The question I ask after every run is simple. What did I have to tell it that it should have already known? Then I move that thing into a file, a hook, or a skill, and the next session starts a little purer than the last.

## The ticket runs the whole thing 🎟️

One more piece before you build. I almost left it out because it felt like plumbing. It is not plumbing. It is the source of truth.

Every pure session I run starts from a ticket. The ticket holds the goal, the definition of done, the decisions already made, and the links to the PRs when they exist. The kickoff prompt is often just a ticket ID. The agent reads the ticket, cuts the worktree named after it, does the work, opens a PR that references it, and closes it on merge. My standing rules say every piece of work has a ticket and every PR references one. Hooks enforce the branch name matches the ticket ID.

That means the ticketing system is not a place you file things after the fact. It is the thing the agent reads first and writes to last. So it has to be fast, well organized, and cheap for an agent to read.

I was on Jira before this. I want to be fair, because Jira works fine for a lot of teams. But for agent driven work it was a context hog. Every call through the MCP server came back heavy, full of fields and nested objects the agent did not need, and that all landed in the window. Search was slow and vague. The agent could not reliably find a duplicate ticket or pull the one fact it needed out of a long one. And the data itself was messy in the way a decade of humans clicking around makes data messy.

I moved to [Linear](https://linear.app) and the workflow opened up. Responses come back small and structured. Search is fast and precise. The API is clean enough that the agent can create, update, link, and close tickets without me touching anything. It feels like a system that was built expecting an agent to be one of its users, and that is exactly what a pure session needs.

I am not saying you have to switch. I am saying the ticket is the spine of this whole approach, and if your agent spends 20k tokens just reading a ticket, you feel that in every session. Pick the system that gives the agent the least to carry.

## The vault: where the why lives 📚

The ticket says what to build. Something else has to say why things are the way they are. What the stack is. Why we picked Postgres over the other thing. How to roll back a deploy. What the coding standards are for each language in the repo.

If that lives in the standing rules file, the file gets huge and every session pays for all of it. If it lives in people's heads, the agent guesses. If it lives in a wiki the agent cannot search, same result.

So it lives in a vault. Plain markdown files, tracked in git, in their own repo that sits next to the code. Mine is an Obsidian vault, but you do not need Obsidian. You need markdown on disk that grep can search in milliseconds. It has one architecture doc that every session reads first, a folder of architecture decision records, a folder of runbooks, and a folder of standards. Every decision that matters has an ADR with a number. Tickets link to the ADRs they depend on. Code comments cite the ADR that explains a strange looking choice. PR bodies cite them too.

Two reasons it is a separate repo and not a `docs/` folder. PRs to the code repo stay lean, with no doc churn mixed into the diff. And one vault can serve several repos.

The agent does not load any of this at session start. It gets one pointer in the standing rules: here is the vault, read the architecture doc first, and if the vault is missing, stop. After that it searches the vault the way it searches code, with grep, and reads only the file it needs.

That is how my standing rules stay under 200 lines. Everything that would have gone in there went into the vault instead, where it costs nothing until it is needed.

A note on hosted tools. Notion is a fine place for humans to write docs. It is a poor place for an agent to read them. Every call through its MCP comes back heavy and the search is weak, so the agent spends tokens finding the doc instead of reading it. If your docs live there today, export them to markdown and let the vault be the copy the agent uses.

The kit builds this too. If you already have docs, it wires them in. If you have nothing, it creates the skeleton and writes a first draft of the architecture doc and the tech stack from your repo, marked as drafts for you to correct.

## Build your own Pure Session Kit 🧰

Everything above is something you can set up in your own repo. You do not build it by hand. Your agent builds it, from this article, in two prompts run in two separate sessions. Then you test it in a third. The exact spec it builds from is `BLUEPRINT.md` in this repo. You do not need to read it. Your agent does.

### Step 1: clean house first 🧹

Do not build the kit on top of a messy setup. If your memory files are stale, your instruction file is 600 lines, and you have four MCP servers you have not used since March, the kit inherits all of that on day one.

So the first prompt is an audit, not a build. Start a fresh session in the repo you care about and paste this:

```text
Audit my agent setup for this repository and recommend how to make it leaner. Do not change anything yet except the backup in step 1.

1. Back up first. Copy every file you are about to inspect into a timestamped folder outside this repo, and print the path. That includes my instruction files (CLAUDE.md, AGENTS.md, or your equivalent), rules directories, skills, subagent definitions, hooks and the settings that register them, MCP server config, and my memory directory for this project.

2. Identify which agent you are and check your own current documentation for how you load context at session start and what tools exist to measure it. Then measure it. Report how many tokens are in the window before I type anything, and how that splits between the parts I cannot change and the parts I own.

3. Inventory everything that loads at session start and everything that loads on demand. For each item give me its size and whether it is used. Flag:
   - Instruction files over 200 lines, or sections that describe things you could derive from the code itself.
   - Rules that could be path scoped but load unconditionally.
   - Memory entries that are stale, duplicated, contradict each other, or restate something an instruction file already says.
   - Skills and subagents that overlap, are never invoked, or would be better as a hook.
   - MCP servers whose tools are never used in this project, and whether their definitions are deferred or load in full.
   - Hooks that are missing for rules I currently enforce only with words.

4. Recommend the changes, ordered by how many tokens each one saves per session. For each: what to change, the file path, the estimated saving, and any risk. Group them into safe to apply now and needs my decision.

5. Stop and wait. I will tell you which ones to apply. When I do, apply them and re-measure so I can see the before and after.
```

Read the recommendations. Approve the ones you like. Let it apply them. Then close the session. You now have a clean floor to build on.

### Step 2: build the kit 🔨

After the cleanup, start a brand new session. 🧼 Not the cleanup session, and not the one you have been chatting in all afternoon. A fresh one, in the repo you want to set up, with nothing in the window but the standing rules. If you read this far and then paste the prompt into a session that is already half full, you have missed the point of the article. 🙃

Then paste this. It works in Claude Code, Codex, or any agent that can fetch a URL and write to your repo. The blueprint carries the detail. The prompt just points at it.

```text
Fetch and read this file in full: https://raw.githubusercontent.com/dknell/pure-session-kit/main/BLUEPRINT.md

It is the blueprint for a Pure Session Kit: a setup for running a coding agent from a ticket to a finished PR with no human interaction during the run. It defines seven pieces, which are a docs vault, standing rules, hooks, skills, subagents, memory, and a kickoff prompt with a headless runner, plus a context gauge.

Build me a Pure Session Kit for this repository.

- First identify which agent you are. The blueprint's examples are written for Claude Code. If that is you, use them. If you are Codex or any other agent, fetch your own current documentation before you write anything, and treat it as the authority over the blueprint's examples. Then, for each of the seven pieces: if you have a one to one equivalent, build it. If you do not, work out the proper way to get the same behavior in your tool and build that. If you cannot get the same behavior with confidence, do not improvise. Stop, tell me exactly where the gaps are and what your options are, and let me decide.
- Once you have that mapping, tell me what each piece maps to before you build it.
- Detect the environment exactly as the blueprint's "Detect your environment" section lists: version control and its host, the default branch, the ticketing system, the real test and build commands, the secrets tool, and the workspace. Report the full mapping before you write anything. Do not overwrite existing config. Extend it.
- Build all seven pieces the way the blueprint describes them, adapted to what you found here. The blueprint is the pattern. This repo is the truth.
- Name the skills exactly new-task, delegate, orchestrate-work, close-session, and memory-vacuum, and the subagents exactly researcher, log-triage, implementer, orchestrate-implementer, and reviewer, exactly as the blueprint names them. Build each one to the spec in the blueprint: same inputs, same steps, same outputs, same never rules. If your tool invokes skills differently, tell me the exact commands in your report.
- Check for a persistent context window indicator. If one is already configured, leave it alone. If not and your tool supports one, set it up as described in the blueprint's "Plus the gauge" section.
- Verify each piece: run every hook against one blocked and one allowed command and show the exit codes, list the skills and subagents to confirm they load, and report current context usage if your tool can show it.
- Beyond the gap check above, do not stop to ask me questions. If you are unsure about a small choice, make the sensible one, write it down as an assumption, and keep going.
- Report back with one line per file you created, every assumption you made, anything you could not build and why, and a Next block containing only what I have to do myself, with exact commands.
```

If the agent comes back with questions, that is your first data point. Whatever it asked, answer it in the prompt next time, and it will not ask again.

### Step 3: run one for real 🚀

Pick something small and real. Then use the skills the kit just built, and keep the two halves in separate sessions. Writing the ticket is one session. Doing the work is another.

Fresh session one:

```text
/new-task <a sentence or two about the change>
```

This is the one session where questions are the point. It reads the relevant code, then interviews you: goal, done criteria, edge cases, what is out of scope, anything touching data or security. Answer everything. It writes the ticket, checks that nothing is left to guess, creates it, and prints the ID. Close the session.

Fresh session two:

```text
/orchestrate-work <TICKET-ID>
```

That session becomes the orchestrator. It cuts the worktree, plans, shows you the plan, implements after you approve, self reviews in a fresh context, opens the PR, waits for you to merge, and cleans up. Your two touches are approving the plan and merging the PR. If a worker hits something unsafe or outside the ticket, it stops and shows you instead of pushing through. That is by design. Everything else, watch the gauge and keep your hands off the keyboard.

Where you had to step in, and where the number climbed, is what you fix next. If your agent is not Claude Code, the build report from Step 2 lists the exact commands to use instead.

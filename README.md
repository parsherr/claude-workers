# claude-workers

> A field journal, research archive, and experience log from someone who has been using Claude Code seriously for a long time.

This is not a library. Not a working codebase. It's a space where someone building a large software system's infrastructure with Claude Code records what they learned along the way — what they solved, where they got stuck, and what they saw coming.

If you take Claude Code seriously, there's something here for you.

---

## What You'll Find

### 🔬 Research (`researches/`)

Numbered research files, each chasing a concrete question. Format: `NNN-slug.md`.

Open questions currently in the backlog:

- How do you spawn Claude Code as a subprocess? (headless mode?)
- What's the lightest IPC mechanism between a worker and a manager process?
- Simplest persistence strategy for a task queue? (SQLite / JSON / Redis)
- How do you prevent race conditions across parallel workers?
- How should worker timeouts and watchdog logic be designed?

Research files are added as they're completed. The backlog stays live.

---

### 🪝 Hooks (`hooks.md`)

A full breakdown of the `check-completion.py` stop hook currently running in this environment.

What it does: whenever Claude finishes a response and is about to stop, the hook intercepts and asks — *"Did Claude actually finish the job, or did it half-complete something and wait for user approval?"*

If it half-completed → `exit 2` + stderr, forcing Claude to continue.

How it works:
1. Capture Claude's last message
2. Ask Haiku model "is this actually done?" (via Rusk proxy)
3. Regex fallback if the LLM call fails
4. MD5 loop guard to prevent infinite retry cycles
5. Decision: `exit 0` (allow stop) or `exit 2` (force continue)

This hook is the foundation of the whole worker agent system. It makes a world where an autonomous Claude cannot ask "should I continue?" — it either finishes or reports a failure.

---

### 🤖 Agent System

#### Kiwimi Workers (`kiwimi-workers/the-project-idea.md`)

Kiwimi is my software company. The idea taking shape here: agents with profiles — like football cards, or the general detail pages in strategy games.

Each agent has:
- A name, profile image, description
- A list of what it's good at
- A portable `.md` file you can drop into any project

You pick the agent you need and use it wherever.

#### Agent Generator (`agent-generator/core-idea.md`)

The construction floor for that idea. The plan:

1. Research what makes a good agent — produce a concrete ruleset
2. Research how to design a team of agents that communicate with each other
3. Write a global-scope agent generator agent (works across all projects)
4. When you need an agent, ask the generator — it produces the most optimal one, handles team configuration, outputs a finished agent file

---

### 📐 Work Management System (`work-management-system-session-notes.md`)

Conceptual design for an autonomous worker agent infrastructure. Task queue, worker registry, result store, parallel execution, work stealing.

Also contains engineering philosophy notes — what it actually means to build software that's still standing in 10 years.

---

## The Core Goal

```
User → Task Definition → Worker Agent → Completed Work
                               |
                    (zero approvals, fully autonomous)
```

Tasks enter a queue. Worker agents pick them up and run them. Every task either completes successfully or closes with a failure report. Multiple workers, parallel execution, work stealing.

Right now: pure planning and research. No code yet. Just ideas, notes, and open questions.

---

## How the Notes Are Organized

Notes are kept as an Obsidian vault. Files link to each other with `[[link]]` syntax.

```
hooks ←→ CLAUDE ←→ work-management-system-session-notes
           ↕
        researches
           ↕
   researches/NNN-slug

kiwimi-workers/the-project-idea ←→ agent-generator/core-idea
```

Every note has a connections line at the top. Navigable in Obsidian's graph view.

---

## Who This Is For

- People using Claude Code as a platform, not just a tool
- Anyone shaping Claude's behavior through the hook system
- People trying to build autonomous agent infrastructure
- Anyone asking not "how do I do this" but "why does this work this way"

There's no finished system here yet. But the thinking process, the research, and the hook solutions are already worth something. There's something being built on top of all this.

---

*This is a second brain. It stays live.*
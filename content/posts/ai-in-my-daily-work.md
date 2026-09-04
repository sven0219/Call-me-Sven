---
title: "How I Use AI in My Daily Work"
date: 2026-09-04 16:00:00+08:00
draft: false
author: "Sven"
summary: "After a year of daily AI use — 500+ sessions, 22,000+ messages — this is what actually worked: AI as a terminal operator, a two-tier model strategy, one-session-per-task discipline, sub-agent exploration, and the honest limits I hit along the way."
showtoc: true
tags: ["AI","Workflow","Terminal","Productivity","Debugging","Agent"]
Categories: ["AI","DevOps"]
---

# How I Use AI in My Daily Work

For the past year, an AI coding agent has been my default companion at the
terminal. When I opened my session history to write this, I found **500+
sessions and over 22,000 messages** scattered across dozens of working
directories. That number surprised even me. This post is a reflection on the
patterns that stuck, told generically — no projects, no companies, just the
workflow itself.

## 1. The starting point: AI in the terminal

I'm a DevOps engineer, which means a large part of my day is: look at a
cluster, read a config, run a command, observe the output, repeat. My tool
world lives in a terminal — `kubectl`, `git`, database clients, CI tooling,
cloud CLIs.

The breakthrough was realizing the agent can live *inside* that world, not
next to it. Instead of pasting errors into a browser tab, I ask the agent
directly. It runs the command, reads the output, greps the logs, and comes
back with an answer — or a follow-up command. The feedback loop that used to
cost me several copy-paste round trips collapsed into one conversation.

## 2. Pattern: AI as the terminal operator

The single most useful framing: **treat the AI as an operator of my machine,
not as a chat bot.** The agent has a shell. I describe intent in a sentence or
two, and it translates that into concrete commands.

A few real examples of requests I make daily:

- "Check the disk usage on these servers and tell me what's filling up."
- "Find every place this env variable is referenced across the repo."
- "Compare the config between the two environments and show me the diff."
- "This deployment didn't pick up the new image — find out why."

The key habit on my side: I stay in the loop. I read the diffs it produces,
approve each action, and verify the result afterward. The agent accelerates
the doing; the ownership stays with me.

## 3. Pattern: Two-tier model strategy

Over time I split my requests between two classes of model, and this
deliberately changed my cost and latency curve:

- **Fast / cheap model** — for lookups, routine edits, simple script changes,
  and anything where the answer is a known pattern. I batch these and move
  quickly.
- **Strong / slow model** — for genuine debugging, cross-repo analysis,
  tricky CI issues, and the rare "I have no idea what's happening" moment.
  This is where I'm willing to wait and spend.

The practical rule I developed: if I can predict the answer, use the fast
model. If I can't, use the strong one. This also keeps the expensive context
budget for the problems that actually need it.

## 4. Pattern: One session per task

My history shows a clear habit: **one session, one task.** Every task gets its
own session, often with a descriptive title like "Investigate why the cert
metrics went missing" or "Fix the flaky build step."

Why this matters:

- Context stays clean. Each session only carries the files and commands
  relevant to that one job.
- I can return to an unfinished session days later and immediately know where
  I left off — the title is the memory.
- It creates a natural audit trail. My session list is effectively a work log
  of everything I touched.
- It prevents cross-task contamination, where one conversation's assumptions
  leak into the next task.

## 5. Pattern: Sub-agents for exploration

For unfamiliar codebases I leaned heavily on parallel exploration agents.
Instead of dumping an entire repo into context, I dispatch a search agent:
"Explore the directory structure," "Find where the HPA is configured," "Look
for existing examples of this pattern."

This matters more than most people realize. Context windows are precious, and
burning them on directory listings wastes the budget that should go to actual
reasoning. A focused search returns only the relevant files, and the main
agent works from a small, high-signal set. My sessions show this became the
standard prologue before any non-trivial change.

## 6. Pattern: AI in incident response

The most visible win is in incident-style debugging. Production incidents are
uncomfortable — time pressure, noisy logs, and a sense of "where do I even
start." The agent changes this in a concrete way:

- **Parallel hypothesis testing.** While I reason about one theory, the agent
  checks another — different commands, different logs, no extra effort on my
  part.
- **Command recall under pressure.** I don't have to remember the exact flag
  for some rarely-used probe; I just describe what I need.
- **Systematic elimination.** It's very good at "if X is fine, then check Y"
  style narrowing, which is exactly what incident response is.

There are also plenty of "morning after" sessions — a 5xx spike overnight, an
alert that fired at 3 a.m. Being able to re-run the investigation quickly and
document the root cause is a quiet but real win.

## 7. Pattern: AI as the notetaker

A surprising amount of my usage has nothing to do with code. I keep a
markdown-based todo and work log, and the agent became its editor:

- Morning: "What's on my list today?" — it summarizes and plans.
- During the day: one-line updates appended to the log.
- Evening: "Summarize what got done and what's left."

The same pattern extends to runbooks and handover notes. Because the agent
records what actually happened (it saw the commands, the outputs, the fixes),
the notes it writes are far more accurate than the ones I used to write from
memory at the end of the day.

## 8. Pattern: AI beyond engineering

The same sessions cover things that have nothing to do with infrastructure:
formatting a legal document, drafting a request, proofreading an email,
writing an arbitration form, even helping review a novel draft. The
transferable skill is the same one I use for code — state the intent clearly,
give it the source material, and review the output with my own judgment.

The lesson: an agent isn't a tool for engineers, it's a tool for people who
can describe what they want. The more I practiced describing intent precisely,
the more useful it became outside my job description.

## 9. Where it still fails

Being honest about limits keeps expectations sane:

- **It guesses when uncertain.** Never take an answer at face value when it
  involves production data or destructive actions. I verify before I trust.
- **Long sessions degrade.** Reasoning quality drops as context grows. I
  learned to split work rather than force everything into one epic session.
- **It doesn't know my team's unwritten rules.** Organizational knowledge —
  who owns what, what the real SLA is, which corners are political — still
  has to come from humans.
- **Silly bugs still happen.** AI-generated changes need review exactly like
  human-generated ones. I read every diff, and I run the verification command
  every time.

## 10. Takeaways

If I had to compress a year of usage into a few lines of advice:

1. **Meet the tool where it lives.** Put the agent in your shell, not your
   browser.
2. **Be the reviewer, not the spectator.** Read diffs, verify results, stay in
   control of anything destructive.
3. **Split by task and by difficulty.** One session per task; fast models for
   the predictable, strong models for the unknown.
4. **Invest in context hygiene.** Explore with focused sub-agents, don't dump
   entire repos into the conversation.
5. **Let it keep your records.** If the agent performed the work, let it write
   the notes — they'll be accurate.
6. **Practice stating intent.** It's the one skill that transfers to every
   other use, code or not.

The numbers in my session history — 500+ sessions, 22,000+ messages — are
ultimately the story of a workflow becoming automatic. Not because I tried to
use AI more, but because it quietly became the fastest way to get my actual
work done.
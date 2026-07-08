---
name: tmux
description: "Read output from and send input to other tmux sessions/panes running on this machine (e.g. other Claude Code sessions the user has open), so agents can relay messages to each other without the user copy-pasting."
---

# tmux

Use this to interact with an existing, already-running tmux session/pane — for example another Claude Code session the user has open in a different terminal. Do not use this for one-shot local commands; that's what the normal shell is for.

## Find sessions

```bash
tmux ls
tmux list-windows -t <session>
tmux list-panes -t <session>:<window>
```

Target format is `session:window.pane`, e.g. `claude-peer-1:0.0`.

## Read what's currently on screen

```bash
tmux capture-pane -t <session>:<window>.<pane> -p
tmux capture-pane -t <session>:<window>.<pane> -p -S -   # full scrollback
```

Always capture and read before sending anything — check whether the target is idle at a prompt or mid-task, so you don't interrupt it.

**Reading the Claude Code TUI correctly:** the prompt line at the bottom of the pane (the input box area) often shows the *last submitted message* rendered in place (a placeholder), NOT unsent text sitting in the input box — you generally cannot tell the difference from a capture, so don't try to. Judge busy/idle only by activity indicators: a working/spinner status line and the words "esc to interrupt" in the footer mean busy; their absence means idle. Don't key off specific glyphs or prompt characters — the TUI's exact symbols change between versions. Text in the prompt line says nothing either way. If the pane looks idle, just send. Worst case the target really did have unsent input and the two messages concatenate — that's acceptable: no information is lost, and the receiving agent (or user) can untangle one merged message far more easily than they can recover a handoff that silently stalled. Observed firsthand: an agent wrongly concluded the user had unsent input and froze the handoff asking for permission to "clear" a line that was only a rendered placeholder — the stall was strictly worse than the concatenation it was avoiding.

## Send a message

Text and Enter are sent as separate commands to avoid paste/newline issues:

```bash
tmux send-keys -t <session>:<window>.<pane> -l -- "your message here"
tmux send-keys -t <session>:<window>.<pane> Enter
```

Special keys when needed: `C-c` (interrupt), `C-d`, `Escape`.

**Sending a message is a three-step action, not two: text, Enter, verify. Never end your turn after step 2.** Different harness TUIs handle a long pasted block differently: some submit immediately on Enter, others collapse it into a placeholder like `[Pasted Content N chars]` in the input box and need a **second, separate** Enter to actually send it (observed firsthand with Codex's CLI — a closing message sat unsent in its input line because this step was skipped). Treat "did it actually go through" as part of the send itself, in the same batch of actions, not a follow-up you might get to:

1. Send the text (`-l --`).
2. Send `Enter`.
3. Immediately `capture-pane` and check: if the message still sits in the input line (as literal text or a pasted-content placeholder) rather than appearing in scrollback as a sent message, send `Enter` again and re-check.

Don't fire text + Enter + Enter every time as a blind habit either — an extra stray Enter on a harness that already submitted on the first one can submit an empty follow-up turn. Check, then act on what you see; but always check.

## Notes

- Sending into another live session is a real interruption for whatever is running there — capture the pane first, and only send when it's actually idle/waiting for input, unless the user explicitly wants it interrupted.
- Sessions persist across SSH disconnects, so a session being unattached doesn't mean nothing is running in it.
- This only lets you talk to sessions on this same machine. It doesn't create new sessions or install anything.

## Agent-to-agent messages

When the user asks you to coordinate with (not just relay a one-off note to) another Claude Code session running in tmux, tag what you send so the receiving agent knows it's from a peer agent, not the user typing directly, and knows it's allowed to send a reply back the same way:

```bash
tmux send-keys -t <session>:<window>.<pane> -l -- "[from-peer:<your-session-name>] <message>"
tmux send-keys -t <session>:<window>.<pane> Enter
```

**`<your-session-name>` must be your own actual tmux session name, verified, not invented or guessed.** Get it with `tmux display-message -p '#S'` before your first send — don't derive it from the project directory, the task at hand, or what sounds plausible. A wrong self-tag is indistinguishable from impersonating a different agent, and breaks the receiving agent's ability to look you up or route a reply. Observed directly: a session actually named `claude-peer-1` tagged its outgoing messages `[from-peer:claude-peer-2]` — a name it never checked — which the user flagged as looking like impersonation.

If a message you read via `capture-pane` starts with `[from-peer:<name>]`, it's from another agent, not the user. You may reply directly using the same mechanism, tagged with your own session name.

**Explicitly ask for a reply when the peer finishes — don't assume it'll spontaneously send one.** A peer agent that completes a task has no inherent reason to type a message back into its own pane; replying to *you* specifically isn't something it does by default just because it's done. State it as part of the request itself, e.g. "...reply back in this pane with DONE/BLOCKED when finished, even if it's just a one-liner — I'm waiting on that before I continue." Without this, you can be left polling a pane that's genuinely finished but silent, unable to tell "done, nothing to report" apart from "still working." This was observed directly: a peer agent finished its task but never sent a completion reply, leaving the initiating agent to repeatedly re-check the pane instead of getting notified.

**Receiving side of the same rule: if a peer asked you to do something, report back when you finish — unprompted.** Completing peer-requested work and going silent is a protocol bug, even if your own user can see the result: the peer is a separate session that cannot see your pane's outcome unless you send it. As soon as the requested work reaches a terminal state (done, blocked, or obsolete), send the peer a `DONE:`/`BLOCKED:` message yourself — don't wait for your user to say "so tell them". Observed firsthand: an agent finished a peer-requested commit+push, summarized it to its own user, and never messaged the peer until the user explicitly ordered it.

**If you hit a blocker mid-coordination (permission denial, classifier refusal, missing access), escalate within ~2 attempts — don't grind.** One retry for a transient error is fine; after that, the blocker is the news. Immediately (a) tell your user plainly what's blocked and what you need, using `PushNotification` if available rather than assuming they're watching, and (b) send the peer a `BLOCKED: <reason>` so it isn't left polling. Observed firsthand: an agent spent many minutes cycling permission-denied commits while both the user and the waiting peer had to dig in manually to find out why nothing was moving.

**Before the first message, set your own budget out loud** (to the user, in your normal response, not to the other agent): estimate how many round trips this actually needs given the task's complexity, e.g. "this is a two-step handoff, budgeting ~3 messages" or "this needs back-and-forth debugging, budgeting ~10". A trivial relay is 1-2. A real coordination task might be 8-15. There's no single correct number — the point is to commit to an estimate instead of going in unbounded, and to notice when you've blown past it.

**Stop and report back to the user when any of these hit, whichever comes first:**
- The other agent (or you) signals completion — end your final message in the exchange with `DONE:` or `BLOCKED:` followed by a one-line reason, so it's unambiguous to both sides and to anyone reading the pane later.
- You've used your stated budget and the task isn't done — re-estimate once, out loud, or stop and hand it to the user. Don't silently extend a second time.
- **No-progress / loop detection**: if your last message and the other agent's reply are substantively the same as the previous round (rephrased but not new information, or you're both just re-asking the same thing), stop immediately regardless of budget remaining — this is the failure mode budgets alone don't catch.
- Hard ceiling regardless of estimate: **20 messages sent by you**. If you're still going at 20, something is wrong with the coordination itself, not just the estimate — stop and report, don't renegotiate the ceiling.

An open-ended instruction like "coordinate until it's done" sets the goal, not a licence to skip the budget/estimate step or the hard ceiling. State your estimate, work to it, and come back to the user at whichever stop condition triggers first — two agents running on each other burns real API cost and can drift without either side noticing.

## Talking to a peer that may not know this protocol

Any session on the other end — a different harness (Codex, a plain shell) or even another Claude Code session that never loaded this skill — may not recognize the `[from-peer:...]` tag or the DONE/BLOCKED/NEEDS_HUMAN conventions. Never assume the peer knows this protocol just because you do, and never assume it doesn't just because it's a different tool — check instead of guessing either way.

- On the **first** message to a session you haven't confirmed the protocol with, include a short one-line explanation of the tag and the expected reply format, in plain language, not just the bare tag — e.g. `[from-peer:claude-peer-2] (heads up: this is an automated message from another AI agent in a different, independent terminal session on this same machine — not the user, and not a subagent you spawned; it's a separate peer session, so reply the same way in this pane rather than trying to address it through any child-agent messaging tool you have. Please send a reply back in this pane when you're done or blocked — even a one-liner — since I'm waiting on that to continue, not just inferring it from silence.) <actual message>`.
- Be explicit that the sender is an independent peer, not a subagent of the receiver. A harness whose own orchestration tooling defaults to a vertical mental model ("an agent messaging me must be one I spawned") may otherwise try to route a reply through its own child-agent addressing mechanism, fail to find a matching agent ID, and only fall back to answering in the pane after that internal attempt errors out. This was observed directly: a receiving Claude Code session tried to reply via its own subagent-messaging tool using the sender's tmux session name as if it were a spawned agent's ID, got "No agent ... is reachable," and only then replied in plain text in the pane.
- Read the reply before sending anything further. If it responds naturally to the content, treat the protocol as understood from here on for that session and drop the explanation from later messages. If it seems confused or replies as if a human wrote to it, restate more plainly rather than repeating the same tag louder.
- This makes the protocol self-bootstrapping — every new kind of peer gets a one-time inline explanation instead of requiring the user to install anything on their side first.
- The inline explanation only bootstraps the *current* exchange, for the *current* session. It does not persist: it's not written anywhere, and the peer's next fresh session won't have seen it. If you want the peer to actually recognize this protocol going forward, that's a separate, bigger action — copying this SKILL.md (or an equivalent) into the peer's own config — and it should not happen on an agent's own initiative just because it seems convenient. Offer it, and let a human decide, the same as any other file write into a system that isn't yours. (Observed in practice: the working copy at `~/.agents/skills/tmux/SKILL.md` for a Codex peer was created because the user explicitly told that Codex session to copy it, not because either agent decided to on its own.)
- Persisting the skill file for *future* sessions does not retroactively give the *current* session the skill — most harnesses only load skills at session start. Picking it up requires restarting that session, and an agent cannot restart itself (doing so kills its own process before it could resume). If continuity matters, say so explicitly and let the human restart it; don't assume the peer "now has" the skill just because the file exists on disk.
- Resist the temptation to self-restart via a detached delayed kill (`nohup`/`disown`/`at`/`systemd-run` scheduling your own process's death and hoping something brings it back). This only works if an external supervisor already restarts the process on exit, which is not the case for a plain `tmux new-session` launcher — killing yourself with no supervisor is permanent, not a restart. It is also, independently, an agent scheduling its own irreversible termination without asking, which the human should decide, not the agent. Just tell the human to restart the session.

## Never impersonate the user for a permission prompt

If you capture a pane and see the *harness itself* (not the peer agent) asking for human confirmation — a permission dialog, a "y/n to proceed", an approval gate for a sensitive/irreversible action — that confirmation is addressed to the user, not to you, even though you have the same keystroke-level ability to answer it that the user does. Having the technical capability to press "y" is not the same as having the authority to.

- Do not answer it, on the peer's behalf or otherwise, even if you're confident what the answer "should" be. Doing so is impersonating the user to their own tooling, which defeats the entire reason that confirmation gate exists.
- Treat this the same as a `NEEDS_HUMAN` case: stop, don't send anything into that pane, and notify the user (with `PushNotification` if available) that session `<name>` is blocked on a permission prompt addressed to them, quoting what it's asking.
- This is different from replying to a question *the peer agent* asked you in its own words as part of the coordination (that's normal agent-to-agent traffic) — the distinction is who the prompt is addressed to, not who technically typed it.
- This obligation to notify does not depend on the peer knowing this protocol. A peer with no `tmux` skill of its own has no way to raise `NEEDS_HUMAN` or reach a notification tool — it may not even register that it's stuck on something the operator needs to see. That doesn't reduce your obligation, it's the whole reason you're watching: you notify because *you* can see the prompt, regardless of whether the session stuck on it has any concept of this protocol at all.

## Monitoring a peer session for problems

If you're coordinating with a peer agent on a task that takes a while, don't just fire a message and disappear — periodically capture its pane to check it isn't stuck: an unhandled error, a permission prompt (see above), a crash back to shell, or genuine no-progress (same state across two checks a reasonable interval apart). If you find it stuck on something that isn't a human-permission prompt, it's fine to point it out or ask what's going on, same as any other agent-to-agent message — the escalate-to-human rule above is specifically about prompts addressed to the user, not about noticing the peer is stuck in general.

## Escalating to the human (consensus → notify)

Watching the tmux pane is a passive form of human-in-the-loop — it only works if the user happens to be looking. For anything that actually needs a decision only the user can make, don't rely on that. Use an explicit signal instead:

- Either agent can raise `NEEDS_HUMAN: <one-line reason>` in a message when it's genuinely unsure and a second opinion from the peer agent doesn't resolve it.
- If the other agent independently agrees it's also unsure (or agrees the question is outside what either of you should decide alone) — that's consensus. Don't keep deliberating past that point hoping to resolve it yourselves; two uncertain agents agreeing they're uncertain is not progress.
- Once there's consensus on `NEEDS_HUMAN`, whichever agent has a notification tool available (e.g. `PushNotification`) uses it to actually alert the user with the reason and what's blocked — don't just leave it sitting in the pane. That turns this from "the user might notice" into a real interrupt.
- After sending the notification, stop and wait. Don't keep messaging the other agent while waiting for the human — that's the same no-progress failure mode as above, just dressed up as "waiting for consensus on what to do while we wait."

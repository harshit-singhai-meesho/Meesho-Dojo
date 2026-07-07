---
name: golden-set
description: Add a question to the Meesho Memory / UCL evaluation golden set, posted to #ucl-developers for crowd-voting on which docs the answer should come from. Use this skill whenever the user says any of "add this to the golden set", "add to eval", "save this question for eval", "add this question to ucl-developers for voting", "golden set add", "/golden-add", "this should be an eval question", "submit this for golden set voting", "/eval-question". Also trigger after the user has just asked a question through Meesho Memory / UCL and indicates they want that exact question evaluated (phrases like "make this an eval question", "send this for crowd voting", "let's grade this", "I want the team to vote on this one"). The skill drives the eval-agent CLI under `eval/agent/cli.py`: it picks the user's question, looks up their contributor handle, runs `python -m eval.agent.cli ask`, surfaces the Slack message that would be posted to #ucl-developers, and (in simulation mode) lets the user demo the voting + finalisation flow without a live workspace. It does NOT score answers or run the eval harness — for that, point the user at `eval/run_eval.py`.
---

# golden-set

You drive the **golden-set agent** at `eval/agent/`. Its job: turn a Meesho Memory query into a crowd-voted entry in `eval/dataset/golden_v1.jsonl`.

This skill is a thin wrapper around the agent CLI — the real logic lives in `eval/agent/handlers.py` and `eval/agent/vote_aggregator.py`. Your job here is to:

1. Figure out what the **question** is.
2. Figure out who the **contributor** is.
3. Decide if it should be added in **live** mode (real Slack to `#ucl-developers`) or **simulation** mode (mock Slack, stdout).
4. Run the right CLI command and show the result.

If unsure, default to **simulation mode** — it's safe, reversible, and doesn't spam the channel.

---

## When to use

Trigger this skill on any of:

- *"add this to the golden set"* / *"add to eval"* / *"send this for crowd voting"*
- The user types `/golden-add` or `/eval-question` (informally — there's no real slash command)
- The user just asked a Meesho Memory question in this session and wants to evaluate that exact query
- The user pastes a question they want labelled by the team
- The user asks a question to Meesho Memory using /meesho-memory

Don't trigger for:

- Running the eval harness itself ("run the eval"). That's `eval/run_eval.py`, not this skill.
- Scoring an existing question. This skill adds questions, doesn't grade.
- Editing or removing already-finalised questions in `golden_v1.jsonl`. Point the user at the file directly.

---

## Inputs you need

| Input | How to get it |
|---|---|
| `question` text | The exact question the user wants evaluated. If they ran one through Meesho Memory just now, **reuse it verbatim**. Otherwise ask. |
| `contributor` Slack ID | Look in `MEMORY.md` for the user's email/Slack ID; if absent, ask. Use the short Slack handle (`U_HARSHIT`) not the email. Save it to memory for next time. |
| `difficulty` (optional) | `easy` / `medium` / `hard`. Skip unless the user mentions it. |
| `should-refuse` (optional) | Flag only if the user explicitly says the wiki shouldn't be able to answer it. |
| Mode | **simulation** by default. Switch to **live** only if `SLACK_BOT_TOKEN` is set AND the user explicitly says "send for real" / "post live". |

---

## Workflow

### Step 1 — Confirm the question (one sentence to the user)

State the question you're about to add, in plain quotes. If you're reusing the user's previous Meesho Memory query, say so:

> "I'll add this to the golden set: *\"How has the CPDO target evolved across our last three planning cycles?\"* — contributor `U_HARSHIT`. Simulation mode (Slack token not set). OK to proceed?"

If they correct anything, update and re-confirm in one line. Don't over-confirm — one back-and-forth is enough.

### Step 2 — Run the ask command

Use Bash:

```
python3 -m eval.agent.cli ask "<question>" --user <contributor> [--difficulty <level>] [--should-refuse]
```

Working directory: the repo root (`/Users/HarshitSinghai/Documents/RAG-OPENAI/openai-knowledge-retrieval/`). The CLI auto-creates `eval/agent/state.sqlite` if missing.

### Step 3 — Show the user what got posted

The CLI prints the Slack message inline (via `MockSlackClient`). Surface:
- The minted `qid` (e.g. `Q052`).
- The 10 candidate docs the bot suggested.
- Where the state was saved.

Format as a tight summary. Don't paste the whole Slack message unless the user asks for it — point at the trace and offer to expand.

### Step 4 — Offer next steps (only if the user is exploring)

If the user seems to be demoing the agent (not just adding one real question), offer the **voting walkthrough**:

> "Want me to simulate 3 votes + a star to finalise this? That'll show you the full flow end-to-end."

If yes, drive these commands (one per Bash call, or chained with `&&`):

```
python3 -m eval.agent.cli vote <qid> <doc_id> yes --user U_AMIT
python3 -m eval.agent.cli vote <qid> <doc_id> yes --user U_PRIYA
python3 -m eval.agent.cli vote <qid> <doc_id> yes --user U_RAVI
python3 -m eval.agent.cli vote <qid> <doc_id> star --user U_AMIT
python3 -m eval.agent.cli tick
```

After the `tick`, show the user the resulting JSONL line from `eval/dataset/golden_v1.jsonl`.

---

## Helpful follow-up commands the user might want

| What they ask | Run |
|---|---|
| *"What's pending?"* | `python3 -m eval.agent.cli list` |
| *"Show me Q051"* | `python3 -m eval.agent.cli show Q051` |
| *"Add a doc the bot missed"* | `python3 -m eval.agent.cli thread-add <qid> "add: <path>" --user <handle>` |
| *"Cast a no vote"* | `python3 -m eval.agent.cli vote <qid> <doc_id> no --user <handle>` |
| *"Finalise stale questions"* | `python3 -m eval.agent.cli tick` |
| *"Pretend 50 hours passed"* | `python3 -m eval.agent.cli tick --simulate-age-hours 50` |
| *"Reset state, start fresh"* | `python3 -m eval.agent.cli reset --also-golden` |

---

## Switching to live Slack (only when explicitly asked)

Don't post to real Slack unless the user says "post live" or "send it for real" AND the env is configured:

1. Verify both env vars are set:
   ```
   echo "${SLACK_BOT_TOKEN:0:5}... ${SLACK_SIGNING_SECRET:0:5}..."
   ```
   If empty, tell the user to set them and stop.
2. Confirm the channel: default is `#ucl-developers` (per `UCL_EVAL_CHANNEL`). State it explicitly before running.
3. Run the same `ask` command. With the token set, `RealSlackClient` is used automatically — same code path, real Slack messages.
4. After: confirm what was posted, give the user the `qid`, and note that the bot's tick loop runs every 10 minutes when `eval/agent/server.py` is running.

---

## Edge cases

### The user's question is ambiguous or empty
Don't run anything. Ask one clarifying question. Drop the skill if they can't supply a question — adding a `""` question to the golden set wastes everyone's voting effort.

### The user already has Q-id pending for the same question
Run `python3 -m eval.agent.cli list` first. If a near-identical question (≥70% token overlap) is already pending, point at it instead of creating a duplicate. Suggest they `show <qid>` and vote on it.

### `_search.json` is missing
The candidate picker depends on `final wikis/_search.json`. If absent, the user needs to (re-)run Stage G via the wiki pipeline. Surface the error clearly; don't hide it.

### State DB locked
If `state.sqlite` is locked (e.g. server.py is running concurrently), wait 1s and retry once. If still locked, tell the user and stop.

### Save the contributor handle to memory
After a successful first run, write a small auto-memory entry so you don't ask for the Slack ID again:
- Memory file: `~/.claude/projects/-Users-HarshitSinghai-Documents-RAG-OPENAI-openai-knowledge-retrieval/memory/user_slack_id.md`
- Type: `user`
- Body: "User's Slack ID for the UCL eval agent is `<U_X>` (e.g. `U_HARSHIT`). Use as `--user` when running `eval/agent/cli.py`."

---

## Reference

- Agent code: [eval/agent/](../../Documents/RAG-OPENAI/openai-knowledge-retrieval/eval/agent/)
- Agent docs: [eval/agent/README.md](../../Documents/RAG-OPENAI/openai-knowledge-retrieval/eval/agent/README.md)
- Channel: `#ucl-developers` (override with `UCL_EVAL_CHANNEL`)
- Quorum: 3 ✅ votes per doc (override `UCL_QUORUM_YES`)
- Timeout: 48 h (override `UCL_TIMEOUT_HRS`)
- Where finalised Qs land: `eval/dataset/golden_v1.jsonl`

If you find yourself wanting capabilities this skill doesn't have — re-ranking by votes, dedup across pending Qs, multi-question batch import — those are agent features, not skill features. Add them to `eval/agent/handlers.py` and update this SKILL.md to match.

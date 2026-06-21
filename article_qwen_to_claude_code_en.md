# How One Agent Failure Pattern Revealed Claude Code's Hidden Design Principle

## The 60 seconds that made me read Claude Code's source

Alibaba, AI Coding evaluation. The agent under test was Qwen (通义千问). The task was mundane: hand it a file path, watch it read and edit code. What I saw next is the cleanest example of "agent without session memory" I've ever observed in the wild.

I gave Qwen a file path. It called `Read`. Errored. Retried. Errored. Retried a third time. Errored. Then Qwen said, "Read seems broken, let me try `cat`." `cat` worked.

So far, so reasonable. A three-strikes-then-fallback pattern is fine.

Then I gave it **another** file path. Same session. Same conversation history visible to the model. Qwen started from step 1: called `Read`, failed, retried, failed, retried, failed, fell back to `cat`. Every new path triggered the full six-step cycle. The model could see all of its previous `Read` failures sitting in the message history. It did not generalize.

That moment — "the evidence is literally in your context window and you still won't use it" — is what sent me into Claude Code's source code (available through the published npm package bundle) looking for how a well-designed agent handles the same situation.

## Two failures in one

The interesting thing about Qwen's behavior is that it's not one bug. It's two failures stacked, at two different layers.

**Runtime layer: no session-scoped "this tool is unreliable" state.** Qwen clearly has a per-turn retry counter — that's why it stops after three attempts. But the counter resets on every new turn. There is no session-level data structure tracking tool reliability that persists across user messages.

**Model layer: no in-context generalization from past failures.** Even with all the failed `Read` tool_results sitting in the message history, the model treats each turn as a fresh attempt. This isn't a prompt engineering problem you can fix with one more system-prompt sentence. It's a training distribution issue — the model wasn't shaped to extract "this tool keeps failing, avoid it" as meta-knowledge from prior tool_results.

To recover gracefully from session-internal failure, an agent has to plug both holes. Qwen plugs neither. Claude Code plugs both, with two separate mechanisms that map cleanly onto these two layers.

## Defense 1: an explicit, immutable failure state machine

File: `src/utils/permissions/denialTracking.ts` — the whole file is 46 lines.

Lines 12-15 define two thresholds:

```ts
export const DENIAL_LIMITS = {
  maxConsecutive: 3,
  maxTotal: 20,
} as const
```

`maxConsecutive: 3` is a **recoverable** counter — it resets to zero on success (see `recordSuccess` at lines 32-38). `maxTotal: 20` is **unrecoverable** — it only ever increases.

The trigger function (lines 40-45):

```ts
export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}
```

Two design choices matter more than they look. First, the fallback target is **the human**, not "another tool". When the agent has hit the same wall enough times, the right move is to admit defeat and ask, not keep guessing tools. Second, all state transitions are immutable — every update function returns `{ ...state, ... }`. Every transition is auditable and reproducible from the recorded sequence.

This is the runtime defense. It handles **burst failures within a session** — fast, deterministic, fully in code. But it does not let the agent carry lessons from one task to the next. For that, you need a second mechanism.

## Defense 2: a markdown template that captures lessons

File: `src/services/SessionMemory/prompts.ts`, lines 11-41.

`DEFAULT_SESSION_MEMORY_TEMPLATE` is just a markdown document with sections. Three of those sections are directly relevant to the "tool keeps failing" problem:

```markdown
# Workflow
_What bash commands are usually run and in what order?
How to interpret their output if not obvious?_

# Errors & Corrections
_Errors encountered and how they were fixed. What did the user correct?
What approaches failed and should not be tried again?_

# Learnings
_What has worked well? What has not? What to avoid?_
```

The italicized lines aren't documentation. They're prompt instructions baked into the template. When the session memory gets updated, an extractor agent reads the conversation and fills in content under each header based on what those italic lines ask for.

If you ran that Qwen session through this mechanism, the "Errors & Corrections" section would end up with something like: "Read tool consistently fails on file paths; use `cat` instead." Next turn loads that note as context. The six-step failure cycle vanishes.

## The part that surprised me: no special code

I expected the extractor to contain dedicated logic for spotting unreliable tools — pattern matching on repeated tool_result errors, a heuristic for promoting them to "Errors & Corrections". I was wrong.

Look at `buildSessionMemoryUpdatePrompt` at lines 226-247:

```ts
export async function buildSessionMemoryUpdatePrompt(
  currentNotes: string,
  notesPath: string,
): Promise<string> {
  const promptTemplate = await loadSessionMemoryPrompt()

  const sectionSizes = analyzeSectionSizes(currentNotes)
  const totalTokens = roughTokenCountEstimation(currentNotes)
  const sectionReminders = generateSectionReminders(sectionSizes, totalTokens)

  const variables = { currentNotes, notesPath }
  const basePrompt = substituteVariables(promptTemplate, variables)

  return basePrompt + sectionReminders
}
```

Three things: load the template, compute section sizes and emit budget reminders, substitute `{{notesPath}}` and `{{currentNotes}}`. That's it. **Nothing** in this function (or anywhere else in the module) detects "tool reliability". The extractor is a forked agent restricted to the Edit tool on the memory file, and it makes those judgments at runtime, guided only by the italic prompt lines in the template.

## The hidden principle: schema-driven extraction

Let me name the pattern: **schema-driven extraction**.

- **Structure drives behavior.** The markdown section headers and their italic descriptions *are* the spec for what the extractor agent does.
- **No subsystem.** There is no "ToolReliabilityTracker" class, no failure-classification rules, no decision matrix.
- **The lighter approach generalizes better.** A traditional approach — building an explicit tool-reliability state machine on top of `denialTracking` — would have to enumerate failure types at design time. Schema-driven extraction can capture failures the designers never imagined, because the schema doesn't say "track tool failures"; it says "reflect on what failed".

The trade-off is real. You're delegating judgment to a model, so the guarantees are softer. The extractor might miss a lesson or summarize one badly. But in exchange you get coverage of failure modes you didn't anticipate when you wrote the template — and you spend a few dozen lines of markdown instead of a few hundred lines of TypeScript.

That's why the lighter approach wins for open-ended behavior. The structure of the schema *is* the architecture.

## The two-layer defense

| Layer | Mechanism | What it catches | Qwen has it? |
|---|---|---|---|
| Runtime | `denialTracking` state machine | Burst failures inside a session, immediate fallback to asking the human | No |
| Schema | Session memory markdown template | Long-term lessons, reloaded as context next turn | No |

Defense 1 handles **now**: trip a counter, stop, ask. Defense 2 handles **next time**: structure today's failures so tomorrow's prompt context already includes them. Qwen is missing both, which is why every new file path triggered the same six-step cycle.

## What to take from this if you're building agents

Three concrete moves:

**1. Build a tiny, immutable state machine for known critical failures.** Don't reach for an in-memory `Map` you mutate. Make every transition return a new state. Pick thresholds with both a recoverable counter (resets on success) and an unrecoverable one (only increases). When either trips, fall back to **asking the user**, not to a guessed alternative tool. Forty-six lines is enough.

**2. Use schema-driven extraction for everything you can't enumerate.** Write a markdown template with sections like "Errors & Corrections", "Learnings", "Workflow". Put the extraction instructions into italic descriptions under each header — these are prompts, not docs. Fork a restricted agent (Edit tool only, scoped to one file) to fill it in. Let the model decide what's worth recording.

**3. Reload the schema next turn.** None of the above matters if the lessons stay in a file the agent never sees again. Make session memory part of the next turn's context window.

I don't think Qwen's behavior reflects on the competence of the team that built it — it looks more like a different set of design priorities and a different training distribution. But as a developer watching the test, the failure was a gift. It gave me a concrete counterexample that told me exactly what to look for when I opened Claude Code's source.

The pattern I found there — runtime state machine plus schema-driven extraction — isn't documented anywhere as a named principle. It's just two files quietly cooperating. But once you see it, you can apply it to any agent failure-recovery problem: API calls, database queries, external commands, anything where the agent will sometimes hit a wall and needs to remember it.

Read the source. It's a quiet education.

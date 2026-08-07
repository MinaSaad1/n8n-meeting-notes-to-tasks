# Architecture

## High-level

```
Webhook (POST /webhook/meeting-notes)
        |
        v
Set node ... normalizes { transcript, meeting_title, meeting_date }
        |
        v
chainLlm + Anthropic chat model ... extraction prompt returns structured JSON
        |
        v
Code node ... strips markdown fencing, parses JSON, falls back gracefully
        |
        +--> Notion (Meeting Summary Page) ... one row per meeting
        +--> splitOut (Action Items) --> Notion (Create Task) ... one row per action
        +--> Slack (Meeting Recap) ... posts summary + tasks list
                       |
                       v
              respondToWebhook ... { received, tasks_created }
```

The workflow has three downstream branches off the parsed payload: a single Notion meeting page, a fan-out into Notion tasks, and a Slack message. They run in parallel. Only the meeting-page branch is wired into the webhook response, which is enough to confirm the workflow ran end-to-end.

## Components

### Meeting Transcript Webhook (`webhook`)

POST endpoint at `/webhook/meeting-notes`, `responseMode: responseNode`. The body is expected as JSON with three optional fields:

```json
{
  "transcript": "string, the raw meeting transcript",
  "meeting_title": "string, optional, defaults to 'Meeting'",
  "date": "ISO date string, optional, defaults to today"
}
```

The `responseNode` mode is what lets the workflow run all three branches in parallel and still return one structured response at the end via the dedicated `Webhook Response` node.

### Extract Meeting Fields (`Set` node)

Pulls the three fields off `$json.body` and assigns sensible defaults (`'Meeting'` for the title, `$now.format('YYYY-MM-DD')` for the date). Naming the fields here once means the rest of the workflow doesn't dig into `body.*` repeatedly, which keeps the downstream expressions readable and lets you tweak the request shape without rewriting every node.

### Claude Extraction Model (`@n8n/n8n-nodes-langchain.lmChatAnthropic`)

The chat model node, configured with `model: claude-sonnet-4-6` and `maxTokens: 2000`. It feeds the chainLlm node beside it via the `ai_languageModel` connection. Swap to `claude-sonnet-4-6` once your Anthropic account has access, no other changes needed.

The model is intentionally separate from the chain node so you can change provider (OpenAI, Gemini, OpenRouter) without rewriting the prompt.

### Extract Actions and Decisions (`@n8n/n8n-nodes-langchain.chainLlm`)

The chain node holds the prompt. The prompt asks Claude to return a single JSON object with five fields: `summary`, `decisions`, `action_items`, `attendees`, and per-item `{ task, owner, due_date, priority }`. The instruction "Return only valid JSON. No markdown fencing." reduces parse failures, and the downstream Code node still defends against the times Claude wraps output in ```json fences anyway.

The prompt is interpolated with the title, date, and transcript from the previous Set node. Keeping the prompt inside the chain node (not in a separate Set node) means the prompt and the model are co-located when you tune them.

### Parse Meeting Data (`Code` node)

This is the safety net. Reads `text` from the chain node output, strips ```json fences with a regex, then `JSON.parse`. If parsing throws, it returns a degraded payload (`summary` becomes the raw text, the arrays become empty) so the rest of the flow still completes. The fields from Extract Meeting Fields get spread back in, so downstream nodes see one merged object with the meeting metadata plus the parsed structure.

The fallback matters. Without it, one bad LLM response would bork the Notion writes, the Slack post, and the webhook response, and the caller would get a 500 with no insight into what happened.

### Notion Meeting Summary Page (`Notion`)

Creates one page in your meeting-notes database with the title, date, summary, and attendees. The integration must be invited to that specific database. The title is templated as `{{ meeting_title }} - {{ meeting_date }}` so multiple meetings on the same day stay distinguishable.

This branch is what feeds the webhook response, so if the Notion write fails, the caller will see a non-200. Treat this as the canonical "the meeting was processed" signal.

### Split Action Items (`splitOut`)

Fans the `action_items` array out to one item per action. Without this step, the Notion task node would only run once with the entire array, and you'd have to write n manually. SplitOut is the clean primitive for "n parallel writes from one upstream item".

### Notion Create Task (`Notion`)

Creates one row per action item in your tasks database, with `task` as the title and `owner`, `priority`, `due_date`, and `Source` (the meeting title) as additional properties. The Source property is what makes the tasks traceable back to their originating meeting later.

### Slack Meeting Recap (`Slack`)

Posts to the configured channel with a templated message: title, date, summary, then a bulleted list of `task -> owner` for every action item. The bot needs `chat:write` and must be invited to the channel.

The recap is a quality-control surface: skim Slack, see if the extraction looks right, edit Notion if it doesn't.

### Webhook Response (`respondToWebhook`)

Returns `{ received: true, tasks_created: N }` to the original caller. `N` reads from `$('Parse Meeting Data').item.json.action_items?.length`, so a parse failure that left `action_items` empty still returns a clean `tasks_created: 0` instead of erroring.

## Design decisions worth calling out

### Why a Code node parses the LLM output instead of a structured-output node

The chainLlm + structured-output combo in n8n is more strict and less forgiving. A small tag drift from the model breaks it. The Code node here is intentionally lenient: strip fences, try to parse, fall back on parse failure. For a workflow that runs on user-typed transcripts, lenient is the right default.

### Why three parallel branches off Parse Meeting Data

Notion summary, Notion tasks, and Slack recap are independent operations. There's no point serializing them. Running in parallel keeps the total latency to "longest of the three" instead of "sum of all three", and a failure on one branch doesn't take down the others.

### Why the webhook response only depends on the summary branch

If Notion accepted the meeting page, the workflow has done its primary job: there's now a record. The Slack post and per-task Notion writes are supportive, not the canonical output. Wiring the response to the meeting-page branch means a caller's success/failure signal aligns with "did the meeting get logged".

### Why both `Claude Extraction Model` and `Extract Actions and Decisions` exist as separate nodes

The chat model node holds credentials and model config. The chain node holds the prompt and orchestrates the call. They communicate via the `ai_languageModel` typed connection. Keeping them separate is the n8n convention and makes provider swaps a one-node change.

### Why `Source` is stored on every task

Six weeks from now, when someone asks "where did this task come from?", the Source field is the breadcrumb. Without it, action items become free-floating and untraceable.

## Performance notes

| Step | Latency expectation |
|---|---|
| Webhook receipt | <100 ms |
| Extract Meeting Fields | <50 ms |
| Claude extraction (Sonnet 4.6, 2k token cap) | 3 to 8 seconds for typical 30-min transcripts |
| Parse Meeting Data | <50 ms |
| Notion write (per row) | 300 to 800 ms |
| Slack post | 200 to 500 ms |
| Total wall-clock | 5 to 12 seconds for a meeting with 5 action items |

If your transcripts are very long (60+ min raw audio), Claude latency dominates. Consider either summarizing first with a smaller model or chunking the transcript.

## Observability

- **n8n Executions panel** is the primary debugging surface. Filter by the workflow name to see every webhook call with green/red on each node.
- The **sticky note inside the workflow** carries the live README. Edit it as you customize so future-you knows what's wired where.
- Consider a final `Log to Sheet` step that records `meeting_title`, `tasks_created`, and `success` per run. Useful for "did the recap go out?" questions without trawling executions.

## See also

- [SECURITY.md](SECURITY.md), threat model, layered defenses, what to lock down before production
- [Catalog architecture principles](https://github.com/MinaSaad1/n8n-ai-agents/blob/main/docs/architecture-principles.md), patterns shared across every template in the collection

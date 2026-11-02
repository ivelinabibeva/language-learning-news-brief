---
name: language-news-brief
description: Set up a daily news brief for learning a foreign language: news summaries at a chosen CEFR level with vocabulary, grammar notes, a read-aloud web page and a running vocabulary log.
---

# Language News Brief

Builds a personal daily "morning brief" for someone learning or relearning a language. Every morning a scheduled task finds fresh news from one country, rewrites it at the learner's level in the target language, adds vocabulary and one grammar point per story, publishes it to a colour-coded web page with read-aloud buttons, and appends the new words to a running vocabulary log so nothing repeats.

Use this when someone wants a daily language-learning brief, a news digest in a foreign language, or asks to reuse the "Morgenbrief" setup for their own language.

## Step 1: Gather the spec (ask once, in one AskUserQuestion round where possible)

Ask only for what is missing. Sensible defaults are in brackets.
- Target language and country of the news (e.g. German / Germany). Required.
- Level: one CEFR level for all summaries [B2]. Explain briefly that the goal is a level the learner could also SPEAK, not just read; a level that is too high makes the brief less useful.
- Mix of stories [3 hard-news + 2 pop-culture/entertainment].
- Topics to include [politics, economy, social affairs, sustainability, science, technology, show business] and exclude [crime, sports].
- Preferred sources (reputable national outlets for hard news; boulevard/tabloid outlets for pop culture). Suggest well-known outlets for that country and note that paywalled sites are harder to use.
- Summary length [100-150 words], terms per story [2-3], grammar points per story [1].
- Language of the explanations [the target language; offer the learner's own language as an alternative for lower levels].
- Delivery time and time zone. Required for scheduling.

## Step 2: Do a test run in the conversation first

Before building anything, search today's news and write one or two stories in chat following the spec. Ask whether the level, length and tone feel right. Adjust until the learner is happy. This is the most important step: the level usually needs one correction.

Writing rules for the summaries (put these into the scheduled prompt too):
- Write the way an educated person would explain the news to a friend, not in newspaper style: mostly short sentences, active verbs, no heavy nominal style, no rare literary words.
- Vocabulary should be words the learner could realistically use themselves when talking about the topic.
- Grammar points: pick a construction that actually appears in the summary, quote it, explain it in 2-4 simple sentences, give one extra everyday example. Prefer structures useful for speaking at the chosen level, and rotate them.
- Write numbers and abbreviations so a text-to-speech voice reads them naturally.
- Never copy sentences from sources; never invent facts. Read the actual articles.

## Step 3: Create the vocabulary log (Docs artifact)

Create a Docs artifact named "Vocabulary - <brief name>" (localised title is fine) with two tables:
- Terms: Date | Term | Explanation | Topic
- Grammar: Date | Structure | Example | Level
Seed it with the terms and grammar from the test run. Each scheduled run reads it first (to avoid repeats and rotate topics) and appends new rows at the end.

## Step 4: Build the brief page (hand-built HTML artifact)

Load the artifact-design skill first. Build one self-contained HTML page whose content comes from a published data file `brief.json` (fetched with a relative URL plus a cache-busting query), so the daily run only replaces the data file and never touches the HTML.

Page requirements:
- Header with the brief name, the date label, the level, and a small colour legend for the three section types.
- Hard-news stories grouped first, pop-culture stories second, each with its topic label and headline.
- Each story has three framed sections, each with a thin 1px border, a light tinted background and its own colour: News summary (blue), Vocabulary (green), Grammar (purple). All colours as theme tokens with proper dark-mode values.
- Section labels in the target language (e.g. Nachricht / Wortschatz / Grammatik).
- Read-aloud: under each summary a Play/Stop button using the browser's speechSynthesis. Split the text into sentences and queue them (avoids cut-offs on long texts) and highlight the sentence being spoken. Pick the best voice for the target language code (prefer names containing natural/neural/premium/enhanced, then known good voices), always set utterance.lang. Add a "slow" toggle (rate about 0.75 vs 0.95) remembered in localStorage (wrapped in try/catch). Add a small speaker button next to each vocabulary term that reads just that term. If speechSynthesis is missing, show a short note instead of the button. Cancel speech on pagehide.
- Grammar section: name + level, the quoted sentence as a blockquote, the explanation, and the example.
- Source link at the bottom of each news section.
- A designed empty state and a load-error state.
- Readable serif for the summaries, clean sans for UI; works at phone width.

brief.json shape:
```json
{
  "date": "YYYY-MM-DD",
  "date_label": "<weekday, date in the target language>",
  "stories": [
    {
      "kind": "news" | "promi",
      "topic": "...",
      "title": "...",
      "summary": "plain text, one paragraph",
      "terms": [{"term": "...", "explanation": "..."}],
      "grammar": {"name": "...", "level": "...", "quote": "...", "explanation": "...", "example": "..."},
      "source": {"name": "...", "url": "https://..."}
    }
  ]
}
```
Publish the page with the test-run stories as its first brief.json (an optional `note` field can mark them as samples).

## Step 5: Schedule the daily run

Create a scheduled task at the requested local time (use CRON_TZ with the learner's time zone, and move a full-hour time a few minutes earlier). Turn on push notifications. The prompt must be fully standalone (each run starts fresh) and contain, in order:
1. Who the learner is, the level, and the writing rules from Step 2.
2. Read the vocabulary log doc (by URL) using the Docs tools; note covered terms, grammar from the last 14 days, and recent topics.
3. Find news from the last ~24 hours per the spec (sources, topics, exclusions, topic rotation). Read the real articles.
4. Write the stories.
5. Publish: Artifact action "read" on the page URL to get the local HTML file; write brief.json exactly in the shape above (no note field); validate it with python3 -m json.tool; publish with url = page URL, file_path = that HTML file, files = {"brief.json": path}; no icon. On a version conflict, retry with the version handed back.
6. Notify with SendUserMessage: a short line with the page link and the headlines. If publishing failed, send the full brief in Markdown instead.
7. Read the log fresh; if rows for today already exist, skip; otherwise append today's terms and grammar rows at the end of each table without changing existing rows.

Then offer a manual trial run (fire the scheduled task once) so the learner can check the page end to end.

## Step 6: Hand-off

Tell the learner in a few lines: when the brief arrives, where the page and the vocabulary log are, that the voice depends on their device (natural/enhanced voices sound much better), and that level, length, topics or grammar focus can be changed any time by asking, which updates the scheduled task's prompt.

## Changing it later

- Level or style change: update the scheduled task's prompt (keep everything else) and show one rewritten story as a sample.
- Visual change: edit the page HTML and republish to the same URL; the data contract (brief.json) must stay compatible.
- Never delete and recreate the scheduled task just to change its prompt.

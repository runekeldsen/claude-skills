---
name: podcast-creator
description: Research a topic and produce a polished 10-minute podcast episode — including web research, a full solo-host script, and audio generation via n8n (OpenAI TTS → Google Drive → Telegram). Use this skill whenever the user says things like "make a podcast about X", "create a podcast episode on X", "podcast on X", "record an episode about X", "generate a podcast", or "I want a podcast". Trigger even for vague requests like "let's do a podcast" or "can you make an episode about this?".
---

# Podcast Creator

Produces a research-backed 10-minute podcast episode. You research the topic, write a compelling solo-host script (~1,500 words), then hand it off to the n8n audio pipeline which generates MP3 audio and saves it to Google Drive, with a Telegram notification when done.

---

## Step 1: Research the topic

Run 4–6 web searches to gather current, accurate, interesting material. Look for:
- Core explanation: what it is and why it matters
- Recent developments (last 6–12 months)
- Surprising angles, counterintuitive facts, or human stories
- Quotes, numbers, or examples that make it feel real

Synthesize across sources — don't summarize each one separately. You're building a narrative, not a literature review.

---

## Step 2: Write the script

Write a ~1,500-word solo-host monologue (reading pace ~150 words/min = 10 minutes).

### Structure

**Intro (~150 words)**
Open with a hook: a surprising fact, a vivid scene, or a question the listener didn't know they had. Then briefly frame what the episode is about and why now is the right time for it.

**Segment 1 — The Foundations (~350 words)**
What is this thing? Where did it come from? Give the listener just enough context to follow the rest.

**Segment 2 — The Interesting Part (~350 words)**
This is the meat. Recent developments, the controversy, the surprising data, the thing that changes how you think about it.

**Segment 3 — Real-World Impact (~350 words)**
How does this affect people, companies, society, or the future? Make it tangible.

**Outro (~150 words)**
Land on one clear takeaway. Leave the listener with a thought-provoking question or image. Sign off warmly.

### Script style

Write for the ear, not the eye:
- Conversational sentences, no bullet points (it's audio)
- Short paragraphs — 2 to 3 sentences each
- Transition phrases to carry momentum: *"Now here's where it gets interesting..."*, *"But wait — here's the twist."*, *"Let's zoom out for a second."*
- Write numbers as words: "three billion" not "3B", "forty percent" not "40%"
- Add `[PAUSE]` on its own line where the listener needs a breath between big ideas (use sparingly — 3 to 5 per episode)

Do NOT include stage directions, sound cues, or "HOST:" speaker labels. The script is pure spoken prose.

### Episode title

Pick a short, punchy title (5–10 words) that a podcast app subscriber would click on.

---

## Step 3: Send to the audio pipeline

After the script is complete, save it to a temp file and POST it to n8n:

```bash
python3 - <<'PYEOF'
import json, subprocess

topic = "TOPIC_HERE"
title = "TITLE_HERE"

with open('/tmp/podcast_script.txt', 'r') as f:
    script = f.read()

payload = {"topic": topic, "title": title, "script": script}

with open('/tmp/podcast_payload.json', 'w') as f:
    json.dump(payload, f)
PYEOF

curl -s -X POST https://n8n.keldsen.org/webhook/podcast-creator \
  -H "Content-Type: application/json" \
  -d @/tmp/podcast_payload.json
```

In practice, use the Bash tool with two separate calls:
1. Write the script and create the JSON payload file using Python (inline script with actual topic/title/script values substituted in)
2. Run the curl command

The n8n workflow handles: splitting the script into TTS-sized chunks → OpenAI TTS (voice: alloy) → combine audio → upload to Google Drive "Podcasts" folder → Telegram notification.

---

## Step 4: Tell the user

After the curl returns 200, report:
- Episode title
- Word count and estimated runtime (word_count ÷ 150 = minutes)
- That audio generation is running and they'll get a Telegram notification when the MP3 is ready in Google Drive

---

## What to do if something fails

- **Curl returns non-200**: show the response body and stop — don't retry automatically
- **Research finds very little**: broaden the search (try related terms) and be transparent that coverage is thin
- **Script goes over 2,000 words**: trim the longest segment; don't pad to make it exactly 1,500 if the content doesn't support it

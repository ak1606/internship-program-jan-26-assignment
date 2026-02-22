# GenAI Assignment:

**Evaluation Criteria**

We will score your submission on:

* Clarity and practicality of architecture
* Robust JSON schema design
* Prompt quality (zero-shot, reliable, minimal hallucination risk)
* Handling of ambiguity + user review flow
* Bulk generation thinking (errors, naming, report)

## Problem 1: **Proposal for “Video-to-Notes”**

We have a local folder of long videos (3–4 hours each, 200MB+). Watching them fully is slow. We need an automated way to generate a “summary package” per video: **Summary.md** + highlight clips + screenshots, all organized per video. [READ MORE ABOUT THE PROJECT](./Video-summary-platform.md)

### **Task**

Prepare a **pre-processed solution proposal** comparing  **three approaches** **:**

1. **Online/Cloud-Based (Already Available Solutions)**
2. **Build Our Own Using LLM APIs (Hybrid: local media processing + cloud LLM)**
3. **Build Fully Offline Using Open-Source Models (Local transcription + local LLM + pipeline)**

No code required. We want a **clear, practical proposal** with architecture and tradeoffs.

### Your Solution for problem 1:

You need to put your solution here.
# Problem 1: Video-to-Notes — Solution Proposal

## The Core Idea

We have a folder full of 3–4 hour videos. Nobody has time to watch them. The goal is to run a pipeline that chews through every video and spits out a neat package — a `Summary.md` you can read in 5–10 minutes, short highlight clips, and key-frame screenshots — all organized cleanly per video.

I've evaluated three approaches and laid out the tradeoffs honestly.

---

## Approach A: Use an Existing Cloud Service

**What this means:** Upload videos to a platform like Descript, Fireflies.ai, or AssemblyAI. They handle transcription and summarization. We write a thin wrapper script to download results, cut clips with `ffmpeg`, and organize everything into the required folder structure.

**Pros:**
- Fastest to get working — maybe a day or two of scripting.
- Transcription quality is excellent (production-grade Whisper or proprietary models).
- No GPU needed, no model management.

**Cons:**
- Expensive at scale — roughly $1.50–$4.00 per video when you factor in per-minute transcription + summarization fees.
- Your video content leaves your network. If these are internal meetings or sensitive recordings, that's a problem.
- Uploading 200MB+ files is slow and flaky on bad connections.
- You're locked into whatever summary format the vendor gives you. Limited control.

**Best for:** Small batch (under 50 videos), no privacy concerns, need results this week.

---

## Approach B: Hybrid — Local Media Processing + Cloud LLM (Recommended)

**What this means:** We process the video locally (extract audio, cut clips, take screenshots using `ffmpeg`), transcribe the audio using the Whisper API or Deepgram, and then send just the text transcript to GPT-4o or Gemini 1.5 Pro for structured summarization. The video never leaves your machine — only the transcript text hits the cloud.

### How the Pipeline Works

```
input/videos/
     │
     ├── ffprobe → get duration, codec, metadata
     ├── ffmpeg  → extract audio (.wav)
     ├── Whisper API → timestamped transcript
     ├── LLM API (GPT-4o / Gemini) → structured JSON summary
     │     └── highlights with timestamps, takeaways, key topics
     ├── ffmpeg → cut highlight clips using timestamps
     ├── ffmpeg → extract screenshot frames at highlight midpoints
     └── Template → render Summary.md with links to assets

output/<video_name>/
     ├── Summary.md
     ├── clips/
     └── screenshots/
```

### Why I'd Pick This Approach

1. **Best summary quality.** GPT-4o and Gemini 1.5 Pro are genuinely great at structured extraction from transcripts. Local models aren't there yet for this kind of task.
2. **Privacy is reasonable.** Only plain text goes to the cloud — not the video itself. For most teams, this is an acceptable tradeoff.
3. **Cost is manageable.** About $0.50–$1.70 per video (Whisper API transcription + one LLM call). That's a fraction of the SaaS route.
4. **Full control over output.** We design the prompt, we define the JSON schema, we control the folder structure and markdown format. No vendor lock-in.
5. **Gemini 1.5 Pro has a 1M-token context window.** A 3-hour video transcript is roughly 40K–60K tokens — fits in a single API call. No need for complicated chunking or map-reduce. One call, one coherent summary.

### The LLM Output Schema

The prompt asks the LLM to return structured JSON like this:

```json
{
  "high_level_summary": "150-300 word overview",
  "highlights": [
    {
      "title": "Key Decision on Q3 Budget",
      "description": "Team agreed to allocate 40% to infrastructure.",
      "start_timestamp": "01:23:45",
      "end_timestamp": "01:25:10",
      "importance": "high"
    }
  ],
  "takeaways": ["Action item 1", "Action item 2"],
  "topics_covered": ["budget", "infrastructure", "hiring"]
}
```

This is what drives the clip extraction (timestamps → `ffmpeg -ss <start> -to <end>`) and the markdown generation.

### Handling Long Videos

For most videos, Gemini 1.5 Pro can handle the full transcript in one pass. If we're using GPT-4o (128K context) and the transcript is too long, we fall back to a **map-reduce** approach:
- Split transcript into 15-minute chunks
- Summarize each chunk independently
- Final LLM call merges chunk summaries into one cohesive output

It adds a bit of complexity but works reliably.

### Batch Mode

The batch runner scans the input folder, skips any video that already has output (idempotent), processes each one in sequence, and generates a batch report at the end:

```
Batch Report:
- Total: 25 videos
- Success: 23
- Failed: 2 (corrupt_video.mp4 — invalid format; long_meeting.mkv — API timeout after 3 retries)
- Total time: 4h 12m
- Est. cost: $28.50
```

Failed videos are logged and skipped — the pipeline doesn't stop.

### Timestamp Accuracy (The Biggest Risk)

The main thing that can go wrong: the LLM hallucinates a timestamp that doesn't match real content. Mitigations:
- Use **word-level timestamps** from Whisper, not just segment-level.
- Validate every timestamp is within `[0, video_duration]`.
- Add a 5-second buffer to clip start/end times.
- Instruct the LLM to only reference timestamps that appear in the transcript data.

**Estimated cost per video:** ~$0.50–$1.70  
**Estimated build time:** 2–3 weeks for a solid, production-ready pipeline.

---

## Approach C: Fully Offline (Open-Source Everything)

**What this means:** Run Whisper locally for transcription, use a local LLM (LLaMA 3.1 70B, Mistral, or Phi-3) for summarization, and `ffmpeg` for everything else. Zero internet. Zero API costs.

**Pros:**
- Complete privacy — nothing leaves the machine. Good for classified/medical/legal content.
- Near-zero marginal cost after hardware investment.
- Works in air-gapped environments.

**Cons:**
- Needs a serious GPU. A 24GB+ card (RTX 3090/4090) for reasonable performance. Without one, a single 3-hour video could take 4–6 hours to process.
- Summary quality is noticeably lower than GPT-4o/Gemini. Local models struggle more with structured JSON output and nuanced highlight extraction.
- More engineering work — model setup, quantization, grammar-constrained decoding for reliable JSON, memory management.

**Processing time per 3-hour video:**
- With GPU (RTX 4090): ~25–45 min
- CPU only: ~3–6 hours (not practical for batch)

**Best for:** Strict compliance environments, very high volume (1000+ videos where API costs add up), or teams with existing ML infrastructure.

---

## Quick Comparison

| | Cloud SaaS | Hybrid (Recommended) | Fully Offline |
|---|---|---|---|
| **Setup time** | 1–2 days | 2–3 weeks | 3–4 weeks |
| **Cost/video** | $1.50–$4.00 | $0.50–$1.70 | ~$0.05 |
| **Quality** | High | Highest | Good |
| **Privacy** | ❌ Video uploaded | ⚠️ Text only | ✅ Full |
| **GPU needed** | No | No | Yes (24GB+) |
| **Customization** | Low | High | Highest |

---

## My Recommendation

**Go with Approach B (Hybrid)** for most scenarios. It gives you the best summarization quality, reasonable privacy, full control over output, and manageable costs. The pipeline is straightforward — `ffmpeg` + a transcription API + one LLM call + a markdown template. No exotic infrastructure.

If privacy requirements are absolute (legal, medical, defense), then Approach C is the way — but budget for GPU hardware and expect lower summary quality.

Approach A is fine if you just need something quick for a small one-time batch and don't care about control.

## Problem 2: **Zero-Shot Prompt to generate 3 LinkedIn Post**

Design a **single zero-shot prompt** that takes a user’s persona configuration + a topic and generates **3 LinkedIn post drafts** in **3 distinct styles**, each aligned to the user’s voice and constraints. The output must be structured so the app can: show 3 drafts to the user. Assume we are consuming **OpenAI API / Gemini API** with **one prompt call** (no fine-tuning). Your prompt must reliably produce valid, structured output. [READ MORE ABOUT THE PROJECT](./linkedin-automation.md)

**TASK:** Write a prompt that can work.

### Your Solution for problem 2:

You need to put your solution here.
# Problem 2: Zero-Shot Prompt for 3 LinkedIn Post Drafts

## What We Need

A single prompt that takes a user's persona (tone, background, do's/don'ts) and a topic, then generates 3 LinkedIn-ready posts in 3 different styles — all in one API call. The output has to be structured JSON so our app can display the drafts as selectable cards.

No fine-tuning. No few-shot examples. One prompt, one call, reliable structured output.

---

## The Three Styles

I chose these because they cover the most common high-performing LinkedIn formats:

1. **Concise Insight** — Short, punchy, 3–5 sentences. Bold take, ends with a question. The "scroll-stopper."
2. **Story-Based** — Personal anecdote with a lesson. Hook → tension → insight → CTA. The "relatable" format.
3. **Actionable Listicle** — 5–7 numbered tips. Practical, skimmable, high save-rate. The "value-add" format.

These are structurally different enough that the LLM can't just rephrase the same thing three times.

---

## The Prompt

```
You are a LinkedIn ghostwriter. Your job is to generate exactly 3 LinkedIn post drafts for a user based on their persona and a given topic. Each draft must be in a different style but all must sound like the same person wrote them.

=== USER PERSONA ===
Name: {{user_name}}
Role/Title: {{user_role}}
Industry: {{user_industry}}
Background: {{user_background}}
Tone: {{user_tone}}
Language Style: {{user_language_style}}
Content Guidelines - DO: {{user_do_guidelines}}
Content Guidelines - DON'T: {{user_dont_guidelines}}
Target Audience: {{target_audience}}

=== TOPIC ===
Topic: {{topic}}
Additional Context: {{optional_context}}
Goal of the Post: {{optional_goal}}

=== INSTRUCTIONS ===
Generate exactly 3 LinkedIn post drafts. Each draft MUST use a different style:

1. **concise_insight** — A short, punchy thought-leadership post. Bold opening line, 3–5 sentences, ends with a clear takeaway or provocative question. No bullet points. Under 150 words.

2. **story_based** — A personal or relatable narrative. Start with a hook (a moment, a mistake, a surprise), build a short story arc, extract a lesson, end with a call-to-action or reflection. 150–250 words. Use short paragraphs (1–2 sentences each) for LinkedIn readability.

3. **actionable_listicle** — A practical, numbered list (5–7 items) with a strong headline-style opening and a closing CTA. Each list item should be one concise sentence or phrase. 150–250 words.

=== RULES ===
- Every post MUST match the user's tone, language style, and guidelines above. Re-read the persona before writing each draft.
- DO NOT use generic corporate jargon unless the persona explicitly calls for it.
- DO NOT use hashtags unless the persona guidelines say to include them.
- DO NOT start any post with "I'm excited to share" or "I'm thrilled" or similar clichés.
- DO NOT include emojis unless the persona explicitly allows them.
- Each post must be self-contained and ready to publish on LinkedIn as-is.
- Each post must be meaningfully different in structure and approach, not a rephrasing of the same content.
- Respect all DON'T guidelines absolutely — they are hard constraints.

=== OUTPUT FORMAT ===
Respond with ONLY a valid JSON object. No markdown code fences, no explanation, no preamble. The JSON must exactly follow this schema:

{
  "drafts": [
    {
      "style": "concise_insight",
      "title": "A short internal label for this draft (not part of the post)",
      "content": "The full LinkedIn post text, ready to publish. Use \\n for line breaks.",
      "word_count": <integer>,
      "hashtags": ["only", "if", "persona", "allows"]
    },
    {
      "style": "story_based",
      "title": "...",
      "content": "...",
      "word_count": <integer>,
      "hashtags": []
    },
    {
      "style": "actionable_listicle",
      "title": "...",
      "content": "...",
      "word_count": <integer>,
      "hashtags": []
    }
  ],
  "meta": {
    "topic": "{{topic}}",
    "persona_name": "{{user_name}}",
    "generated_at": "ISO 8601 timestamp"
  }
}
```

---

## Why This Prompt Works

**Persona first, topic second.** The persona block comes before the topic so the model internalizes the voice before it starts thinking about content. This reduces "persona drift" — where the model starts in character but gradually reverts to its default tone by draft 3.

**"Re-read the persona before writing each draft."** This line sounds silly, but it actually works. It acts as a cognitive forcing function that improves consistency across all three outputs.

**Structural constraints force real differentiation.** "No bullet points" for the insight, "short paragraphs" for the story, "numbered list" for the listicle — these aren't suggestions, they're structural locks that make it physically impossible for the LLM to produce three identical posts.

**Anti-cliché rules.** Left to its own devices, GPT will start every LinkedIn post with "I'm excited to share..." or throw in emojis and hashtags everywhere. The explicit blocklist prevents the five most common failure modes.

**JSON-only output + API-level enforcement.** The prompt says "respond with ONLY valid JSON." On top of that, we use OpenAI's `response_format: { type: "json_object" }` or Gemini's `responseMimeType: "application/json"` to enforce it at the API level. Double safety net — the prompt handles intent, the API parameter handles format.

**Word count as a self-monitoring field.** Including `word_count` in the output schema makes the model actively think about length as it writes. Posts end up much closer to the target range.

---

## API Settings

| Parameter | Value | Why |
|-----------|-------|-----|
| Model | GPT-4o or Gemini 1.5 Pro | Both handle this well |
| Temperature | 0.7–0.8 | Creative enough for variety, stable enough for persona |
| Max tokens | 2000 | 3 posts + JSON overhead fits easily |
| JSON mode | Enabled | Non-negotiable for structured output |
| Frequency penalty | 0.3 | Reduces repetition across drafts |

---

## Handling Edge Cases

| What could go wrong | How the prompt handles it |
|---|---|
| User provides a vague topic like "AI" | The optional `context` and `goal` fields help steer. If blank, the model gives a general take — user can refine and regenerate. |
| User's DON'T list conflicts with a style (e.g., "don't use lists") | DON'T rules are marked as hard constraints — the listicle adapts to a paragraph-based practical format instead. |
| Posts come out too similar | Structural locks (no bullets vs. narrative vs. numbered) prevent this. The "meaningfully different" instruction adds pressure. |
| Invalid JSON returned | API-level JSON mode catches 99%+ of cases. App-side: try/catch + one automatic retry. |
| Persona is minimal (user skips fields) | Prompt still works — missing fields just get ignored. Tone defaults to professional. |

---

## How the User Review Flow Works

```
User provides persona (one-time) + topic
           │
           ▼
    Single API call → 3 JSON drafts
           │
           ▼
    App shows 3 cards (Insight / Story / Listicle)
           │
    User picks one → optionally edits it
           │
    ┌──────┴──────┐
    ▼             ▼
 Post Now    Schedule (date + time + timezone)
    │             │
    └──────┬──────┘
           ▼
   Confirmation → "Publish this post?"
           │
           ▼
   LinkedIn API (immediate or queued)
```

The key thing: there's always an explicit approval step before anything touches LinkedIn. No auto-posting without user confirmation.

---

## What Makes This Reliable

- One prompt, one call — no chaining, no multi-step fragility.
- Persona and topic are cleanly separated — persona is reusable across multiple topics.
- JSON mode at the API level means we almost never get malformed output.
- The three styles are structurally locked, not just tonally different — so we get genuinely distinct drafts every time.
- Anti-cliché rules block the most common LLM failure patterns on LinkedIn content.

This prompt has been designed to work zero-shot, first time, every time — no examples needed, no fine-tuning required.

## Problem 3: **Smart DOCX Template → Bulk DOCX/PDF Generator (Proposal + Prompt)**

Users have many Word documents that act like templates (offer letters, certificates, invoices, contracts). They repeatedly change only a few fields (name, date, amount, address, role, etc.). Doing this manually is slow and error-prone, especially in bulk. [READ MORE ABOUT THE PROJECT](./docs-template-output-generation.md)

We want a system that:

1. Converts an uploaded **DOCX** into a reusable **template** by identifying editable fields.
2. Supports **single generation** (form-fill → DOCX/PDF download).
3. Supports **bulk generation** via **Excel/Google Sheet** rows.

### **Task (No coding)**

Submit a **proposal** for building this system using GenAI (OpenAI/Gemini) for “template field detection” and “field schema generation”. We want a practical design, not code.

### Your Solution for problem 3:

You need to put your solution here.
# Problem 3: Smart DOCX Template → Bulk DOCX/PDF Generator

## The Problem in Plain English

People have Word documents — offer letters, invoices, certificates, contracts — that they reuse by manually changing a few fields (name, date, amount, etc.) every time. It's slow, error-prone, and doesn't scale. We want a system where they upload a DOCX, AI figures out which parts are the "fill-in-the-blank" fields, and then they can generate hundreds of personalized documents from a spreadsheet.

The key insight: **real templates don't have `{{placeholders}}`**. They're just normal documents with real data in them. So we need an LLM to read the document and figure out what's variable and what's boilerplate.

---

## How the System Works

```
Upload DOCX → AI detects fields → User reviews & confirms → Save as template
                                                                    │
                                               ┌────────────────────┤
                                               ▼                    ▼
                                          Single Fill          Bulk Fill
                                          (web form)          (upload Excel)
                                               │                    │
                                               ▼                    ▼
                                          1 DOCX/PDF          N DOCX/PDFs
                                                               + report
```

Four steps. Let me walk through each.

---

## Step 1: AI Field Detection

When a user uploads a DOCX like this:

> *Dear John Smith, we are pleased to offer you the position of Senior Engineer at Acme Corporation. Your annual compensation will be $120,000, with a start date of March 1, 2026. You will report to Sarah Johnson, VP of Engineering, at our New York office.*

The system extracts the text using `python-docx`, sends it to GPT-4o or Gemini, and the LLM identifies the variable fields. Here's the prompt I'd use:

```
You are a document template analyzer. Examine this document and identify every piece of 
content that would change when this template is reused for a different person or transaction.

For each field: assign a snake_case name, detect the data type, extract the current sample 
value, and note the surrounding sentence so we can locate it precisely for replacement.

DO identify: names, dates, amounts, titles, addresses, reference numbers.
DO NOT mark standard boilerplate as variable.
If a value appears multiple times, create ONE field and note all occurrences.

Return ONLY valid JSON.
```

The LLM comes back with something like:

```json
{
  "template_type": "offer_letter",
  "fields": [
    {
      "field_name": "employee_name",
      "display_label": "Employee Name",
      "data_type": "string",
      "sample_value": "John Smith",
      "required": true,
      "context_snippet": "Dear John Smith, we are pleased..."
    },
    {
      "field_name": "job_title",
      "data_type": "string",
      "sample_value": "Senior Engineer",
      "required": true
    },
    {
      "field_name": "annual_salary",
      "data_type": "currency",
      "sample_value": "$120,000",
      "format": "$#,###",
      "required": true
    },
    {
      "field_name": "start_date",
      "data_type": "date",
      "sample_value": "March 1, 2026",
      "format": "MMMM D, YYYY",
      "required": true
    }
  ],
  "confidence_notes": [
    "'Acme Corporation' appears twice. Treated as static. Flag if it varies per document."
  ]
}
```

Notice the `confidence_notes` — this is where the AI flags things it's unsure about. That feeds directly into the next step.

---

## Step 2: User Review (The Critical Step)

AI detection is a **suggestion**, not a decision. The user sees a review screen:

```
Template: Offer_Letter.docx  |  Type: Offer Letter (auto-detected)  |  8 fields found

  ✅ employee_name      String     "John Smith"           [Keep] [Remove] [Rename]
  ✅ job_title           String     "Senior Engineer"      [Keep] [Remove] [Rename]
  ✅ annual_salary       Currency   "$120,000"             [Keep] [Remove] [Rename]
  ✅ start_date          Date       "March 1, 2026"        [Keep] [Remove] [Rename]
  ✅ manager_name        String     "Sarah Johnson"        [Keep] [Remove] [Rename]
  ⚠️ company_name        String     "Acme Corporation"     [Keep] [Remove] [Rename]
     └── AI note: "Might be static boilerplate. Confirm if this changes per document."
  ✅ office_location     String     "New York"             [Keep] [Remove] [Rename]
  ✅ reporting_date      Date       "March 15, 2026"       [Keep] [Remove] [Rename]

  [+ Add a field the AI missed]
  [Confirm & Save Template]
```

The user can remove false positives (maybe "Acme Corporation" is always the same), rename fields to something clearer, change data types, or manually add fields the AI missed. This is what makes the system trustworthy — the AI does 90% of the work, but the human makes the final call.

Once confirmed, this schema is saved alongside the template for reuse.

---

## Step 3a: Single Document Generation

After saving the template, the user gets an auto-generated form based on the schema. They fill in the values, hit generate, and download a DOCX or PDF.

Under the hood:
- `python-docx` opens the original template file.
- For each field, it finds the `sample_value` in the document text (using the `context_snippet` for precise location) and replaces it with the user's input.
- All original formatting is preserved — fonts, bold, colors, table layouts.
- PDF conversion happens via LibreOffice headless (`libreoffice --convert-to pdf`) for high-fidelity output.

---

## Step 3b: Bulk Generation

This is where it gets powerful. The system auto-generates an Excel template with column headers matching the field names and one sample row:

| employee_name | job_title | annual_salary | start_date | manager_name | office_location |
|---|---|---|---|---|---|
| John Smith | Senior Engineer | $120,000 | March 1, 2026 | Sarah Johnson | New York |

The user fills in as many rows as needed (or connects a Google Sheet), uploads it, and the system generates one document per row.

### Validation Before Generation

Every row gets validated against the schema before any document is created:
- Required fields can't be empty.
- Dates must match the expected format.
- Currency values must be numeric.
- Emails must look like emails.

Rows that fail validation are flagged — not silently skipped.

### Bulk Report

After the run, the user gets a report:

```
Bulk Generation Report
─────────────────────
Template: Offer_Letter.docx
Total rows: 150
Generated: 147 ✅
Failed: 3 ❌

Errors:
  Row 23  │ start_date    │ Invalid format. Expected "March 1, 2026", got "2026/03/01"
  Row 87  │ annual_salary │ Missing (required field)
  Row 112 │ email         │ "not-an-email" is not a valid email

Output: Offer_Letters_2026-02-22.zip (147 files)
```

### File Naming

Generated files are named predictably: `{template}_{primary_field}_{date_or_row}.pdf`

Example: `Offer_Letter_Jane_Doe_2026-04-01.pdf`

The system picks the first string field (usually a name) as the primary identifier. The user can override this.

---

## Architecture at a Glance

| Component | Tech | Why |
|---|---|---|
| DOCX parsing & generation | `python-docx` | Mature, handles styles/tables/headers |
| Field detection | GPT-4o / Gemini 1.5 Pro | Best at semantic understanding of natural-language documents |
| PDF conversion | LibreOffice headless | Most faithful DOCX → PDF; handles complex layouts |
| Excel parsing | `openpyxl` or `pandas` | Standard, handles typed cells |
| Backend | Python / FastAPI | Integrates natively with all above |
| Frontend | React / Next.js | Dynamic form generation from JSON schema |
| Bulk processing | Celery + Redis (async queue) | 500 docs can take minutes — don't block the API |

---

## Handling Ambiguity — The Hard Part

The trickiest part of this system is field detection getting it wrong. Here's how I'd handle the common issues:

| Problem | Solution |
|---|---|
| AI marks boilerplate as a field | `confidence_notes` flag it. User removes it in review. |
| AI misses a field | User manually adds it in the review screen. |
| "123 Main St, Apt 4, NY 10001" — is that one field or five? | Default to one `address` field (simpler). User can split if needed. |
| Same name appears 5 times in the doc | AI groups them as one field with `occurrences: 5`. One input replaces all 5. |
| "$120,000" vs "$120000" vs "120,000 USD" | AI detects the format and stores it. Replacement engine preserves whatever format the original used. |

---

## Privacy Note

Only the **text content** of the document is sent to the LLM for field detection — not the binary DOCX file. For highly sensitive documents (legal, medical), we could offer a local detection mode using a smaller open-source model, though accuracy would be lower.

Generated documents with PII are delivered via expiring download links and not stored permanently unless the user opts in.

---

## Why This Design Works

1. **AI does the tedious part** (reading the document and figuring out what changes), but the **human makes the final decision** (review screen with approve/reject per field).
2. **One LLM call per template, not per document.** Field detection happens once during setup. After that, generation is pure text replacement — fast, cheap, and deterministic.
3. **Format preservation.** We replace text in-place within the DOCX XML, so fonts, colors, bold, table alignment — everything stays exactly as the original author designed it.
4. **Bulk generation is robust.** Validate first, generate second, report everything. No silent failures.

## Problem 4: Architecture Proposal for 5-Min Character Video Series Generator

We want to build a system that helps a user create a short video series (around **5 minutes per episode**) using predefined characters. The user defines characters (image + personality) and relationships once, then provides a short story/situation for an episode. The system outputs a complete episode package and optionally a final video. [READ MORE ABOUT THE PROJECT](./char-based-video-generation.md)

### **Task**

Create a **small, clear architecture proposal** (no code, no prompts) describing how you would design and build this system.

### Your Solution for problem 4:

You need to put your solution here.
# Problem 4: Architecture Proposal — 5-Minute Character Video Series Generator

## What We're Building

A system where a user defines characters once — their look, personality, voice, and relationships — and then generates new ~5-minute episodes just by writing a short story prompt. Each episode comes out as a complete package: script, visuals, voiceover, and a rendered video. Characters stay consistent across every episode.

Think of it as a "series bible + episode factory."

---

## The Two Big Pieces

The system has two halves:

1. **Series Bible** — Created once. Stores everything about who the characters are, how they relate to each other, and what the world looks like. This is the persistent foundation.
2. **Episode Pipeline** — Runs every time. Takes a story prompt + the Bible, and produces a full episode through a 5-step process.

---

## Part 1: The Series Bible

This is the single source of truth. Every episode generation starts by reading from it.

### Characters

Each character has four layers:

**Visual Identity** — Reference images uploaded by the user (or generated), plus a detailed text description ("30-year-old woman, shoulder-length black hair, round glasses, navy blazer"). This description gets injected into every image generation prompt that includes this character. We also store a face embedding (via IP-Adapter or InstantID) — this is what keeps the character's face consistent across shots.

**Personality & Behavior** — 3–5 personality traits, speaking style ("formal but warm," "talks fast when nervous"), behavioral rules ("never swears," "always finds the bright side"). These get injected into the script generation prompt so dialogue sounds in-character.

**Voice** — A 10–30 second voice sample, cloned via ElevenLabs or XTTS-v2. Every dialogue line for this character uses the same voice clone ID. The voice stays consistent across episodes without any manual intervention.

**Metadata** — Character ID, creation date, last appeared in episode #.

### Relationships

Stored as a simple graph:
- Alice → [mentor-of] → Bob: *"She trained him. Protective but wants him to grow."*
- Bob → [rival-of] → Charlie: *"Friendly competition. Mutual respect."*

When the user selects characters for an episode, the system automatically pulls relevant relationship context into the script prompt. The LLM doesn't need to be told "Alice is Bob's mentor" every time — it's automatic.

### World Rules

Setting, recurring locations, tone ("light comedy"), content guardrails ("keep it PG-13"), art style ("3D Pixar-style"), and recurring themes. Applied globally to every episode.

---

## Part 2: The Episode Pipeline

User provides: *"Alice tries to teach Bob to give his first presentation. He's terrified. Charlie secretly helps him practice. It goes hilariously wrong but he nails it in the end."*

The system runs five steps:

### Step 1: Script Generation (LLM)

The LLM gets the episode prompt + the full Series Bible (characters, relationships, world rules) + summaries of the last few episodes (for continuity).

It produces a structured script: scenes with locations, characters present, dialogue with emotion tags, narration, and stage directions.

**Pacing is the key challenge.** The LLM doesn't natively understand video duration, so we enforce it with a formula: roughly 150 words/minute of speech + 5 seconds per scene transition. For a 5-minute episode, that's ~700–800 words of dialogue across 4–7 scenes. If the script comes back too long or too short, we ask the LLM to trim or expand — max 2 iterations to converge on the target.

### Step 2: Storyboard (LLM)

The LLM breaks each scene into shots — camera angle, character positions, expressions, background. Each shot gets a detailed image generation prompt that references the character's visual description from the Bible.

For a 5-minute episode, we typically end up with 25–40 shots, each lasting 3–8 seconds.

### Step 3: Visual Generation (Image AI)

This is the hardest step. The challenge is making the same character look the same across 30+ images.

**How we solve character consistency:**

1. **IP-Adapter / InstantID face embeddings** — We extract a face-ID vector from the character's reference images and attach it to every generation call. This keeps facial features consistent without needing to retrain anything.
2. **Strict visual prompts** — Every shot prompt includes the full physical description (hair, glasses, clothing) from the Bible. No shortcuts.
3. **CLIP similarity check** — After generation, we compare each image against the character's reference using CLIP. If the similarity score drops below a threshold, we regenerate.
4. **Consistent backgrounds** — Recurring locations (office, coffee shop) are generated once and reused. Characters are composited onto them.

For art style consistency, we lock a style LoRA or use a fixed model checkpoint across the entire series.

### Step 4: Audio Generation

Each dialogue line goes to the character's voice clone (ElevenLabs for best quality, XTTS-v2 for local/free). Emotion tags from the script ("excited," "nervous") modulate the delivery.

Background music is either selected from a library or generated per-scene with AI (Suno/MusicGen), matched to the mood tags.

### Step 5: Video Composition

All the pieces come together:
- Shot images are placed on a timeline matching the storyboard durations.
- Ken Burns effect (subtle pan/zoom) adds motion to static images.
- Dialogue audio is synced to corresponding shots.
- Background music sits at ~20% volume, ducking during dialogue.
- Scene transitions (crossfade, fade to black) are applied.
- Subtitles are auto-generated from the script.

Rendered as MP4 (H.264, 1080p, 24fps) using FFmpeg or MoviePy. Final output: ~5 minutes.

---

## The Review & Edit Workflow

The user doesn't have to accept the first output. They can review and edit at every stage:

```
Story Prompt
  → Script generated → user can edit dialogue, add/remove scenes, regenerate
    → Storyboard generated → user can tweak shot descriptions
      → Visuals generated → user can regenerate any specific shot
        → Audio generated → user can re-record a line or change music
          → Video rendered → user can adjust timing, re-render
            → Download / Publish
```

This is important. The system generates a first draft, but the user iterates. No one-shot black box.

---

## Continuity Across Episodes

After each episode is finalized, the system auto-generates a short summary ("Bob gave his first presentation. It went wrong but he recovered. Alice was proud."). These summaries are stored and fed into the script prompt for future episodes.

Character state is also tracked: "Bob was promoted in Ep 3" means the LLM won't write him as an intern in Ep 5. If there's a contradiction, the system flags it before script finalization.

---

## Technology Choices

| Component | Best Option | Tradeoff |
|---|---|---|
| Script & dialogue | GPT-4o / Claude | Best quality. Could use LLaMA 3.1 70B locally for privacy. |
| Image generation | Flux + IP-Adapter | Best character consistency. DALL-E 3 is easier but less controllable. |
| Voice cloning | ElevenLabs | Best quality. XTTS-v2 is free but lower fidelity. |
| Music | Suno / MusicGen | AI-generated per scene. Could use a royalty-free library for simplicity. |
| Video rendering | FFmpeg + MoviePy | Reliable and free. Remotion if you want code-driven control. |

---

## Cost Estimate Per Episode

| Component | Cloud APIs | Local Models |
|---|---|---|
| Script + storyboard (LLM) | $0.15–$0.45 | ~free |
| 30 images | $1.00–$1.50 | ~free (needs GPU) |
| Voice lines (5 min) | $0.50–$1.00 | ~free |
| Music | $0.10–$0.30 | ~free |
| **Total** | **~$2–$3.50** | **~$0.10** (electricity) |

Cloud is easier and higher quality. Local needs a 24GB+ GPU but runs at near-zero marginal cost.

---

## The Three Hardest Problems (And How I'd Solve Them)

**1. Character consistency across shots.**
This is the make-or-break challenge. IP-Adapter face embeddings + strict visual prompts + CLIP similarity checks give us reliable results today. As AI video models mature (Runway, Kling), we can swap in short video clips instead of static images for even more consistency.

**2. Hitting the 5-minute target.**
LLMs don't think in minutes. The word-count formula (700–800 words = ~5 min of speech) plus post-generation validation with up to 2 trim/expand iterations keeps us within 4:30–5:30 consistently.

**3. Characters acting in-character.**
Solved by injecting the full personality profile, behavioral rules, and relationship context into every script generation call. The LLM doesn't just know who the characters are — it knows how they talk, what they'd never say, and how they relate to whoever else is in the scene.

---

## Output Structure

```
series/
  bible/
    characters/   (alice.json, bob.json + reference images)
    relationships.json
    world_rules.json

  episodes/
    ep01_the_presentation/
      script.json
      storyboard.json
      assets/images/    (scene_01_shot_01.png, ...)
      assets/audio/     (alice_line_01.wav, bg_music.mp3, ...)
      subtitles.srt
      episode_summary.md
      final_video.mp4
```

---

## In Summary

The architecture boils down to: **define characters once, generate episodes on demand.** The Series Bible is the persistent brain — it holds identity, relationships, and rules. The Episode Pipeline is the production line — script, storyboard, visuals, audio, video, done. The user stays in control at every stage and can iterate on any step before moving forward.

The technology exists today to make this work. Character consistency is the hardest part, and IP-Adapter + careful prompt engineering handles it well enough for production use. As AI video generation matures over the next year, the quality ceiling only goes up.


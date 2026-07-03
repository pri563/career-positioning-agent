# career-positioning-agent
AI-powered resume and career positioning agent — runs in Claude.ai
# Career Positioning Agent

An AI-powered resume and career positioning tool that runs entirely inside [Claude.ai](https://claude.ai). No installation. No account beyond Claude. No data leaves your browser except the API calls to Anthropic.

---

## What it does

You paste in your resume, a target job description, and some background about yourself. The agent analyzes the fit, identifies gaps, and produces tailored career documents — without inventing facts or inflating your experience.

**Six outputs, one tool:**

| Output | What it produces |
|---|---|
| **Tailored Resume** | A rewritten resume positioned for the specific role, with a reasoning summary of every tailoring decision made |
| **What to Change** | A section-by-section edit guide if you'd rather update your own resume than use the rewrite |
| **LinkedIn Updates** | Revised About and Experience sections for your LinkedIn profile |
| **Executive Bio** | A third-person bio in short and long form |
| **Pitch Deck** | A 5-slide career narrative for networking conversations and referrals |
| **Master Resume** | A broad, un-tailored version that captures everything — use this as your source document |

---

## Requirements

- A [Claude.ai](https://claude.ai) account (free tier works; paid tier runs outputs faster in parallel)
- A browser — no installs, no CLI, no API key setup

---

## How to use it

### Step 1 — Get the file

Download `CareerAgent_Generic.html` from this repo.

On the repo page: click the filename → click the **Download raw file** button (the down-arrow icon in the top-right of the file view).

### Step 2 — Open it in Claude

1. Go to [claude.ai](https://claude.ai) and start a new conversation
2. Click the **paperclip / attachment** icon in the chat input
3. Upload `CareerAgent_Generic.html`
4. Claude will render it as an interactive artifact — the agent UI appears in the chat

### Step 3 — Paste your materials

The agent has four input fields:

| Field | What to paste |
|---|---|
| **About Me** | A narrative description of your career — who you are, what you've built, what you're proud of, what you're looking for. Write it like you'd explain yourself to a trusted mentor. The more honest and specific, the better the output. |
| **Master Resume** | Your full current resume. Paste it as plain text. |
| **LinkedIn Profile** | Optional. Your current LinkedIn About and Experience sections. |
| **Target Role** | The full job description you're applying for. |

**Tips for better output:**
- The About Me is your most important input. It carries context the resume can't — reasons for moves, what you actually owned vs. coordinated, achievements you haven't put in writing yet.
- Paste the full JD, not a summary. The agent reads it for role intent, not just keywords.
- If you have a master pitch deck or career narrative document, there's a field for that too.

### Step 4 — Run the analysis

Click **Analyze**. The agent scores the fit across three lenses (ATS screening, recruiter, hiring manager) and produces a positioning strategy before generating any documents.

If your fit score is below 70, the agent will ask clarifying questions before proceeding. You can answer them, skip them, or proceed anyway.

### Step 5 — Generate outputs

Select the outputs you want and click **Generate**. Outputs appear in tabs. Each can be copied individually.

---

## After the first run

### Refining with corrections

Use the **Additional Context or Corrections** field under the outputs to fix anything the agent got wrong. Be specific:

> "The PMO was inherited, not built from scratch. Change every instance."

> "Remove Python from the tools section — it's not my skill."

> "The title should be Senior TPM, not Director."

The agent applies corrections as hard overrides on the next generation, not suggestions.

### Saving your profile

Click **💾 Save Profile** in the header to save your About Me, resume, and LinkedIn text to browser storage. It auto-loads next time you open the agent in the same Claude conversation or artifact context.

### Clearing the cache

Click **🗑 Clear cache** in the header to wipe all saved and entered materials and start fresh.

---

## What the agent will not do

- **Invent facts.** Every bullet traces to your source materials. If a proof point isn't there, it isn't written.
- **Inflate verbs.** "Coordinated" is not upgraded to "led." The verb matches what actually happened.
- **Manufacture matches for gaps.** If you don't have a required skill, that gap is flagged — not papered over with adjacent language.
- **Move accomplishments between employers.** Work done at Company A stays at Company A.
- **Claim ownership of work you coordinated or contributed to.** The framing matches the actual level of involvement.

---

## How the agent works (for the curious)

The agent makes direct calls to the Anthropic API from inside the Claude artifact sandbox. It uses two models:

- **Claude Haiku** — fast analysis, fact extraction, scoring
- **Claude Sonnet** — all resume and document generation

On a paid Claude account, outputs run in parallel. On a free account, they run sequentially with automatic retry on rate limits.

Your materials are sent to Anthropic's API as part of the prompt. Nothing is stored on any server — the only persistence is the browser storage in the Claude artifact, which you can clear at any time.

---

## Troubleshooting

**"Analysis failed" error**
Try enabling Concise mode (checkbox near the Analyze button) and running again. Long inputs can occasionally cause JSON parse issues on the first attempt.

**Outputs are cut off**
The agent is optimized for two-page resumes. Very long source materials can push against token limits. Trim your About Me to the most relevant 800–1000 words for the target role.

**Rate limit errors on free tier**
The agent detects rate limits and switches to sequential mode automatically. If it keeps failing, wait 30 seconds and try again with fewer outputs selected.

**The UI doesn't appear after uploading**
Make sure you're uploading the `.html` file directly to Claude chat (not pasting the contents). The artifact renderer needs the file, not the raw HTML text.

---

## License

MIT — use it, share it, modify it.

---

*Built with Claude Sonnet. Runs on Claude.ai.*

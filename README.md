# 🦸 Superhero Database Telegram Bot

A Telegram bot powered by an AI agent that answers natural-language questions about 731 superheroes and supervillains — built entirely in [n8n](https://n8n.io) using an LLM Tools Agent, a local character database, and structured output parsing.

Ask it things like:
- "Who is Batman?"
- "How strong is the Hulk?"
- "What does Spider-Man look like?"
- "What about his speed?" *(follow-up, thanks to conversational memory)*

The bot replies with a tailored text answer **and** the character's image.

👨‍💻 **Created by:** Pranav Choubey — 🔗 [github.com/PRANAV4248](https://github.com/PRANAV4248/)

---

## How it works

This project is split into two n8n workflows:

### 1. Data Extraction (`Superhero extraction.json`)
Builds the local character database:
1. Generates a list of character IDs (1–731)
2. Loops through each ID one at a time (`Split in Batches`) and fetches character data from the [Superhero API](https://superheroapi.com)
3. Cleans the raw API response — the source data uses inconsistent placeholders (`"null"` strings, `"-"` for empty fields) which are normalized into real `null` values
4. Splits combined fields like height/weight (`["5'8\"", "173 cm"]`) into separate imperial/metric columns
5. Inserts each cleaned record into an n8n **Data Table**

### 2. AI Agent (`Superhero AI.json`)
Handles live Telegram conversations:
1. **Telegram Trigger** receives incoming user messages
2. An **AI Agent** (running on Groq) interprets the query, normalizes character names (e.g. "ironman" → "Iron Man"), and decides whether to look up a character
3. The agent queries the **Superhero database** (n8n Data Table) as a callable tool, filtering by `name` or `full_name`
4. A **Structured Output Parser** enforces a consistent response shape (`reply`, `hero_found`, `hero_name`, `image_url`)
5. **Simple Memory** (per-user, windowed) allows natural follow-up questions without repeating the character's name
6. The final response is sent back via Telegram as both a **text message** and a **photo** of the character

---

## Architecture

```
Data Pipeline:
Manual Trigger → Generate ID List → Loop Over Items → Fetch Character (HTTP)
   → Clean & Normalize Fields → Insert into Data Table

Chat Pipeline:
Telegram Trigger → AI Agent ─┬─ Chat Model (Groq)
                              ├─ Memory (per-user, windowed)
                              ├─ Tool: Superhero Database (Data Table lookup)
                              └─ Structured Output Parser
                    → Send Text Message
                    → Send Photo
```

---

## Tech stack

| Component | Tool |
|---|---|
| Automation / orchestration | [n8n](https://n8n.io) |
| LLM | Groq (Llama / GPT-OSS models) |
| Data source | [Superhero API](https://superheroapi.com) |
| Storage | n8n Data Table |
| Messaging | Telegram Bot API |

---

## Setup

### Prerequisites
- A running n8n instance (cloud or self-hosted)
- A [Superhero API](https://superheroapi.com) access token (GitHub login)
- A Telegram bot token ([@BotFather](https://t.me/BotFather))
- A Groq API key (or swap in your preferred LLM provider)

### Steps
1. Import both workflow JSON files into n8n
2. Create a Data Table named `superheroes` with the columns listed in [`Superhero extraction.json`](./Superhero%20extraction.json)
3. Add your credentials:
   - Superhero API token → HTTP Request node in the extraction workflow
   - Telegram Bot API token → both Telegram nodes
   - Groq API key → Chat Model node
4. Run the **Superhero extraction** workflow once (manual trigger) to populate the Data Table — this takes a while due to 731 sequential API calls
5. Activate the **Superhero AI** workflow
6. Message your bot on Telegram to test

---

## Lessons learned

- **Unfiltered tool queries can silently blow past LLM token limits.** An early version of the Data Table tool had no filter condition, returning up to 50 rows per lookup — filtering and limiting results at the tool level is as important as the query logic itself.
- **Structured output + tool-calling agents don't behave identically across LLM providers.** Some Groq models failed to correctly invoke the output parser's internal formatting step; swapping models resolved it.
- **Prompting an agent on what *not* to include** (e.g. don't show power stats unless explicitly asked) required more iteration than getting it to fetch data correctly in the first place.
- **Manual loop-based iteration is fragile.** An early version of the extraction workflow used a hand-rolled ID-increment loop, which caused duplicate inserts and a stack overflow. Switching to n8n's built-in `Split in Batches` (Loop Over Items) node fixed both issues.

---

## Disclaimer
Character data is sourced from a third-party API and may contain inaccuracies. This project is for educational/personal use.

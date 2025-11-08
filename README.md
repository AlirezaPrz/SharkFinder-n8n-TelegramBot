# SharkFinder — Telegram Intake & Matching

n8n-powered Telegram bot for startup–investor matchmaking. Users pick a role, send text or a voice note, and the bot saves structured profiles to Google Sheets, then returns **ranked matches** using a simple, explainable scoring model.
Includes a workflow JSON ready to import.

---

## 🎥 Demo

https://github.com/user-attachments/assets/cf721230-98bd-4618-88e0-d3ad5b556617

---

## ✨ Features

* **Telegram onboarding** with `/start` and role selection (Business / Investor)
* **Text or voice** inputs (voice is auto-transcribed with Gemini)
* **Google Sheets** backend with 3 tabs: `UserState`, `Businesses`, `Investors`
* **AI Intake Agent (Gemini)** parses free text/voice → structured fields
* **Matching engine (JavaScript)** ranks:
  * Sector fit
  * Stage compatibility
  * Funding needed ↔ ticket size overlap
  * Location fit

---

## 🧭 Architecture & Flow

1. **Telegram Trigger** → **Switch** routes:
   * `/start` → “Welcome” + role buttons
   * Callback “Business” → instructions → `UserState` upsert `{Chat_id, role=business, stage=In_progress}`
   * Callback “Investor” → instructions → `UserState` upsert `{Chat_id, role=investor, stage=In_progress}`
   * **Text or Voice** → normalize → look up user role → **AI Agent (Gemini)** → **Switch** by role
2. **By role**:
   * **Business input**: save/merge into `Businesses`, then pull `Investors`, run **Code (JS)** → send top matches
   * **Investor input**: save/merge into `Investors`, then pull `Businesses`, run **Code (JS)** → send top matches

**Key nodes (by purpose)**

* Trigger & UX: `Telegram Trigger`, `Send a Welcome message`, `Send Instructions`, `Send Instructions1`
* Intake & parsing: `Google Gemini (audio STT)`, `AI Agent (Gemini chat)`, `Simple Memory`
* Storage: `Add the business to DB` / `Add the investor to DB` (`UserState`), `write_business_profile`, `write_investor_profile`
* Matching: `list_investors`, `list_businesses`, `get_user_business`, `get_user_investor`, `Code in JavaScript`, `Code in JavaScript1`
* Delivery: `Send a text message`, `Send a text message1`

---

## 📊 Google Sheets Schema

Create a single spreadsheet (the “DB”) with **three** sheets:

### `UserState`

| Column       | Example                 | Notes                     |
| ------------ | ----------------------- | ------------------------- |
| `Chat_id`    | `123456789`             | Telegram chat id (string) |
| `role`       | `business` | `investor` | Selected role             |
| `updated_at` | `2023-12-18`            | Auto-timestamp            |
| `stage`      | `In_progress`           | Basic state marker        |

### `Businesses`

| Column           | Example           |
| ---------------- | ----------------- |
| `chat_id`        | `123456789`       |
| `name`           | `Acme Robotics`   |
| `industry`       | `Robotics; AI`    |
| `stage`          | `seed`            |
| `funding_needed` | `300k–800k USD`   |
| `location`       | `Toronto, Canada` |
| `email`          | `founder@acme.ai` |

### `Investors`

| Column        | Example              |
| ------------- | -------------------- |
| `chat_id`     | `123456789`          |
| `name`        | `North Star Capital` |
| `sector`      | `AI; DevTools`       |
| `stage`       | `pre-seed/seed`      |
| `ticket_size` | `200k–500k USD`      |
| `location`    | `Canada / US`        |
| `email`       | `partner@ns.vc`      |

> The workflow uses **upsert by chat_id** for profile rows and by `Chat_id` for `UserState`.

---

## 🔐 Credentials & Environment Variables

### n8n Credentials (create in n8n UI)

* **Telegram**: *Telegram account* (BotFather token)
* **Google Sheets OAuth2**: *Google Sheets account* (with access to your spreadsheet)
* **Gemini API**: *Google Gemini (PaLM) API account* (API key)

> When you import the workflow, **relink** these credentials to your own.

---

## 🚀 Setup

1. **Clone & import the workflow**

```bash
git clone https://github.com/AlirezaPrz/SharkFinder-n8n-TelegramBot.git
# In n8n: Import ./workflows/TelegramBot.json
```

2. **Create credentials** in n8n:

* Telegram (Bot token)
* Google Sheets OAuth2
* Gemini API key

3. **Set environment variables** shown above and **restart** n8n.

4. **Create the Google Sheet** with the three tabs (`UserState`, `Businesses`, `Investors`) and headers.

5. **Activate** the workflow and DM your bot `/start`.

---

## 🧠 Intake Agent

* Receives two inputs from n8n: `role` and `inputText` (or voice→transcript)
* Extracts structured fields and calls exactly **one tool**:
  * `write_business_profile` **or** `write_investor_profile`
* Tool calls upsert to the corresponding sheet using `chat_id` as the key
* System prompt enforces one-turn, tool-only behavior to keep runs cheap and predictable

---

## 🧮 Matching Logic

Implemented in `Code in JavaScript` nodes:

* **Normalization:** lower-case, trim; simple stage aliases (`idea, pre-seed, seed, series a/b/c, growth, late`)
* **Funding/Capital parsing:** `200k–500k`, `1.2m` recognized; overlap or ~±30% proximity earns points
* **Sector fit:** exact or partial match; a few cross-sector pairings (e.g., fintech↔payments, ai↔saas/mlops/devtools)
* **Location fit:** exact/city/region/global acceptance
* **Weights:** `sector:4, stage:3, capital:5, location:2` (normalized to 0–100)
* **Output:** Top 10 matches with name, sector, stage, capital, location, contact (Telegram HTML formatting; messages chunked <3.5k chars)

> You can tune weights or add more cross-sector mappings inside the JS code nodes.

---

## 🧰 Troubleshooting

* **No matches returned:** ensure your sheet tabs & headers match exactly; make sure `chat_id` exists for your row.
* **Sheet read/write errors:** relink Google Sheets OAuth2 in n8n; confirm the account has access to the spreadsheet.
* **Telegram not responding:** re-create the Telegram credential and re-activate the workflow; verify webhook is set (n8n handles this).
* **Agent not working:** make sure you still haven't hit the limit for the model you're using.

---

## 🗺️ Roadmap (nice-to-haves)

* Admin panel: approve/ban users, edit profiles, manual match
* Adding more columns to the database
* Using SQL databases instead of Google Sheets
* Using vector embeding for the matching the users
* Adding a feature so users can update their profile
* Adding a multi-turn conversation flow for the agent
* Adding error triggers to the workflow

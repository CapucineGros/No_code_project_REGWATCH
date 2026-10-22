# RegWatch

RegWatch is a Telegram bot that helps bank risk and ALM teams follow ECB publications. Each user chooses the topics they care about (for example liquidity or IRRBB). The bot reads the ECB press feed, removes duplicates, asks an AI model to rate each new publication against the user's topics, and sends only the relevant ones with a one-sentence summary and the link.

Built with n8n for the course "No code software development". Level 2 (intermediate), with some level 3 elements: sub-workflows, retries, an error workflow and state kept between runs.

## The need

Risk and ALM teams must follow ECB press releases, speeches and interviews on precise topics. Today they scan long feeds by hand, which is slow and makes it easy to miss something important. RegWatch does that filtering for them and delivers the result in the chat tool they already use.

## What the bot does

| Command | What happens |
|---|---|
| `/start` | Welcome message that explains the bot and its commands |
| `/topics liquidity, IRRBB` | Saves the user's topics (replaces the previous ones). An empty `/topics` returns a usage message |
| `/fetch` | Checks the ECB feed now and sends the relevant new publications |
| `/stop` | Removes the user from the subscribers |
| anything else | Usage message |

The same pipeline also runs automatically every day at 08:00 (Europe/Paris) for every subscriber.

If nothing new was found, or nothing was relevant, the bot says so instead of staying silent.

## Architecture

Summary of the flow:

1. A Telegram message (or the 08:00 schedule) triggers the main workflow.
2. A Switch node routes the command.
3. For `/fetch` and the schedule, the main workflow calls the sub-workflow **Process subscriber** once per subscriber.
4. Process subscriber reads the ECB RSS feed, drops items already sent to that user (matching on link and on title), and keeps at most 3 new items.
5. Each item goes to the sub-workflow **Score item**, where an LLM chain (Gemini) returns a score from 0 to 10, a matched topic and a one-sentence summary. The answer is validated, and an invalid answer falls back to a score of 0.
6. Items with a score of 6 or more are sent on Telegram. Every scored item is logged, so it is never processed twice for the same user.
7. Any failed run of the main workflow triggers the **Error alert** workflow, which sends the maintainer a Telegram message.

## Repository content

```
.
├── docker-compose.yml        n8n + Postgres (self-hosted setup)
├── .env.example              environment variables, placeholders only
├── .gitignore
├── workflows/                exported n8n workflows (JSON)
│   ├── main.json
│   ├── process_subscriber.json
│   ├── score_item.json
│   └── error_alert.json
└── README.md
```

## What you need

| Service | Used for | Cost |
|---|---|---|
| n8n (Cloud or self-hosted) | Running the workflows | Cloud trial, or free when self-hosted |
| Telegram account and a bot token from @BotFather | The chat interface | Free |
| Google account (Google Sheets, OAuth) | Storing subscribers and sent items | Free |
| Google Gemini API key (from aistudio.google.com) | Scoring and summarising | Free tier |

## Setup

### 1. Telegram bot

1. In Telegram, open @BotFather and send `/newbot`.
2. Choose a name and a username ending in `bot`.
3. Copy the token it gives you. Keep it private.

### 2. Google Sheet

Create a Google Sheet named `RegWatch` with two tabs. The header spelling matters.

| Tab | Row 1 headers |
|---|---|
| `Subscribers` | `chat_id`, `topics` |
| `Sent` | `chat_id`, `link`, `title` |

### 3. n8n

Option A, n8n Cloud (used for the demo): create an account at n8n.io.

Option B, self-hosted with Docker:

```bash
git clone <this-repo-url>
cd regwatch
cp .env.example .env
# edit .env, then generate a key:  openssl rand -hex 32
docker compose up -d
```

n8n is then available at `http://localhost:5678`. Telegram needs a public HTTPS address to reach n8n, so set `WEBHOOK_URL` in `.env` to a tunnel URL (ngrok or cloudflared).

> Note: the demo of this project ran on n8n Cloud. The Docker setup is provided for self-hosting but was not tested end to end within the project deadline.

### 4. Environment variables (Docker only)

| Variable | Meaning |
|---|---|
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | Database credentials used by n8n |
| `N8N_VERSION` | Pinned n8n image version |
| `N8N_ENCRYPTION_KEY` | Encrypts the credentials stored by n8n. Keep it stable |
| `WEBHOOK_URL` | Public HTTPS URL that Telegram can call |

`.env` is ignored by Git. Only `.env.example` is committed.

### 5. Import the workflows

Import in this order (Workflows, then Import from file):

1. `Score item.json`
2. `Process subscriber.json`
3. `RegWatch - Error alert.json`
4. `RegWatch - Main.json`

Then:

1. Create the credentials in n8n: Telegram (bot token), Google Sheets (OAuth) and Google Gemini (API key). Assign them to the nodes that show a missing credential.
2. Point every Google Sheets node to your `RegWatch` sheet.
3. In the Execute Workflow nodes, reselect the sub-workflows (their IDs change after an import).
4. In `RegWatch - Main`, open Settings and set the Error Workflow to `RegWatch - Error alert`. Set the timezone to Europe/Paris.
5. In `RegWatch - Error alert`, set your own Telegram chat ID.
6. Publish all four workflows. Sub-workflows must be published to be called.

## How to test

You can fire every part on demand from Telegram.

1. Send `/start`. You should get the welcome message.
2. Send `/topics climate`. You should get a confirmation, and a row should appear in the `Subscribers` tab.
3. Send `/fetch`. You get a waiting message first, then the relevant publications. The run can take a few minutes (see limitations). The `Sent` tab fills up.
4. Send `/fetch` again. Publications already logged are not sent again.
5. Send `/topics` with nothing after it. You should get a usage message and no new row.
6. Send `/stop`. The row disappears from `Subscribers`.

To test the error path, change the RSS URL in `Process subscriber` to a wrong one, republish, and send `/fetch`. The error alert arrives on Telegram. Put the real URL back afterwards:

```
https://www.ecb.europa.eu/rss/press.xml
```

The Executions tab in n8n shows every run and the data that went through each node. Look there first when something fails.

## Design choices

- **One shared pipeline.** `/fetch` and the daily schedule both call Process subscriber, so the logic exists once.
- **Scoring isolated in a sub-workflow.** Score item can be tested and changed without touching the rest.
- **Deduplication per user, on link and on title.** The feed can list one publication under two links, so matching on the link alone is not enough.
- **Every scored item is logged**, including the irrelevant ones. Otherwise the same items would be picked again at every run.
- **Google Sheets as the data store.** Enough for a small internal tool and easy to show on screen. A relational database would replace it in production.

## Course requirements covered

| Requirement | Where |
|---|---|
| Message trigger (not Chat Message) | Telegram Trigger in the main workflow |
| If or Set node | `Has new?`, `Relevant?` and the topics validity check |
| Code node | Topics parsing, deduplication, AI answer validation, final check |
| Router or Switch | Switch with five routes in the main workflow |
| API connection with stored credential | Telegram, Google Sheets, Gemini, RSS feed |
| Database for persistence | Google Sheets (see limitations) |
| AI node with written prompt | LLM Chain with a Gemini model in Score item |
| Sub-workflow | Process subscriber and Score item |
| Error handling and retries | Retry On Fail on the sub-workflow calls, Error alert workflow |

## Known limits

- The score and the summary are based on the publication title only, not on the article text. The link in each message lets the user check.
- The score is the model's own judgement. Nothing calibrates what a 4 or an 8 means, and the threshold of 6 is a choice.
- The Gemini free tier allows 5 requests per minute, so each run scores at most 3 items and `/fetch` can take up to about 3 minutes.
- Items logged for a user are not rescored when that user changes topics.
- A rejected or failed AI answer gets a score of 0 and is still logged, so that item is not retried later.
- The bot has no authentication: anyone who finds it can subscribe.
- Google Sheets is not a real database and does not scale to many users.

## Possible improvements

- Pass the feed description or the article text to the AI, not only the title.
- Add a second source (EBA) and a daily digest.
- Add a scoring scale in the prompt and use a lower temperature.
- Replace Google Sheets with Postgres, which is already part of the compose file.
- Use a paid AI tier to remove the limit of 3 items per run.
- Restrict the bot to an allowlist of users.

## Screenshots

Add your screenshots to `docs/` and reference them here:

- `docs/screenshot-demo.png`: a Telegram conversation with a successful `/fetch`
- `docs/screenshot-executions.png`: the n8n Executions tab with green runs
- `docs/screenshot-sheet.png`: the Subscribers and Sent tabs
Multi-Inbox Email Digest
Problem: managing 4 work email addresses under one domain, with no easy way to get a single overview across all of them.
Approach: built the same solution two ways — an n8n workflow and a custom Claude Code pipeline — to compare a low-code vs. code-first architecture for the same task. Both pull from all 4 Gmail inboxes via the Gmail API, generate one combined summary using the Claude API, post it to Slack every morning, and store the raw emails in a database that's queryable directly through Claude.
n8n
Claude Code
Style
Visual workflow, low-code
Custom scripts
Hosting
Small VPS or n8n Cloud
Free (GitHub Actions) or small VPS
Best for
Non-technical maintenance
Cheaper, more customizable
Folder
n8n/
claude-code/
Full architecture writeup and cost comparison: see docs/architecture.md.
Repo structure
email-digest/
  README.md                    ← you are here
  .gitignore
  .env.example                 ← copy to .env, fill in real values, never commit .env
  docs/
    architecture.md            ← full approach comparison + cost breakdown
  n8n/
    workflow.json              ← exported n8n workflow (add once built)
  claude-code/
    fetch_emails.py
    summarize.py
    store.py
    post_to_slack.py
    main.py
    requirements.txt
    .github/workflows/daily-digest.yml
​
Setup
0. Google Cloud / Gmail API (required for both approaches)
This is the one part that's identical no matter which path you pick, and the main upfront effort in the whole project.
Note on naming: this uses Google's own OAuth 2.0 / Gmail API — not Auth0 (a separate, unrelated identity product from Okta).
Create a Google Cloud project at console.cloud.google.com (e.g. "email-digest"). No billing required — Gmail API has a generous free quota.
Enable the Gmail API: APIs & Services > Library → search "Gmail API" → Enable.
Configure the OAuth consent screen: APIs & Services > OAuth consent screen.
User type: Internal if all 4 addresses are on a Google Workspace domain you administer (skips Google's verification review). Otherwise External, and add all 4 addresses as test users.
Scopes: gmail.readonly is sufficient — avoid requesting gmail.modify or gmail.send unless actually needed.
Create OAuth 2.0 credentials: APIs & Services > Credentials > Create Credentials > OAuth client ID → type Web application. Save the Client ID and Client Secret into your local .env (never commit them).
Authorize each of the 4 accounts. Each inbox needs its own consent flow — one Client ID/Secret is reused, but each of the 4 accounts has to individually authorize and produce its own refresh token. (Shortcut if you administer the Workspace domain: domain-wide delegation with a service account skips the per-account clicking — optional, the per-account flow works regardless.)
Store the results: 1 Client ID, 1 Client Secret, 4 refresh tokens. All go in .env (see .env.example), never in code or git history.
A. n8n approach
Host n8n — self-hosted on a small VPS (Docker) for the cheapest option, or n8n Cloud for zero server management.
Add Google OAuth2 credentials in n8n — one per inbox (4 total), using the Client ID/Secret from step 0.4. n8n shows you its redirect URI; add that back into the Google Cloud credential's authorized redirect URIs.
Build the workflow:
Schedule Trigger (daily, e.g. 7:00am)
4× Gmail nodes (one per credential), fetching recent/unread mail
Set node on each to tag source_inbox, then Merge into one list
HTTP Request node → Claude API (api.anthropic.com/v1/messages) → ask for one unified digest, not 4 separate summaries, grouped by urgency and noting source inbox
Postgres/Supabase node → insert raw emails (with source_inbox) for later querying
Slack node → post the digest
Enable "chat with the data" in Claude — connect an off-the-shelf Postgres/Supabase MCP connector to the same database.
Test manually, then turn the schedule on.
Export the finished workflow as n8n/workflow.json for this repo.
B. Claude Code approach
Set up the claude-code/ folder structure shown above.
pip install google-api-python-client google-auth google-auth-oauthlib anthropic requests
Write a one-time local script (using google-auth-oauthlib) to run the OAuth flow per inbox and print a refresh token — run it 4 times, one per account.
fetch_emails.py — loops over the 4 credential sets, pulls recent messages via the Gmail API, tags each with source_inbox.
summarize.py — sends the combined list from all 4 inboxes to the Claude API in one call, asking for a single unified digest.
store.py — writes raw emails to SQLite (simplest) or hosted Postgres (Supabase/Neon free tier, better for remote querying).
post_to_slack.py — posts the digest to a Slack Incoming Webhook.
main.py — orchestrates fetch → store → summarize → post.
Scheduling: a GitHub Actions workflow (.github/workflows/daily-digest.yml) running main.py on a daily cron is free at this volume. Add all secrets (Client ID/Secret, 4 refresh tokens, Anthropic API key, Slack webhook URL) under Settings > Secrets and variables > Actions — never in code.
Write a minimal MCP server exposing a query tool over the same database, and register it in Claude Desktop (or claude.ai connector settings) so the data is chat-queryable.
Test end-to-end locally, then push and confirm the Actions workflow runs on schedule.
Cost
n8n
Claude Code
Hosting
~$5–10/mo (VPS) or ~$20–24/mo (n8n Cloud)
$0 (GitHub Actions free tier) or ~$5–10/mo (VPS)
Database
Free tier (Supabase/Neon)
Free (SQLite) or free tier (Supabase/Neon)
Claude API (summarization)
Cents to ~$1/day depending on volume
Same
Slack
Free
Free
Rough daily total
~$0.50–2/day
~$0–1.50/day
Notes
Part 0 (Google Cloud setup) requires manual browser clicks — no AI can complete the OAuth consent step, since it needs the actual Google login. Everything after that (building the n8n workflow, or writing/running the Claude Code scripts) is where an AI assistant can do the heavy lifting.
Never commit .env, any downloaded credentials JSON, or token files — see .gitignore.

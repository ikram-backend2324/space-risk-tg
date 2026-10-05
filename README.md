# SPACE RISK Telegram bot

One-file implementation for Python 3.10+, using aiogram 3.26.0.

## Local setup (Windows)
1. Create a bot using @BotFather → /newbot and copy its token.
2. Put `space_risk_bot.py`, `requirements.txt` and `.env` in one folder. The separately supplied `space-risk.env` has identical content: rename it to `.env` if needed.
3. Set BOT_TOKEN in `.env`. Set OPENROUTER_API_KEY only if you want AI explanations and chat. An empty key still supports forecasts using the deterministic engine.
4. Open a terminal in this folder:

```powershell
py -3.10 -m venv .venv
.venv\Scripts\activate
python -m pip install -r requirements.txt
python space_risk_bot.py
```

Leave MODE=polling locally. `/start` opens the language picker. `/forecast`, `/regions`, `/history`, `/language`, `/help` are registered automatically. Karakalpak users get default Uzbek command descriptions because Telegram does not offer a three-letter Karakalpak command language code.

## Render deployment
Use a separate bot web service; SITE_URL points to the existing Django site, whereas WEBHOOK_BASE_URL points to this bot service.

1. Upload the Python file and requirements.txt to your GitHub repository. Do not upload `.env` or the SQLite database.
2. Create a Python Web Service from the repository.
3. Build command: `pip install -r requirements.txt`.
4. Start command: `python space_risk_bot.py`.
5. Set BOT_TOKEN, MODE=webhook, SITE_URL=https://space-risk.onrender.com, WEBHOOK_BASE_URL=https://YOUR-BOT-SERVICE.onrender.com.
6. Generate WEBHOOK_SECRET locally: `python -c "import secrets; print(secrets.token_urlsafe(32))"`. Copy its output into Render's environment settings.
7. Set DATABASE_URL to your PostgreSQL connection URL. Local SQLite on an ephemeral Render filesystem will not persist across restarts or deployments.
8. Optionally set OPENROUTER_API_KEY, OPENROUTER_MODEL, OPENROUTER_MODEL_KAA (a stronger model for Karakalpak chat) and ADMIN_IDS (comma-separated Telegram IDs).
9. Health check path: `/`. Set a Render-supported Python 3.10 patch release or newer Python version in the service settings.
10. Deploy and send `/start` to the bot. Do not simultaneously run polling with the same token.

Webhook path and Telegram's secret header are both verified. Access logging is disabled to keep the webhook secret out of request logs. `/stats` and `/broadcast text` are restricted to ADMIN_IDS. Broadcast text is escaped and throttled.

## Scientific and operational limits
The supplied baseline values and trend formula are scenario indices. They are not calibrated probabilities, real-time warnings, validated forecasts, or a satellite data pipeline. This implementation does not download satellite observations. Progress messages and result disclaimers explicitly reflect that. AI adds commentary to deterministic scores, keeping the same numerical basis whether the AI service is available or not. AI confidence is subjective and explicitly labeled unverified.

Radar charts require at least three dimensions; one or two selected hazards use a readable grouped bar chart. API ranking uses website scores when available; the forecast engine uses the separately supplied per-hazard baseline table because the website API provides only a composite score. The API cache lasts one hour. Failed refreshes reuse stale cached data, then built-in data.

FSM state, recent chat messages, rate limits and the API cache are in memory. Restarting clears active wizards and conversations, while PostgreSQL preserves users and forecasts. Use a single instance. The provided per-user limit is one forecast start per 20 seconds; simultaneous generation is capped at two. Free services may sleep; a health endpoint does not prevent sleeping. Production continuity needs persistent storage, backups and suitable hosting capacity.

Forecast JSON is validated, numbers are clamped and all AI text is escaped before Telegram HTML output. Language instructions are sent for every AI call, but model compliance and Karakalpak wording should be reviewed by a native speaker before a public presentation.

## Architecture
Settings → SQLAlchemy async storage → localized aiogram Router/FSM → website API + deterministic engine + OpenRouter commentary → matplotlib image → Telegram result. Aiohttp serves webhook requests and the health endpoint in webhook mode.

## Consistency with the website and Android app
- Forecast texts (summary, key drivers, recommendations) come from the website's hand-checked catalog via `GET /api/v1/scenario/?region=&horizon=&hazards=&lang=`, so the bot, website and app say the same thing in all 4 languages. If the site is unreachable, the bot's built-in text is used.
- Karakalpak never uses AI-written forecast text (models mix it with Uzbek/Kazakh); Karakalpak chat answers get a style guide and can use `OPENROUTER_MODEL_KAA`.
- Each hazard has the same colour as on the website/app (charts: points and labels); "today" is grey/dashed, the forecast is cyan.

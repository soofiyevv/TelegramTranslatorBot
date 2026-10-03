# Telegram Translator Bot

A Telegram bot that translates text, photos with text, and voice messages into 22 languages. It detects the source language automatically, works in private chats and, through inline mode, in any chat, and is powered by the Gemini API.

Bot: [@ostralik_bot](https://t.me/ostralik_bot)

## Features

- **Text translation.** Send a message and get the translation back.
- **Photo translation.** Send a photo with text (menus, signs, labels, letters). The bot reads the text, translates it, and shows the detected language.
- **Voice and audio translation.** Send a voice message or an audio file. The bot transcribes it, shows the detected language and the original transcript, and translates it.
- **Inline mode.** Type `@ostralik_bot your text.` in any chat and pick the translation. The text must end with `.`, `!` or `?`.
- **22 target languages**, chosen with `/lang` (the choice is saved per user).
- **Fallback chain for text:** Gemini, then Google Translate, then MyMemory, so text keeps working when one service fails.
- **Prompt-injection protection.** Every Gemini prompt starts with a security rule that treats the user's content as data to translate, never as a command, with a safe `UNSAFE_REQUEST` way out if the content tries to hijack the model.
- **Automatic retry on Gemini overload.** A `503` ("high demand") response is retried once before falling back.
- **Daily limits per user** and size limits to protect the free quotas.

## Commands

| Command | Description |
|---|---|
| `/start` | Greeting and language selection |
| `/lang` | Change the target language |
| `/help` | Usage, limits and privacy note |

## Supported languages

English, Russian, Hungarian, German, Spanish, French, Italian, Ukrainian, Polish, Turkish, Chinese (Simplified), Japanese, Azerbaijani, Korean, Arabic, Kazakh, Czech, Georgian, Hindi, Urdu, Bengali, Vietnamese.

## Limits

| Limit | Value | Setting in `bot.py` |
|---|---|---|
| Requests per user per day (text, photo, audio, inline) | 100 | `DAILY_LIMIT` |
| Text length | 2000 characters | `MAX_CHARS` |
| Audio length | 2 minutes | `MAX_AUDIO_SEC` |
| Audio file size | 15 MB | `MAX_AUDIO_BYTES` |

Supported audio formats: Telegram voice messages (OGG), MP3, WAV, FLAC, AAC.

## How it works

```
Text   -> Gemini -> (on error) Google Translate -> (on error) MyMemory
Photo  -> Gemini (reads the text and translates it)
Audio  -> Gemini (transcribes and translates)
```

- The bot uses **long polling** (`aiogram` 3), so it needs a permanently running process but no open ports or domain.
- User settings (target language and daily counter) are stored in a local **SQLite** file, `bot.db`, created automatically.
- Photos and audio are processed in memory and are not saved to disk.
- Gemini is called through the `google-genai` SDK. Google Translate and MyMemory are used through `deep-translator`.

## Prompt engineering

Every Gemini prompt (`build_text_prompt`, `build_image_prompt`, `build_audio_prompt`) combines four techniques:

1. **Personality / role prompting** — the model is told it is a professional translator with a warm, natural voice (`PERSONA`).
2. **Zero-shot instruction** — a direct task description with explicit rules (keep meaning, tone, emojis, names, numbers; translate idioms by meaning).
3. **Few-shot examples** — sample input/output pairs for text, and sample output formats for photos and audio.
4. **Chain of thought** — the model follows numbered steps silently and prints only the final result.

A **security rule** is placed first in every prompt (highest priority) and repeated in the chain-of-thought step: the user's content, however phrased, is always data to translate, never a command. If the content is a clear attempt to hijack the model (prompt injection), it must reply with `UNSAFE_REQUEST` instead of following the embedded instruction. The handlers check for this marker the same way they check for `NO_TEXT` / `NO_SPEECH`.

## Generation parameters

Stored in one place, `GEN_PARAMS` in `bot.py`:

| Parameter | Text | Photo | Audio | Purpose |
|---|---|---|---|---|
| `temperature` | 0.3 | 0.1 | 0.1 | Low values keep translation accurate and repeatable; text is slightly higher so idioms sound natural. |
| `top_p` | 0.9 | 0.9 | 0.9 | Nucleus sampling, cuts off unlikely word choices. |
| `max_output_tokens` | 2048 | 4096 | 4096 | Photo/audio answers also contain the detected language and the original text, so they get a bigger budget. |

## Reliability

- **Gemini overload (`503`).** `_call_gemini()` retries once after a short pause when Gemini answers "high demand", since this is usually a temporary spike, not a quota problem.
- **MyMemory error text.** MyMemory (the last fallback) sometimes returns an error message as if it were a translation (for example `'AUTO' IS AN INVALID SOURCE LANGUAGE...`). `mymemory_translate()` detects this and raises an exception instead, so the user sees a clean "couldn't translate" message rather than a technical error string.
- Every external call is wrapped in `try/except`; a failure is logged and the next fallback is tried.

## Requirements

- Python 3.10 or newer
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- A Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey) (optional, see below)

Without `GEMINI_API_KEY` the bot still translates text through Google Translate and MyMemory, but photo and audio translation are disabled.

## Project structure

```
translator-bot/
├── bot.py             # the whole bot
├── requirements.txt   # dependencies
├── .env               # secrets (never commit this file)
├── .gitignore
└── bot.db             # created at first run
```

`requirements.txt`:

```
aiogram>=3.4
deep-translator
python-dotenv
google-genai
```

`.gitignore`:

```
.env
venv/
bot.db
__pycache__/
```

## Configuration

Create a `.env` file next to `bot.py`, without quotes or spaces around `=`:

```
BOT_TOKEN=your_telegram_bot_token
GEMINI_API_KEY=your_gemini_api_key
```

The Gemini model is set in `bot.py`:

```python
GEMINI_MODEL = "gemini-3.5-flash-lite"
```

Model names and availability change over time (Google has retired and replaced models more than once during this project). If you get a `404 NOT_FOUND` error, open Google AI Studio, pick a currently available model, and update this line. If you get repeated `503 UNAVAILABLE` errors, the model is temporarily overloaded on Google's side (common on the free tier, since free-tier requests use "sheddable" capacity) — this is not a bug in the bot and usually resolves on its own.

## Run locally

Windows (PowerShell or cmd):

```powershell
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
python bot.py
```

macOS / Linux:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python bot.py
```

When the log shows `Run polling for bot @your_bot`, the bot is running. Only **one** copy of the bot may run with the same token at a time, otherwise Telegram returns `Conflict: terminated by other getUpdates request`.

## Enable inline mode

In [@BotFather](https://t.me/BotFather): `/mybots` → your bot → **Bot Settings** → **Inline Mode** → **Turn on**. Set a short placeholder text for the input field when asked.

## Deploy on a Linux server (Ubuntu, systemd)

1. Install Python tooling and create the project folder:

   ```bash
   sudo apt update && sudo apt install -y python3-venv
   mkdir -p ~/translator-bot
   ```

2. Copy `bot.py`, `requirements.txt` and `.env` to the server (for example with `scp`), then install dependencies:

   ```bash
   cd ~/translator-bot
   python3 -m venv venv
   venv/bin/pip install -r requirements.txt
   ```

3. Create `/etc/systemd/system/translator-bot.service`:

   ```ini
   [Unit]
   Description=Telegram translator bot
   After=network-online.target
   Wants=network-online.target

   [Service]
   User=ubuntu
   WorkingDirectory=/home/ubuntu/translator-bot
   ExecStart=/home/ubuntu/translator-bot/venv/bin/python bot.py
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

   Adjust `User` and the paths to match your server.

4. Enable and start the service:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now translator-bot
   sudo systemctl status translator-bot
   ```

5. Read the logs with `journalctl -u translator-bot -f`.

**Updating the bot:** copy the new `bot.py` to the server, verify it compiles (`venv/bin/python -m py_compile bot.py`), then run `sudo systemctl restart translator-bot`. Stop any other copy running with the same token first.

## Adding a language

Add the language in two places in `bot.py`, then restart the bot:

1. `LANGS`: add an entry such as `"pt": "🇵🇹 Português"`. The code must be supported by Google Translate.
2. `MYMEMORY_CODES`: the same code with a region, for example `"pt": "pt-PT"`. Without it the fallback translator will use English.
3. Restart the bot. The new button appears in `/lang` automatically.

## Security

- Never publish `.env`. If the bot token leaks, use `/revoke` in BotFather. If the Gemini key leaks, delete it in Google AI Studio and create a new one.
- Photos, audio and texts are sent to Google services for processing. On the free tier of the Gemini API, Google may use submitted data to improve its products, according to its terms. Tell your users not to send passports, bank documents or other sensitive data.
- The prompts tell the model to translate and ignore instructions found inside texts, photos or audio (see **Prompt engineering** above). This reduces prompt-injection risk but does not remove it completely; no prompt-based defence is guaranteed against every possible phrasing.

## Troubleshooting

| Problem | Cause and fix |
|---|---|
| `ModuleNotFoundError: No module named 'aiogram'` | The virtual environment is not active or dependencies are not installed. Activate `venv` and run `pip install -r requirements.txt`. |
| `BOT_TOKEN not found` | `.env` is missing, misnamed (for example `.env.txt`) or not next to `bot.py`. |
| `Conflict: terminated by other getUpdates request` | Two copies of the bot are running with one token. Stop one of them. |
| `SyntaxError` right after editing/transferring the file | The file was corrupted during a manual edit or an interrupted transfer (quotes can get mangled by some editors). Run `python -m py_compile bot.py` locally before uploading; if it fails, replace the file completely instead of patching it. |
| `Gemini failed: 404 NOT_FOUND` | The model name is outdated or retired. Open AI Studio, pick a current model, update `GEMINI_MODEL`. |
| `Gemini failed: 503 UNAVAILABLE` (one or two) | Temporary overload on Google's side; the bot already retries once automatically. |
| `Gemini failed: 503 UNAVAILABLE` (persistent, many requests in a row) | A longer overload window on the free tier. Check [Google's status page](https://aistudio.google.com) or wait; this is not fixable from the bot's code. |
| `Google failed: TooManyRequests` | The unofficial Google Translate endpoint blocked the requests. The bot falls back to MyMemory automatically. Try `pip install -U deep-translator`. |
| Photo or audio translation says it is unavailable | `GEMINI_API_KEY` is not set or is not loaded from `.env`. |
| A new language button does not appear | The bot is still running an old version of `bot.py`. Restart it. |

## Known limitations

- Free quotas: Gemini has small daily and per-minute limits, and free-tier requests can be deprioritized ("sheddable") under load, which causes occasional `503` errors. Google Translate through `deep-translator` is an unofficial method and can be blocked at any time. MyMemory accepts only short texts.
- Photo and audio translation work only through Gemini. If Gemini is unavailable, there is no fallback for them.
- Handwriting, blurry photos, background noise and strong accents reduce quality.
- Language detection can be wrong for very short phrases and closely related languages.
- The daily counter and language choice live in one SQLite file. Deleting `bot.db` resets all users to English.

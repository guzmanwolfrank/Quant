# BollingerReversalFX

A Bollinger Band mean-reversion trading bot for an **OANDA practice (demo) account**. On every 5-minute candle close it scans every forex pair your account can trade, enters when price snaps back inside the bands in a ranging market, manages the stop in three stages, and writes a self-refreshing QuantStats-style dashboard.

> ⚠️ **Practice accounts only.** The bot is hard-wired to OANDA's practice API host (`api-fxpractice.oanda.com`). It will not trade a live account. Nothing here is financial advice.

---

## Table of contents

- [How the strategy works](#how-the-strategy-works)
- [Requirements](#requirements)
- [Installation](#installation)
- [Connecting your OANDA account](#connecting-your-oanda-account)
  - [1. Create a practice account](#1-create-a-practice-account)
  - [2. Generate an API token](#2-generate-an-api-token)
  - [3. Find your account ID](#3-find-your-account-id)
  - [4. Create `oanda_config.txt`](#4-create-oanda_configtxt)
  - [5. (Optional) Set up email alerts](#5-optional-set-up-email-alerts)
  - [6. Test the connection](#6-test-the-connection)
- [Running the bot](#running-the-bot)
- [Commands](#commands)
- [Output files](#output-files)
- [Configuration reference](#configuration-reference)
- [Troubleshooting](#troubleshooting)
- [Keeping your credentials safe](#keeping-your-credentials-safe)

---

## How the strategy works

### Entry (M5, closed candles only)

| Step | Rule |
|---|---|
| **Range filter** | ADX(14) must be **below 25**. Fading moves in a trend is how mean-reversion bots blow up, so this filter is never skipped. |
| **Long** | Signal candle closes **below** the lower Bollinger Band (20, 2.0) with RSI(14) **< 30**, then the next candle closes back **inside** the band and still below the middle band. |
| **Short** | Mirror image: close **above** the upper band with RSI **> 70**, then the next candle closes back inside and above the middle band. |
| **Execution** | Market order only if the spread is ≤ 3 pips. With several signals at once, the calmest market (lowest ADX) goes first. Max 5 open trades. Pairs that already have a trade on the account are skipped (US accounts are FIFO / no hedging). |

### Stop loss in three stages

1. **Initial stop** – signal candle's extreme + 2 pips, capped at the smaller of 60 pips or 3 × ATR(14), and never tighter than 10 pips.
2. **Profit lock** – at **+20 pips**, the stop moves to **entry + 5 pips**.
3. **Trailing stop** – a **25-pip** OANDA server-side trailing stop is added at the same time. It keeps working even if the program stops, and takes over from the +5 lock once the trade has been +30 pips.

**Take profit:** 50 pips, held server-side (or the middle band if `EXIT_AT_MIDDLE_BAND = True`).

### Position sizing

Each order is sized so the loss at the initial stop is **at most $2.00**, using OANDA's own currency conversion rates. The stop is attached as a *distance* from the actual fill, so slippage can't increase the risk. A price gap through the stop (e.g. over a weekend) can still lose more.

### Budget and shutdown

| Setting | Value |
|---|---|
| Account size (dashboard baseline) | $1,000 |
| Strategy budget | $100 – shuts down if lost |
| Profit target | +$10 – closes open trades, emails you, exits |

After a shutdown the bot refuses to trade again until you run it with `--reset`.

---

## Requirements

- **Python 3.9+**
- The [`requests`](https://pypi.org/project/requests/) package
- An **OANDA practice (fxTrade Practice) account** with a v20 API token
- *(Optional)* An email account with SMTP access for alerts (Gmail works with an App Password)

---

## Installation

```bash
git clone https://github.com/<your-username>/BollingerReversalFX.git
cd BollingerReversalFX
python -m pip install requests
```

---

## Connecting your OANDA account

The bot reads its credentials from a plain-text file called **`oanda_config.txt`**. You need two things from OANDA: an **API token** and your **v20 account ID**.

### 1. Create a practice account

If you don't have one, sign up for a free **fxTrade Practice** (demo) account on [oanda.com](https://www.oanda.com). Make sure you're working in the *practice* environment, not live – a live token will not work against the practice API.

### 2. Generate an API token

1. Log in to your OANDA **practice** account.
2. Open your account / profile settings and choose **Manage API Access**.
3. Click **Generate** (or **Regenerate**) to create a personal access token.
4. Copy the token right away and keep it private – anyone with it can place trades on your account.

> Regenerating a token invalidates the old one. If you regenerate, update `oanda_config.txt`.

### 3. Find your account ID

Your v20 account ID is shown in the OANDA account dashboard / account selector. It looks like this:

```
101-001-12345678-001
```

The bot checks the format (`NNN-NNN-NNNNNNN-NNN`) and will refuse to start if it doesn't match.

### 4. Create `oanda_config.txt`

Create a file named `oanda_config.txt` **in the same folder as `BollingerReversalFX.py`** and fill it in:

```ini
# ---- OANDA (required) ----
OANDA_TOKEN=paste-your-practice-api-token-here
OANDA_ACCOUNT_ID=101-001-12345678-001
OANDA_ENV=practice

# ---- Email alerts (optional) ----
EMAIL_TO=you@example.com
SMTP_USER=your.sender@gmail.com
SMTP_PASSWORD=your-16-char-app-password
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
```

Format rules:

- One `KEY=VALUE` per line. Spaces around `=` are fine.
- Anything after `#` is a comment.
- Quotes around values are optional (`"abc"` and `abc` are the same).
- The token must be your real token – if the value still contains the word `paste`, the bot treats it as unfilled and stops.

**Where the bot looks for the file** (first one found wins):

1. The path in the `BRFX_CONFIG` environment variable
2. `oanda_config.txt` next to the script
3. `oanda_config.txt` inside the output folder

**Environment variables override the file.** You can skip the file entirely and set `OANDA_TOKEN`, `OANDA_ACCOUNT_ID`, etc. as environment variables instead – handy for servers or CI.

### 5. (Optional) Set up email alerts

The bot emails you when it starts, every hour with the dashboard attached, and when it stops (target hit, budget lost, Ctrl+C, crash).

**Using Gmail:**

1. Turn on 2-Step Verification for the Google account you'll send from.
2. Create an **App Password** (Google Account → Security → App passwords).
3. Put the sending address in `SMTP_USER` and the app password in `SMTP_PASSWORD`. Spaces in the app password are stripped automatically.
4. Leave `SMTP_HOST=smtp.gmail.com` and `SMTP_PORT=465`.

**Other providers:** set `SMTP_HOST` and `SMTP_PORT`. Port `465` uses SSL; any other port (e.g. `587`) uses STARTTLS.

If `SMTP_USER` or `SMTP_PASSWORD` is missing, the bot still trades – it just logs that the email wasn't sent.

> Set `EMAIL_TO` to your own address. If you leave it out, alerts go to the default address hard-coded in the script (`DEFAULT_EMAIL_TO`), so change that too if you fork the repo.

### 6. Test the connection

```bash
# Check email settings
python BollingerReversalFX.py --test-email

# Preview the dashboard layout with sample data (no OANDA login needed)
python BollingerReversalFX.py --demo
```

When you start the bot for real, it logs in to OANDA before doing anything else. If the token or account ID is wrong, it stops with:

```
ERROR: could not log in to your OANDA practice account: ...
```

---

## Running the bot

### Set the output folder

By default, logs and the dashboard are written to a Windows path defined in the script:

```python
OUTPUT_DIR = Path(os.getenv("BRFX_OUTPUT_DIR", r"C:\Users\wolfr\Desktop\Trading\LogsandDash\BollingerReversalFX"))
```

Point it somewhere on your machine with the `BRFX_OUTPUT_DIR` environment variable, or edit the default in the script.

### Start it

```bash
python BollingerReversalFX.py
```

Leave the window open. Press **Ctrl+C** to stop – open trades keep their stop, trailing stop and take profit on OANDA's servers, but the +5-pip profit lock only happens while the program is running.

### Windows: double-click launcher

Create `run_BollingerReversalFX.bat` next to the script:

```bat
@echo off
cd /d "%~dp0"
python BollingerReversalFX.py
```

Optionally add `set BRFX_OUTPUT_DIR=C:\path\to\output` above the `python` line.

### Resuming

The bot keeps its memory in `state.json`. If you stop and restart it, it resumes the same run and re-adopts any of its trades still open on OANDA (they're tagged `BollingerReversalFX`).

---

## Commands

| Command | What it does |
|---|---|
| `python BollingerReversalFX.py` | Run the strategy |
| `python BollingerReversalFX.py --test-email` | Send a test email alert |
| `python BollingerReversalFX.py --dashboard` | Rebuild `Dashboard.html` without trading |
| `python BollingerReversalFX.py --demo` | Write `Dashboard_demo.html` with **sample** data |
| `python BollingerReversalFX.py --reset` | Archive the current run (`state_YYYYMMDD_HHMMSS.json`) and start fresh |

---

## Output files

All written to the output folder:

| File | Contents |
|---|---|
| `Log.txt` | Every trade opened, stop moved and closed, skipped signals, start/stop events |
| `Dashboard.html` | Intraday tearsheet – equity curve, underwater plot, per-trade P&L, R distribution, hourly returns, weekday × hour heatmap, Sharpe/Sortino/SQN and more. Refreshes every 30 s. |
| `state.json` | The bot's memory (trades, start time). **Don't edit it while the bot is running.** |

---

## Configuration reference

### `oanda_config.txt` / environment variables

| Key | Required | Default | Description |
|---|---|---|---|
| `OANDA_TOKEN` | ✅ | – | Practice v20 API token |
| `OANDA_ACCOUNT_ID` | ✅ | – | Practice account ID, e.g. `101-001-12345678-001` |
| `OANDA_ENV` | | `practice` | Informational only – the bot always uses the practice host and warns if this says otherwise |
| `EMAIL_TO` | | `DEFAULT_EMAIL_TO` in script | Where alerts are sent |
| `SMTP_USER` | | – | Sending email address / SMTP login |
| `SMTP_PASSWORD` | | – | SMTP password or app password |
| `SMTP_HOST` | | `smtp.gmail.com` | SMTP server |
| `SMTP_PORT` | | `465` | `465` = SSL, anything else = STARTTLS |

### Environment-only variables

| Variable | Description |
|---|---|
| `BRFX_CONFIG` | Full path to a config file elsewhere |
| `BRFX_OUTPUT_DIR` | Where logs, dashboard and state are written |
| `FF_NO_PAUSE` | Set to anything to skip the "Press Enter to close" prompt |

### Strategy settings (top of the script)

| Setting | Default | Meaning |
|---|---|---|
| `ACCOUNT_SIZE` | `1000.0` | Baseline for all percentages |
| `STRATEGY_BUDGET` | `100.0` | Shut down after losing this much |
| `PROFIT_TARGET_USD` | `10.0` | Shut down after making this much |
| `MAX_RISK_USD` | `2.00` | Max loss per trade at the initial stop |
| `MAX_OPEN_TRADES` | `5` | Concurrent trade limit |
| `BB_PERIOD` / `BB_STD` | `20` / `2.0` | Bollinger Bands |
| `RSI_PERIOD` / `RSI_OVERSOLD` / `RSI_OVERBOUGHT` | `14` / `30` / `70` | RSI filter |
| `ADX_PERIOD` / `ADX_MAX` | `14` / `25` | Range filter (period also used for ATR) |
| `STOP_BUFFER_PIPS` | `2` | Beyond the signal candle's extreme |
| `MAX_INITIAL_STOP_PIPS` / `ATR_STOP_MULT` / `MIN_STOP_PIPS` | `60` / `3.0` / `10` | Initial stop limits |
| `TAKE_PROFIT_PIPS` | `50` | Server-side take profit |
| `LOCK_TRIGGER_PIPS` / `LOCK_PROFIT_PIPS` | `20` / `5` | Profit-lock trigger and level |
| `TRAIL_PIPS` | `25` | Trailing stop distance |
| `EXIT_AT_MIDDLE_BAND` | `False` | Take profit at the 20 SMA instead |
| `MAX_SPREAD_PIPS` | `3.0` | Skip signals with a wider spread |
| `STATUS_EMAIL_MINUTES` | `60` | Status email interval (`0` = off) |

---

## Troubleshooting

| Message | Fix |
|---|---|
| `the 'requests' package is not installed` | `python -m pip install requests` |
| `OANDA_TOKEN and OANDA_ACCOUNT_ID are not set` | Create/fill `oanda_config.txt` next to the script, and make sure the token doesn't still say `paste...` |
| `OANDA_ACCOUNT_ID '...' doesn't look like an OANDA v20 account ID` | Use the full ID with dashes, e.g. `101-001-12345678-001` |
| `could not log in to your OANDA practice account ... 401` | Token is wrong, expired, regenerated, or from a **live** account. Generate a new token from the **practice** site. |
| `... 403` / `... 404` on login | The account ID doesn't belong to that token |
| `Email NOT sent ... SMTP_USER / SMTP_PASSWORD missing` | Add both to the config, or ignore if you don't want email |
| `Email FAILED ... Username and Password not accepted` | For Gmail, use an App Password, not your normal password |
| `already shut down on ...` | The run hit its target or budget. Run `--reset`, then start again. |
| `practice NAV is X, not 1000` (warning) | Harmless – the bot tracks its own P&L. You can reset the practice balance in OANDA if you like. |

---

## Keeping your credentials safe

Your API token can place trades on your account, and the SMTP password can send email as you. **Never commit `oanda_config.txt`.** Add a `.gitignore` to the repo:

```gitignore
oanda_config.txt
state*.json
Log.txt
Dashboard*.html
*.tmp
__pycache__/
```

Consider committing an `oanda_config.example.txt` with placeholder values instead, so others can copy it.

---

## Disclaimer

This software is for educational use on a **practice account only**. Trading foreign exchange carries a high level of risk. Past or simulated performance does not guarantee future results. Use at your own risk.


# Tesla Invoice Downloader

## Overview

**Tesla Invoice Downloader** is a Python script that retrieves Tesla charging invoices via the Fleet API. It authenticates via OAuth, fetches charging history, and downloads invoices as PDF files. The script can run as a one-time operation or as a daemon checking for new invoices every hour.

## License

**Copyright © 2025 Alastair D'Silva**

This project is licensed under the **GNU General Public License v3 (GPLv3)**. See the [LICENSE](LICENSE) file for details.

## Features

- OAuth authentication with Tesla Developer API
- Fetches charging history and invoices
- Supports filtering by VIN
- Only retrieves invoices since the last saved charge session
- Saves metadata alongside invoices in JSON format
- Daemon mode: checks for new invoices every hour
- Optional Moneybird upload: pushes each invoice into Moneybird's Documenten inbox and lets Moneybird's OCR fill in the details, so the bookkeeper can convert it to a purchase invoice with the correct regional Tesla supplier
- Configurable output directory and logging

## Installation

### Prerequisites

- Python 3.7+
- A Tesla Developer application (created in [Running daily — Step 0](#step-0--create-a-tesla-developer-app))
- Optional: a Moneybird personal API token if you want invoices uploaded automatically

### Install in a virtual environment

Clone the repo, create a venv inside it, and install the single dependency:

```sh
git clone https://github.com/Thomvh/tesla_invoice_downloader_moneybird.git
cd tesla_invoice_downloader_moneybird
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install requests
```

From here on, every command in this README assumes the venv is active (`source .venv/bin/activate`). To leave the venv: `deactivate`.

## Usage

Run the script with the following options:

```sh
python tesla_invoice_downloader.py [OPTIONS]
```

### Command-line Arguments

| Option                       | Description |
|------------------------------|-------------|
| `--vin VIN`                  | Restrict invoices to a specific VIN |
| `--output-dir DIR`           | Directory to save invoices. Omit to enable streaming mode (no PDFs on disk — requires Moneybird credentials). |
| `--log-file FILE`            | File to save logs (optional) |
| `--daemon`                   | Run as a background process, checking for new invoices every hour |
| `--on-or-after YYYYMMDD`     | Only download invoices on or after the given date |
| `--moneybird-token TOKEN`    | Moneybird personal API token. Enables uploading each downloaded invoice into the Moneybird Documenten inbox |
| `--moneybird-admin-id ID`    | Moneybird administration ID to upload into |
| `--moneybird-list-config`    | List administrations available to `--moneybird-token`, then exit |
| `--moneybird-setup`          | Interactively save Moneybird token + administration into the config file, then exit |
| `--debug`                    | Enable debug logging |

### Example Usage

1. **One-time download**
   ```sh
   python tesla_invoice_downloader.py --output-dir ~/invoices
   ```

2. **Filter invoices for a specific VIN**
   ```sh
   python tesla_invoice_downloader.py --vin 5YJSA1E26JF278XXX
   ```

3. **Run in daemon mode**
   ```sh
   python tesla_invoice_downloader.py --daemon --output-dir ~/invoices --log-file ~/logs/invoice.log
   ```

## Running daily

Once installed, the recommended deployment is a daily cron job that downloads any new invoices (and optionally uploads them to Moneybird).

### Step 0 — Create a Tesla Developer app

Skip if you already have a Client ID + Client Secret. Otherwise:

1. Go to <https://developer.tesla.com/> and click **Get Started**.
2. Create an application with:
   - **Application Name:** anything that does not contain "Tesla"
   - **OAuth Grant Type:** *Authorization Code and Machine-to-Machine*
   - **Allowed Origin URL:** `http://localhost:8585`
   - **Allowed Redirect URI:** `http://localhost:8585/callback`
   - **API & Scopes:** tick *Vehicle Charging Management*
3. Note the **Client ID** and **Client Secret** Tesla shows you — you will enter them in Step 1.

### Step 1 — First-time Tesla authentication

Run the script once interactively so it can open a browser for Tesla's OAuth flow. From here on it caches and refreshes the tokens automatically.

```sh
source .venv/bin/activate
python tesla_invoice_downloader.py --output-dir ~/tesla-invoices
```

The script will:

1. Prompt for your Tesla **Client ID**, **Client Secret**, and region (NA/EU).
2. Open Tesla's authorisation page in your browser.
3. Catch the redirect on `http://localhost:8585/callback` and exchange the code for tokens.
4. Download any existing invoices into `~/tesla-invoices/`.

Tokens, region, and credentials are written to `~/.tesla_invoice_downloader.json` (chmod 0600). Subsequent runs are non-interactive.

### Step 2 — Optional: configure Moneybird

If you want each invoice uploaded automatically into the Moneybird Documenten inbox, generate a personal API token at <https://moneybird.com/user/applications/new> (tick the `documents` scope), then run:

```sh
python tesla_invoice_downloader.py --moneybird-setup
```

This prints the administrations available to your token, lets you pick one interactively, and saves the token + administration id into `~/.tesla_invoice_downloader.json`. Cron lines can now omit `--moneybird-token` / `--moneybird-admin-id` for that machine, or pass them explicitly when you want a single machine to serve multiple administrations.

### Step 3 — Schedule daily runs

Two equivalent options. **systemd timer** (Option A) is recommended on any modern Linux distribution: journal-based logging, native failure handling, easy on/off via `systemctl`. **cron** (Option B) is simpler and works everywhere; use it on macOS, BSDs, or if you just want a one-liner.

Both call the script in one-shot mode. Don't combine either with `--daemon` — the daemon's internal hourly loop would duplicate the timer's scheduling.

#### Option A — systemd timer (recommended for Linux servers)

This shape assumes a dedicated unprivileged service user `tesla` with home at `/var/lib/tesla` (where `~/.tesla_invoice_downloader.json` will live) and the repo cloned into `/opt/tesla-moneybird`. Adjust the paths if your conventions differ.

**Install code + venv as the service user:**

```sh
sudo useradd --system --home /var/lib/tesla --create-home --shell /usr/sbin/nologin tesla
sudo git clone https://github.com/Thomvh/tesla_invoice_downloader_moneybird.git /opt/tesla-moneybird
sudo chown -R tesla:tesla /opt/tesla-moneybird
sudo -u tesla bash -c 'cd /opt/tesla-moneybird && python3 -m venv .venv && .venv/bin/pip install --upgrade pip && .venv/bin/pip install requests'
```

**Seed the config interactively.** The Tesla OAuth flow needs a browser, which a headless server doesn't have. Easiest path: complete the OAuth flow on your workstation (Steps 1 + 2 above), then copy the resulting config file to the server:

```sh
scp ~/.tesla_invoice_downloader.json YOUR_SERVER:/tmp/tesla-config.json
ssh YOUR_SERVER 'sudo install -o tesla -g tesla -m 600 /tmp/tesla-config.json /var/lib/tesla/.tesla_invoice_downloader.json && rm /tmp/tesla-config.json'
```

(Alternative: SSH-forward port 8585 from the server to your laptop with `ssh -L 8585:localhost:8585`, then run the OAuth flow on the server through the tunnel.)

**Create the service unit** at `/etc/systemd/system/tesla-moneybird.service`:

```ini
[Unit]
Description=Tesla invoice downloader to Moneybird Documenten
Documentation=https://github.com/Thomvh/tesla_invoice_downloader_moneybird
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=tesla
Group=tesla
WorkingDirectory=/opt/tesla-moneybird
ExecStart=/opt/tesla-moneybird/.venv/bin/python /opt/tesla-moneybird/tesla_invoice_downloader.py

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/tesla
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
LockPersonality=true
```

Notes on this unit:
- `Type=oneshot` is correct because the script exits when done. systemd treats the run as "active" only while it's executing.
- No `--log-file` is passed — the script logs to stderr by default, which systemd captures into the journal. View with `journalctl -u tesla-moneybird.service`.
- `--output-dir` is intentionally omitted, putting the script in streaming mode (PDFs go straight to Moneybird, nothing written to disk). If you want a local archive, add `--output-dir /var/lib/tesla/invoices` to the `ExecStart` line (the `tesla` user already owns `/var/lib/tesla`, so no `ReadWritePaths` change is needed for a subdirectory of it).
- `ProtectSystem=strict` makes the whole filesystem read-only; `ReadWritePaths=/var/lib/tesla` re-opens just the service user's home so the script can write its config + state file there. `ProtectHome=true` keeps the script away from other users' home directories under `/home` and `/root`.

**Create the timer unit** at `/etc/systemd/system/tesla-moneybird.timer`:

```ini
[Unit]
Description=Daily Tesla -> Moneybird run
Requires=tesla-moneybird.service

[Timer]
OnCalendar=*-*-* 08:00:00
RandomizedDelaySec=15m
Persistent=true

[Install]
WantedBy=timers.target
```

`Persistent=true` makes systemd fire a missed run at next boot if the server was off at 08:00. `RandomizedDelaySec=15m` jitters the start so the request hits Moneybird at some point in the 08:00–08:15 window — good etiquette if many people use the same script.

**Enable and start:**

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now tesla-moneybird.timer
systemctl list-timers tesla-moneybird.timer    # confirm next firing
```

**Multiple Moneybird administrations:** copy the `.service` + `.timer` pair to `tesla-moneybird-b.service` / `tesla-moneybird-b.timer`, add `--moneybird-token BBB --moneybird-admin-id 222` to the `ExecStart` line of the new service, stagger the `OnCalendar=` time, enable. They share the same code and the same `tesla` user, but the local upload-state map in the config is keyed per admin id so they won't clash.

#### Option B — cron (portable, simpler)

Find the absolute paths to the venv's Python interpreter and the script:

```sh
which python                       # absolute path to the venv's python
realpath tesla_invoice_downloader.py   # absolute path to the script
```

Open your crontab:

```sh
crontab -e
```

Add one line per administration. Pick one of the two shapes:

**Disk archive + Moneybird** — keeps PDFs on disk alongside the Moneybird upload:

```cron
0 8 * * * /Users/you/path/to/repo/.venv/bin/python /Users/you/path/to/repo/tesla_invoice_downloader.py --output-dir /Users/you/tesla-invoices --log-file /Users/you/tesla-invoices/cron.log
```

**Moneybird only** — no local files, requires Moneybird credentials in the config (Step 2) or on the cron line:

```cron
0 8 * * * /Users/you/path/to/repo/.venv/bin/python /Users/you/path/to/repo/tesla_invoice_downloader.py --log-file /Users/you/tesla-cron.log
```

Multiple Moneybird administrations — repeat the line, override credentials per cron entry:

```cron
0 8 * * * /Users/you/path/to/repo/.venv/bin/python /Users/you/path/to/repo/tesla_invoice_downloader.py --log-file /Users/you/tesla-cron-a.log --moneybird-token AAA --moneybird-admin-id 111
5 8 * * * /Users/you/path/to/repo/.venv/bin/python /Users/you/path/to/repo/tesla_invoice_downloader.py --log-file /Users/you/tesla-cron-b.log --moneybird-token BBB --moneybird-admin-id 222
```

`0 8 * * *` runs at 08:00 every day. Adjust the minute/hour to taste.

### Step 4 — Verify the scheduled job

For systemd:

```sh
systemctl list-timers tesla-moneybird.timer    # next firing + last result
journalctl -u tesla-moneybird.service -n 100   # recent run logs
sudo systemctl start tesla-moneybird.service   # force a run now for testing
```

For cron:

```sh
tail -n 50 ~/tesla-cron.log
```

You should see Tesla auth + `Total charging sessions retrieved: N`, and (if Moneybird is configured) `Uploaded session ... to Moneybird` lines for any new sessions. The first run after activation will quietly do nothing if you already downloaded the backlog manually in Step 1.

### macOS notes

On modern macOS, the user-level cron daemon needs Full Disk Access to write into protected directories (Desktop, Documents, Downloads, iCloud Drive). If your `--output-dir` or `--log-file` falls under one of those, grant `/usr/sbin/cron` Full Disk Access in **System Settings → Privacy & Security → Full Disk Access**. Writing into a plain folder in your home directory (e.g. `~/tesla-invoices/`) does not need it.

## Moneybird upload (optional)

When a Moneybird token and administration id are provided (either via CLI flags or via the config file), every downloaded invoice is also uploaded to Moneybird as a typeless document (Documenten inbox). The script uploads the PDF as an attachment with the Tesla `sessionId` as the document `reference`; Moneybird's OCR fills in supplier, amounts, currency and tax, and the bookkeeper converts it to a purchase invoice in the Moneybird UI with the correct contact. The script deliberately does not pick a contact or ledger account, because Tesla bills from different legal entities per region. (We use `typeless_documents` rather than `general_documents` because the latter auto-marks the document as paid.)

### Streaming mode

Omitting `--output-dir` switches the script into streaming mode: Tesla PDFs are fetched into memory and pushed straight to Moneybird, never touching the local filesystem. The `.json` metadata sidecar is also skipped. Streaming mode requires Moneybird credentials; running the script with no `--output-dir` and no Moneybird credentials exits with a clear error. Dedup still works (local `moneybird.uploaded.<admin_id>` map + Moneybird-side `reference` lookup), so re-runs are safe.

### Idempotency

The script tracks uploaded sessions per administration under `moneybird.uploaded.<admin_id>` in the config file, and additionally queries Moneybird for an existing document with the same `reference` before creating a new one. Re-running the script (or wiping local state) will not produce duplicate documents.

## File Naming Convention

Invoices are saved with the following filename format:

```
YYYYMMDD.Tesla.Charging - <location> - <charging_usage>kWh.<currencySymbol><total_due>.pdf
```

Where:
- `YYYYMMDD`: Charge start date
- `<location>`: Site location
- `<charging_usage>`: Total charging kWh
- `<currencySymbol>`: Currency symbol (e.g., `$` for AUD/CAD, `€` for EUR)
- `<total_due>`: Total charge cost

## Configuration File

The script stores authentication tokens and charging history in:

```
~/.tesla_invoice_downloader.json
```

This file is updated automatically and should be kept secure.

## Contributing

Pull requests and contributions are welcome! Please ensure your code adheres to the project structure and style.

## Issues & Support

If you encounter any issues, feel free to open a GitHub issue or reach out.

## Acknowledgments

- [Tesla Developer API documentation](https://developer.tesla.com/docs/)
- Open-source community for [Python](https://www.python.org/)

## Disclaimer

This project is not affiliated with or endorsed by Tesla, Inc.



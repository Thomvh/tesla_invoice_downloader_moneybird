# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file Python script (`tesla_invoice_downloader.py`) that authenticates against the Tesla Fleet API via OAuth and downloads charging invoices as PDFs plus a sibling `.json` metadata file. When Moneybird credentials are configured, the same run also uploads each invoice as a typeless document into the Moneybird Documenten inbox. Licensed GPLv3.

## Run / dependencies

Only runtime dependency is `requests` (no `requirements.txt`, no test suite, no lint config in the repo):

```sh
pip install requests
python tesla_invoice_downloader.py [--vin VIN] [--output-dir DIR] [--log-file FILE] \
    [--daemon] [--force-auth] [--on-or-after YYYYMMDD | --since-days N] [--debug] \
    [--moneybird-token TOKEN --moneybird-admin-id ID] \
    [--moneybird-list-config | --moneybird-setup]
```

`--daemon` uses the Unix double-fork pattern, so it will not work on Windows. `--on-or-after` is validated as `YYYYMMDD` at arg-parse time. `--since-days N` is a relative alternative that resolves to `--on-or-after` at run time (and is re-resolved at the top of every `--daemon` cycle via `applySinceDays`, so a long-running daemon's window slides forward instead of freezing); the two flags are in an argparse mutually exclusive group.

`--output-dir` is **optional**. If set, behaviour is "download to disk, then upload to Moneybird if configured". If omitted, the script runs in **streaming mode**: it requires Moneybird credentials and pushes PDFs straight from Tesla to Moneybird without ever writing to disk (the `.json` sidecar is also skipped). At least one of `--output-dir` or Moneybird credentials must be present; `main()` exits non-zero with a clear error otherwise.

The Moneybird flags resolve from CLI first, then from `config["moneybird"]["defaults"]`. `--moneybird-list-config` and `--moneybird-setup` short-circuit `main()` and exit before any Tesla auth happens.

## Architecture

The script is one module organised around a single `TeslaInvoiceDownloader` class plus free helpers. Key flow:

1. `loadConfig` / `saveConfig` persist everything (credentials, tokens, charging history) to `~/.tesla_invoice_downloader.json` (chmod 0600, timestamped backups on every save).
2. `authenticate()` is the token state machine: prefer cached `access_token` if `expires_at` is > 5 min in the future, otherwise try `refreshAccessToken`, otherwise fall back to `performOAuthFlow`.
3. `performOAuthFlow` binds a local TCP socket on `localhost:8585`, opens the Tesla auth page in a browser, parses the `code` and `state` out of the single inbound HTTP request, and exchanges the code at `fleet-auth.prd.vn.cloud.tesla.com`. State is verified to prevent CSRF.
4. `getBaseUrlForRegion` switches between the EU and NA Fleet API hosts based on `config["region"]`. The chosen host is also passed as the OAuth `audience`.
5. `fetchChargingHistory` paginates `/api/1/dx/charging/history` at pageSize 50, and when a `--vin` is given uses the most recent stored `chargeStartDateTime` as `startTime` so only new sessions are pulled. Results are merged by `sessionId` into `config["charging_history"][vin]`.
6. `downloadInvoices` iterates records, filters by `--vin` and `--on-or-after`, computes the filename via `getInvoiceFilename`, skips files that already exist, calls `fetchInvoicePdfBytes` (which hits `/api/1/dx/charging/invoice/{contentId}`), writes the PDF + a sibling `.json` with the raw record. `fetchInvoicePdfBytes` is the shared primitive — the streaming path calls it directly instead of going through `downloadInvoices`.
7. `makeRequest` is the single HTTP wrapper with exponential backoff on HTTP 429. It supports `GET` (params), `POST` (form `data`, JSON `json_body`, or multipart `files`), so both the Tesla and Moneybird paths share the same retry logic.
8. `MoneybirdUploader` uploads to the **`documents/typeless_documents`** endpoint, not `purchase_invoices` (which requires `contact_id` + line items the script can't supply because Tesla bills from different legal entities per region) and not `general_documents` (which auto-marks the document as paid). The typeless document is created with just `reference = sessionId`, then the PDF is attached; Moneybird's OCR fills in supplier, amounts, currency and tax, and the bookkeeper promotes it to a purchase invoice in the UI. Two entry points: `uploadDownloaded` (called by `runMoneybirdUpload`) reads PDFs from disk after `downloadInvoices` wrote them; `streamRecords` (called by `runMoneybirdStream`) calls `fetchInvoicePdfBytes` per record and streams bytes via `attachPdf(document_id, filename, bytes)` — `attachPdf` accepts either a path or `bytes`. Dedup: local state at `config["moneybird"]["uploaded"][admin_id][sessionId]` first, then a Moneybird-side `filter=reference:<sessionId>` lookup as fallback. In streaming mode the dedup checks happen *before* the Tesla GET, so duplicates cost zero Tesla bandwidth.

### Things that will trip you up

- `args` is a **module-level global** set inside `main()` and read by `TeslaInvoiceDownloader.authenticate` (it checks `args.force_auth`). Adding new CLI flags that the class needs to see follows this same pattern.
- `saveConfig` renames the existing config to a timestamped backup on every write. Repeated runs will pile up `~/.tesla_invoice_downloader.json.YYYYMMDD.HHMMSS` files.
- `redactSensitive` is what keeps tokens out of `--debug` logs. Any new code path that logs a request/response body should route through it.
- Filename construction in `getInvoiceFilename` sums `usageBase` for fees with `feeType == "CHARGING"` and sums `totalDue` over **all** fees, then maps `currencyCode` via the in-function `CURRENCY_SYMBOLS` table (unmapped currencies produce an empty symbol). `safeFilename` strips `\ / : * ? " < > |`.
- Charging history is keyed by VIN inside `config["charging_history"]`; records with no VIN are stored under `"Unknown"`.

## File naming convention (produced output)

```
YYYYMMDD.Tesla.Charging - <location> - <usage>kWh.<currencySymbol><totalDue>.pdf
```

Accompanied by a `.json` with the full charging-history record.

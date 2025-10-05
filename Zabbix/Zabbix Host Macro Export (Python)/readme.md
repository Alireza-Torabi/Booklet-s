# README — Zabbix Host Macro Export (Python)

Export a list of **all hosts** in Zabbix and the **host-level** value of a specific macro (default: `{$HOST.MACRO1}`) to **CSV** or **TSV**.

> Tested with **Zabbix 7.0.16**.
> This script reads **host-level** user macros only. If a host inherits a macro from a **template** or **global** macro, that inherited value will not appear here (see FAQ).

---

## What it does

* Calls `host.get` with `selectMacros` to fetch every host and its **host-level** user macros.
* Extracts the value for the macro you choose (default `{$HOST.MACRO1}`).
* Writes a sorted table to `hosts_macro.csv` (or `.tsv`).

Example CSV:

```csv
host,name,HOST.MACRO1
web01,Web Frontend #1,@owner_alice
db01,Primary DB,@owner_bob
api02,API Node 2,
```

---

## Requirements

* **Python 3.8+**
* **Zabbix API token** (Zabbix UI → your user menu → *Profile* → *API tokens* → *Create*).
* Python package: `requests`

Install dependency:

```bash
pip install requests
```

(Optionally use a virtual environment: `python3 -m venv .venv && source .venv/bin/activate` on Linux/macOS, or `.\.venv\Scripts\Activate.ps1` on Windows.)

---

## Files

* `zbx_export_macro.py` — the script
* *(optional)* `requirements.txt`

  ```
  requests
  ```

Install from `requirements.txt`:

```bash
pip install -r requirements.txt
```

---

## Usage

Basic CSV export:

```bash
python3 zbx_export_macro.py \
  --url https://your.zabbix.example \
  --token YOUR_ZABBIX_API_TOKEN \
  --macro '{$HOST.MACRO1}' \
  --out hosts_macro.csv
```

TSV (plain text):

```bash
python3 zbx_export_macro.py \
  --url https://your.zabbix.example \
  --token YOUR_ZABBIX_API_TOKEN \
  --macro '{$HOST.MACRO1}' \
  --format tsv \
  --out hosts_macro.tsv
```

Connect to Zabbix with a self-signed cert:

```bash
python3 zbx_export_macro.py ... --insecure
```

### All options

```
--url       Zabbix base URL (e.g., https://zbx.example.com) [required]
--token     Zabbix API token [required]
--macro     Macro name to export (default: {$HOST.MACRO1})
--out       Output file path (default: hosts_macro.csv)
--format    csv | tsv (default: csv)
--insecure  Disable TLS certificate verification
```

---

## Windows notes

On Windows PowerShell, use `python` instead of `python3`:

```powershell
python zbx_export_macro.py --url https://your.zabbix.example --token YOUR_TOKEN --out hosts_macro.csv
```

If you use a virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install requests
```

---

## Security

* Treat your API token like a password. Prefer environment variables or a secret manager if you script this in CI.
* Avoid `--insecure` unless necessary.

---

## Troubleshooting

* **`HTTPError: 401` or `Zabbix API error: Not authorized`**
  Token is invalid or expired; generate a new one and ensure your user has API access.

* **`SSLError: certificate verify failed`**
  Your Zabbix uses a self-signed certificate. Use `--insecure` or install the CA cert.

* **Empty value in the output**
  The host may not have a **host-level** value for that macro. It might be inherited from a template/global macro (see FAQ).

* **`Zabbix API error …`**
  The message usually includes the method and reason. Re-check `--url`, token, and the macro spelling.

---

## FAQ

**Q: Can this show inherited values (template/global)?**
A: This script intentionally exports **host-level** values only. If you need the **effective** value (host → templates → global), use an extended version that traverses template chains and applies macro precedence. If you want it, ask and we’ll add an “effective resolver” variant.

**Q: Can I export multiple macros at once?**
A: Yes—run the script multiple times with different `--macro` names, or extend it to collect several macros into one CSV.

**Q: Does the script change anything in Zabbix?**
A: No. It performs read-only API calls.

---

## License

Use freely within your organization. Add a LICENSE file if you plan to redistribute.

---

## Appendix — Script

Place this file next to the README as `zbx_export_macro.py`.

```python
#!/usr/bin/env python3
import argparse, csv, sys, requests
from urllib.parse import urljoin

def zbx_call(base_url, token, method, params, verify=True, timeout=30):
    api = urljoin(base_url if base_url.endswith('/') else base_url + '/', 'api_jsonrpc.php')
    payload = {
        "jsonrpc": "2.0",
        "method": method,
        "params": params,
        "auth": token,
        "id": 1
    }
    r = requests.post(api, json=payload, headers={"Content-Type":"application/json"}, verify=verify, timeout=timeout)
    r.raise_for_status()
    data = r.json()
    if "error" in data:
        raise RuntimeError(f"Zabbix API error {data['error'].get('code')}: {data['error'].get('message')} - {data['error'].get('data')}")
    return data["result"]

def fetch_hosts_with_macro(base_url, token, macro_name, verify=True):
    result = zbx_call(base_url, token, "host.get", {
        "output": ["host", "name"],
        "selectMacros": ["macro", "value"]
    }, verify=verify)

    rows = []
    for h in result:
        macros = h.get("macros") or []
        val = next((m.get("value","") for m in macros if m.get("macro") == macro_name), "")
        rows.append({
            "host": h.get("host",""),
            "name": h.get("name",""),
            macro_name.strip("{}$"): val
        })
    rows.sort(key=lambda r: (r.get("name") or "").lower())
    return rows

def write_table(rows, path, fmt="csv"):
    headers = ["host", "name"] + [k for k in rows[0].keys() if k not in ("host", "name")] if rows else ["host","name","HOST.MACRO1"]
    with open(path, "w", newline="", encoding="utf-8") as f:
        if fmt == "tsv":
            writer = csv.DictWriter(f, fieldnames=headers, dialect="excel-tab", extrasaction="ignore")
        else:
            writer = csv.DictWriter(f, fieldnames=headers, extrasaction="ignore")
        writer.writeheader()
        writer.writerows(rows)

def main():
    ap = argparse.ArgumentParser(description="Export Zabbix host-level macro values to CSV/TSV")
    ap.add_argument("--url", required=True, help="Zabbix base URL, e.g. https://zbx.example.com")
    ap.add_argument("--token", required=True, help="Zabbix API token")
    ap.add_argument("--macro", default="{$HOST.MACRO1}", help="Macro name to export (default: {$HOST.MACRO1})")
    ap.add_argument("--out", default="hosts_macro.csv", help="Output file path")
    ap.add_argument("--format", choices=["csv","tsv"], default="csv", help="Output format (default: csv)")
    ap.add_argument("--insecure", action="store_true", help="Disable TLS cert verification")
    args = ap.parse_args()

    try:
        rows = fetch_hosts_with_macro(args.url, args.token, args.macro, verify=not args.insecure)
        write_table(rows, args.out, fmt=args.format)
        print(f"Wrote {args.out} with {len(rows)} rows.")
    except Exception as e:
        print(f"ERROR: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

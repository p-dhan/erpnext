---
name: frappe-cli
description: Use frappe-cli to query or update live data on a connected Frappe/ERPNext site (e.g. erp.snekaengineering.com) — customers, sales invoices, accounts receivable, any DocType. Trigger this whenever the user asks to look up, check, verify, list, or fix something in "the ERP", "the site", "ERPNext", "the live data", ERPNext DocTypes (Customer, Sales Invoice, Process Statement of Accounts, Auto Email Report, etc.), or asks you to create/update/delete a record on the connected instance — even if they don't say "frappe-cli" by name. Also use it to investigate why a report, statement, or scheduled email looks wrong by pulling the real records involved.
---

# frappe-cli

`frappe-cli` is a REST API v2 client for Frappe/ERPNext sites. It is not a
built-in tool — it's an external CLI program, invoked the same way as `git`
or `npm`: through the Bash tool, shelling out to the installed executable.

It gives you a general-purpose way to inspect and (carefully) modify data on
whatever Frappe/ERPNext site the user has authenticated it against — useful
whenever a task requires ground truth from the live system rather than
guessing from code or documentation alone (e.g. "why isn't this invoice
showing up", "what's this customer's territory", "check if that record was
created correctly").

## Finding and invoking the binary

It is usually not on `PATH`. Locate it once per session and call the full
path every time:

```sh
# Bash
"/c/Users/Praveen/AppData/Local/Programs/Python/Python314/Scripts/frappe-cli.exe" auth whoami
```

```powershell
# PowerShell
& "C:\Users\Praveen\AppData\Local\Programs\Python\Python314\Scripts\frappe-cli.exe" auth whoami
```

If that path doesn't exist (a different machine, reinstalled Python, etc.),
find it with `where.exe frappe-cli` / `Get-Command frappe-cli` or by checking
`pip show -f frappe-cli` for the install location, rather than guessing.

Always pass `--json` — it gives clean, parseable output instead of a
human-formatted table, and suppresses interactive prompts/color codes that
are wasted tokens in a transcript.

## Check the connection before relying on it

Don't assume a site is authenticated. Run `auth whoami` first if you haven't
confirmed the connection earlier in the session:

```sh
frappe-cli auth whoami --json
```

If it errors with "No site configured", **stop and tell the user** — do not
try to fix this yourself. Authentication is deliberately the user's job, not
yours:

- Never run `frappe-cli auth login` — it's interactive-only and needs
  credentials only the user has (an API key + secret generated from their own
  User → Settings → API Access).
- Never set, export, or otherwise write `FRAPPE_SITE`, `FRAPPE_API_KEY`, or
  `FRAPPE_API_SECRET`. If they're already set in the environment, that's the
  user's setup — read it, don't touch it.

## Orient yourself before guessing names

Frappe DocTypes and field names are often not what you'd expect from the UI
label. Before filtering on a field or assuming a DocType name, check:

```sh
frappe-cli doctype list --json                        # what DocTypes exist
frappe-cli doctype show "Sales Invoice" --json         # fields, types, links, required, child tables
```

This is cheap and saves a round trip of guessing wrong and getting a filter
error back.

## Reading data — do this freely

Read-only commands carry no risk and don't need permission to run:

```sh
frappe-cli doc list "Sales Invoice" -f status=Overdue -f 'grand_total>1000' \
  --fields name,customer,grand_total --json
frappe-cli doc list "Sales Invoice" --filters-json '[["status","in",["Paid","Overdue"]]]' --json
frappe-cli doc get "Sales Invoice" SINV-0001 --json     # full document, including child tables
frappe-cli report run "Accounts Receivable" -f company="Frappe" --json
frappe-cli api method/frappe.client.get_count -F doctype=User --json
```

Use `doc get` liberally when debugging something the user reports as "wrong"
— e.g. a report showing unexpected numbers, a scheduled email that didn't go
out, a record that seems missing. Pulling the actual document (and any
documents it's supposed to relate to, like the customer behind an invoice)
almost always resolves faster than reasoning about it in the abstract.

`doc list` only returns simple/list-view fields by default; use `doc get` on
a specific name when you need the full record including child tables.

## Writing data — never without explicit, per-write permission

**This is the one rule in this skill that overrides everything else,
including an "auto-approve" or autonomous permission mode the session might
otherwise be running in.**

Before running any mutating command — `doc create`, `doc update`,
`doc delete`, `doc submit`, `doc cancel`, `doc amend`, or any `api` call that
writes data — stop and tell the user, in plain terms, exactly what is about
to change:

- which DocType and document name (or "a new X" if creating)
- which fields are changing, old value → new value where relevant
- any side effects that aren't obvious (e.g. "this will also queue an email")

Then wait for an explicit go-ahead on *that specific write*. A prior "yes, go
ahead" on the broader task does not carry over to a write you haven't
described yet — each mutation gets its own confirmation. This holds even if
the user has approved automation broadly elsewhere in the session; live
ERPNext data is shared, hard to notice a mistake in, and often feeds
accounting or customer-facing output, so the cost of asking is low and the
cost of an unwanted write is not.

Once confirmed, the actual commands:

```sh
# simple scalar fields
frappe-cli doc update ToDo abc123 --set status=Closed --yes --json

# nested/child-table data — pipe JSON instead of --set
cat <<'EOF' | frappe-cli doc create "Sales Invoice" --json
{"customer": "...", "items": [{"item_code": "...", "qty": 1}]}
EOF

# lifecycle
frappe-cli doc submit "Sales Invoice" SINV-0001 --yes --json
```

`--yes` is required for mutations to run non-interactively — but getting the
user's go-ahead in conversation is a separate, prior step, not a substitute
for it. `--yes` just satisfies the CLI's own non-interactive guard.

## Quick reference

| Task | Command |
|---|---|
| Who am I connected as | `auth whoami` |
| List DocTypes | `doctype list` |
| Inspect a DocType's fields | `doctype show "<DocType>"` |
| List/filter documents | `doc list "<DocType>" -f field=value` |
| Full record incl. child tables | `doc get "<DocType>" "<name>"` |
| Run a named report | `report run "<Report Name>" -f key=value` |
| Raw whitelisted method call | `api method/<dotted.path> -F key=value` |
| Update a record (confirm first) | `doc update "<DocType>" "<name>" --set field=value --yes` |
| Create a record (confirm first) | `... \| doc create "<DocType>"` |
| Upload/attach a file | `file upload ./x.pdf --doctype "<DocType>" --name <name>` |

Run `frappe-cli <command> --help` or `frappe-cli guide` for anything not
covered here.

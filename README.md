# Nessteams Pricing Button

Zoho CRM widget (Sigma extension) — the **Nessteams Pricing Simulator** for a
**Custom Button on an existing Deal** record.

This is one of two sibling projects under `Nessteams Pricing Calculator/`:

| Project | Launch context | Reads inputs via | Writes outputs via | Status text |
|---------|----------------|------------------|--------------------|-------------|
| **nessteams-pricing-button** (this one) | Custom Button on an **existing** Deal | `ZOHO.CRM.API.getRecord` | `ZOHO.CRM.API.updateRecord` | "Connected to Zoho CRM" |
| nessteams-pricing-wizard | Wizard during record **creation** (no record yet) | `PageLoad` payload (`readWizardField`) | `ZOHO.CRM.WIZARD.post` | "Connected to Wizard" |

> The two `widget.html` files are **identical except** the CRM read/write layer.
> This build uses `getRecord`/`updateRecord` against a real record id resolved
> from the `PageLoad` payload (`resolveEntityId`). If no record id is present it
> falls back to **Manual Mode** (it does not try to write to a wizard form).

## What it does
Opened from a button on an existing Deal, it loads the record's current
`Salary_Low`/`Salary_High`/`Seniority`/`Job_Role`, lets the user adjust seniority
and salary range with sliders, computes client prices, and writes the values
back to the same record on Save.

- **Reads / writes** (Deals module): `Salary_Low`, `Salary_High`, `Seniority`,
  `Seat_Cost_Min`, `Seat_Cost_Max`, `Job_Role`. `Deal_Name` is the job title /
  cheat-sheet lookup key.

## Pricing formula (do not change without intent)
- `MULT = 1.25` — cost multiplier (salary + Panama employer taxes/fees).
- `price = r125(salary * 1.25 + max(marginDollars, TIER_FLOORS[tier]))`
- `r125(v) = Math.ceil(v / 125) * 125` — rounds **up** to nearest 125.
- `TIER_FLOORS` is currently a **flat 1000** for all tiers (business decision
  2026-06-16); per-tier values are kept in the code for easy reinstatement.

## Backend dependency
Calls the Catalyst `cheatsheet-api` Advanced I/O function (Node 20) for role
lookups and usage counts. `CS_API` in `widget.html` points at the **absolute**
Catalyst URL (required because the widget is served from `*.zappsusercontent.com`):

```
https://pricing-calculator-916648406.development.catalystserverless.com/server/cheatsheet-api
```

**Access key.** Writes are gated by a shared key sent **in the request body**
(`{ key: CS_KEY }`), with `Content-Type: text/plain` so the call stays CORS
"simple" and avoids an OPTIONS preflight (Catalyst's gateway doesn't return CORS
headers on preflights → a header-based key would block the request entirely).
`CS_KEY` is a constant in `widget.html` and **must match `WIDGET_KEY`** in
`../nessteams-pricing-backend/functions/cheatsheet-api/index.js` (current value
`nss-cheatsheet-7Qx3v9Kp2Lr8`). Obscurity-level protection — the key is visible
in page source — not true auth. See `../nessteams-pricing-backend/README.md`.

**Writes on save.** The save button calls `POST /upsert` with
`{ Role, Seniority, MinSalary?, MaxSalary? }` — creating the role if the
Role + Seniority pair is new (salary seeded from the sliders) or bumping its
`Counter` if it exists. A typed role with no match shows a "+ Use new role"
option in the search box; salary is only sent for brand-new roles so existing
curated ranges aren't overwritten.

Currently the **development** environment only. See `../legacy/HANDOFF.md` for
promoting to production and `../MEMORY.md` for project context.

## Build & deploy
This is a standard Zoho ZET scaffold. From this folder:

```
zet pack
powershell -ExecutionPolicy Bypass -File .\fix-zip.ps1
```

`zet pack` produces `dist/extension.zip` but deliberately omits
`plugin-manifest.json`; `fix-zip.ps1` injects it back in. Then upload
`dist/extension.zip` at <https://sigma.zoho.com> and wire it to a Custom Button
on the Deals module.

> **Index Page must be `/widget.html`** (NOT `/app/widget.html`) — Zoho roots
> the server inside `app/`, so the `/app` prefix would 404.

## Files
```
plugin-manifest.json        { "service": "CRM" }
app/widget.html             the widget (button-adapted)
app/favicon.png
app/translations/en.json
package.json, server/       local ZET dev server (not shipped to Sigma)
fix-zip.ps1                 post-pack manifest injection
cert.pem, key.pem           local dev HTTPS certs
```

# Pricing Button — Zoho CRM widget

A Zoho CRM embedded widget (Sigma extension) that computes client-facing deal
pricing for an **existing** deal. It opens from a custom button on a Deal record,
loads the record's current values, and writes the recomputed pricing back to the
same record.

This is one of two UI variants of the same pricing engine. The
[Wizard widget](../zohocrm-pricing-wizard) is the other; they share all pricing
and cheat-sheet logic and differ only in how they read from and write to the CRM.

## What it does

1. **Reads** the deal's current salary band, seniority, and job role from the
   record (via `ZOHO.CRM.API.getRecord`, using the record id from the launch
   payload).
2. **Prices** the role live as the rep adjusts seniority and the salary/margin
   sliders.
3. **Writes** the salary band and seat-cost min/max back to the same record on
   save (via `ZOHO.CRM.API.updateRecord`).
4. **Suggests** typical salary ranges by querying the shared backend "cheat sheet."

If the widget is ever opened without a resolvable record id, it falls back to a
read-only **Manual Mode** (calculate only, no write).

## Pricing

```
cost  = salary × MULT                   // grosses salary up to true employer cost
price = roundUpTo125( cost + margin )    // margin floored per seniority tier
```

- `MULT` grosses a salary up to its real cost (salary + employer taxes/fees).
- `margin` is the rep's target but is clamped to a per-seniority minimum, so a role
  can't be accidentally underpriced.
- Prices round **up** to the nearest 125, and the effective margin % is shown back
  to the rep as a sanity check.

## Backend dependency

Calls the companion [Catalyst backend](../zohocrm-pricing-backend) for role lookups
and usage counts. Because the widget is served cross-origin from Zoho's widget
host, `CS_API` in `widget.html` is an **absolute** URL to the function, and writes
carry a shared key **in the request body** (not a header) to stay within the
browser's CORS "simple request" rules — see the backend README for the full
reasoning. Set `CS_API` and `CS_KEY` in `widget.html` before packaging.

> Unlike the Wizard variant, this build writes through a normal record API whose
> promise resolves cleanly, so it needs no save-timeout workaround.

## Build & deploy

Standard Zoho ZET widget. From this folder:

```
zet pack
powershell -ExecutionPolicy Bypass -File ./fix-zip.ps1
```

`zet pack` produces `dist/extension.zip` but omits `plugin-manifest.json`;
`fix-zip.ps1` injects it back in. Upload `dist/extension.zip` at
<https://sigma.zoho.com> and wire it to a custom button on the Deals module.

> The widget's Index Page must be `/widget.html` (not `/app/widget.html`) — Zoho
> serves from inside `app/`, so the `/app` prefix would 404.

## Layout

```
app/widget.html          widget UI + logic (button variant)
app/favicon.png
app/translations/en.json
plugin-manifest.json      { "service": "CRM" }
package.json, server/     local ZET dev server (not shipped to Sigma)
fix-zip.ps1               post-pack manifest injection
```

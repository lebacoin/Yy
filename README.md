# Lebaneeds Payments — Landing Page

Enterprise B2B fintech infrastructure landing page for **Lebaneeds Payments**.

Static site — no build step required.

```
connect.html  # Connect product site (root of connect.lebaneeds.com)
home.html     # brand landing (root of lebaneeds.com)
partners.html # For Payment Partners page
privacy.html  # privacy notice
styles.css    # design system & responsive layout
script.js     # mobile nav, API example tabs, contact forms, scroll reveals
vercel.json   # host routing + security headers
```

Open `connect.html` or `home.html` in a browser, or serve locally:

```sh
python3 -m http.server 8000
```

## Notes

- Light theme first, with automatic dark mode via `prefers-color-scheme`.
- Fully responsive (desktop → mobile), accessible (skip link, ARIA tabs, focus states, reduced-motion support).
- Lebaneeds Payments is presented as a technology company; regulated payment services are attributed to licensed financial partners throughout the copy.

## CleanWallet build plan

`docs/cleanwallet/` holds the specification for **CleanWallet** — the payment-assurance workflow for
high-value OTC crypto deals (non-custodial: dealer-held keys, read-only checks). Start at
[`docs/cleanwallet/README.md`](docs/cleanwallet/README.md).

Proposed workflow, not a live product. No backend exists in this repository.

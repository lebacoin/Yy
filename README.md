# Lebaneeds Payments — Landing Page

Enterprise B2B fintech infrastructure landing page for **Lebaneeds Payments**.

Static site — no build step required.

```
connect.html # Connect product site (index of connect.lebaneeds.com)
home.html    # brand landing (lebaneeds.com)
partners.html# For Payment Partners page
styles.css   # design system & responsive layout
script.js    # mobile nav, API example tabs, scroll reveals
```

Open `connect.html` or `home.html` in a browser, or serve locally:

```sh
python3 -m http.server 8000
```

## Notes

- Light theme first, with automatic dark mode via `prefers-color-scheme`.
- Fully responsive (desktop → mobile), accessible (skip link, ARIA tabs, focus states, reduced-motion support).
- Lebaneeds Payments is presented as a technology company; regulated payment services are attributed to licensed financial partners throughout the copy.

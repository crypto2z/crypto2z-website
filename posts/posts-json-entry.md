Add this as the FIRST entry in `posts.json` (before the celeron-to-ryzen entry):

```json
{
  "slug": "2026-07-27-rack-dl380-migration-self-hosted-bots",
  "title": "Rack, Rebuild, Repeat: Two Server Migrations and a NAS Build in Ten Weeks",
  "date": "2026-07-27",
  "category": "node-setup",
  "tags": [
    "ethereum",
    "dl380",
    "homelab",
    "rack",
    "ssv",
    "trading-bots",
    "self-hosted",
    "truenas"
  ],
  "excerpt": "From a Ryzen tower on a desk to a proper 42U rack running a dual-Xeon DL380 Ethereum node, a NAS build in progress, and two trading bots migrated off DigitalOcean — everything that changed in ten weeks."
}
```

Then commit both files to `crypto2z/crypto2z-website`:
- `posts/2026-07-27-rack-dl380-migration-self-hosted-bots.md`
- updated `posts.json`

Push to `main` and Cloudflare Pages auto-deploys.

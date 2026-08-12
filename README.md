# Driftbound Labs — site

Static site for driftboundlabs.com, served by GitHub Pages.

## Deploy (one time)

1. Create a public repo (e.g. `driftbound-site`) and push these files to `main`:

   ```
   git init
   git add index.html CNAME README.md
   git commit -m "Initial site"
   git branch -M main
   git remote add origin git@github.com:<USERNAME>/driftbound-site.git
   git push -u origin main
   ```

2. Repo → **Settings → Pages** → Source: *Deploy from a branch* → `main` / `/ (root)` → Save.

3. In the same Pages screen, the custom domain should auto-populate from the
   `CNAME` file as `driftboundlabs.com`. If not, enter it and save.

## DNS (at your registrar)

Apex domain — four **A** records on `@`:

```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

Optional but recommended — **CNAME** record on `www` pointing to:

```
<USERNAME>.github.io
```

Delete any registrar parking/forwarding records on `@` first; they conflict.

## HTTPS

After DNS propagates (minutes to ~24 h), GitHub Pages provisions a Let's
Encrypt certificate automatically. Return to Settings → Pages and check
**Enforce HTTPS** once the option becomes clickable.

## Email

`hello@driftboundlabs.com` on the contact card is a placeholder until
forwarding exists. Most registrars offer free email forwarding — point
`hello@` at a real inbox, or swap the address in `index.html`.

## Verify before Aug 19

- `https://driftboundlabs.com` loads with a padlock
- `https://www.driftboundlabs.com` redirects to the apex (if the www CNAME was added)
- Page renders correctly on a phone
- Contact email actually delivers

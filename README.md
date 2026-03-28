# FranchiseDealBreaker

**franchisedealbreaker.com** — AI-powered franchise due diligence. Find the red flags before you sign.

## Repo Structure

```
/
├── index.html              # Main landing page
├── CNAME                   # Custom domain config for GitHub Pages
├── _config.yml             # GitHub Pages config
├── 404.html                # Custom 404 page
├── privacy.html            # Privacy policy
├── terms.html              # Terms of service
├── database.html           # Franchise database index
└── franchises/
    ├── subway.html         # Individual franchise profile (template)
    ├── anytime-fitness.html
    └── ...                 # One file per franchise
```

## Deployment

This site is deployed via GitHub Pages with a custom domain.

- Branch: `main`
- Custom domain: `franchisedealbreaker.com`
- HTTPS: Enforced via GitHub Pages settings

## DNS Setup (Namecheap)

In Namecheap, set the following A records pointing to GitHub Pages:

```
A  @  185.199.108.153
A  @  185.199.109.153
A  @  185.199.110.153
A  @  185.199.111.153
```

And a CNAME record:
```
CNAME  www  jaredleiman.github.io
```

## Adding Franchise Pages

1. Copy `franchises/subway.html` as a template
2. Replace all `[FRANCHISE_NAME]`, `[INVESTMENT_RANGE]`, etc. placeholders
3. Paste in the Claude-generated analysis content
4. Commit and push — GitHub Pages deploys in ~60 seconds

## Stripe Payment Links

Update the two CTA buttons in `index.html`:
- Quick Scan ($97): Replace `href="#"` on the Quick Scan button
- Full Report ($497): Replace `href="#"` on the Full Report button

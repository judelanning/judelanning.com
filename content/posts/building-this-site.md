---
title: "building this site"
date: 2026-01-26
draft: false
summary: "a quick walkthrough of the tech behind this site."
---

i recently put together this personal site and wanted to document the setup while it's still fresh.

## the stack

- **hugo** for static site generation
- **s3** for hosting
- **cloudflare** for dns, cdn, and https

## why hugo

i wanted something simple. no databases, no server-side code, just html and css. hugo fit the bill. it's fast, has a straightforward templating system, and generates plain static files.

## custom theme

i built a minimal theme from scratch rather than using a pre-made one. the aesthetic i was going for was something between an academic paper and a medieval manuscript. the key pieces:

- **eb garamond** font from google fonts
- parchment-inspired color palette
- a subtle wobbly border effect using svg filters (`feTurbulence` and `feDisplacementMap`)
- all lowercase text via css `text-transform`

the svg filter trick is fun. it gives the border a hand-drawn quality without any images.

```html
<svg>
  <filter id="wobbly">
    <feTurbulence type="fractalNoise" baseFrequency=".02" numOctaves="3" />
    <feDisplacementMap in="SourceGraphic" scale="5" />
  </filter>
</svg>
```

then apply it with `filter: url(#wobbly)` in css.

## hosting on s3

hugo generates a `public/` directory with all the static files. this is what gets uploaded to s3.

```
public/
├── index.html
├── index.xml
├── 404.html
├── sitemap.xml
├── about/
│   └── index.html
├── posts/
│   ├── index.html
│   └── building-this-site/
│       └── index.html
└── assets/
    └── css/
```

when a visitor requests `judelanning.com/about/`, s3 serves `about/index.html`. it's just files in a bucket.

to make these files publicly accessible, the bucket has a policy that allows `s3:GetObject` from any principal.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::judelanning.com/*"
    }
  ]
}
```

this means anyone can download the html, css, and any other files in the bucket.

the setup steps are:

1. create a bucket named after your domain
2. enable static website hosting
3. set index document to `index.html`
4. add a bucket policy for public read access
5. disable the public access block

deployment is just `hugo --minify && aws s3 sync public/ s3://yourbucket --delete`.

## cloudflare for https

s3 website endpoints don't support https directly. cloudflare solves this by sitting in front.

- point your domain's dns to cloudflare
- add a cname record pointing to your s3 website endpoint
- enable the proxy (orange cloud)
- set ssl mode to "flexible"

this gives you free https, caching, and ddos protection.

## cost

one of the nice things about this setup is it's essentially free.

- **s3** charges for storage and requests, but for a small static site it's pennies per month
- **cloudflare** free tier covers everything i need. dns, cdn, ssl, and ddos protection
- **hugo** is open source
- **github** free tier for the repo

i expect to pay less than $1/month unless the site gets significant traffic. if that happens, s3 costs would go up from GetObject requests and data transfer out. cloudflare's caching helps here since it serves repeat visitors from their edge servers instead of hitting s3 every time.

## future improvements

right now deployment is manual. i run `hugo` to build and `aws s3 sync` to upload. it works, but i'd like to automate it.

the plan is to set up a github actions workflow that triggers on merges to main. write changes, push to github, and the site updates automatically. no need to run deployment commands locally.

## source

the full source is on [github](https://github.com/judelanning/judelanning.com) if you want to poke around.

# enable-feedproof

A [Claude Code skill](https://skills.sh) that connects a [Vercel Shop](https://vercel.shop) storefront to [FeedProof](https://getfeedproof.com), the Shopify app that fixes product data for AI shopping agents.

## Install

Run this in the storefront repo:

```bash
npx skills add danielcanton/enable-feedproof-skill
```

## What it does

- **Logs AI-agent visits.** `proxy.ts` tells FeedProof when GPTBot, ClaudeBot, PerplexityBot or another AI agent fetches a page, and whether it read HTML or Markdown. The report runs in `waitUntil` and never delays the response.
- **Adds product details for agents.** It reads the `feedproof.*` metafields (material, care, country of origin, dimensions, warranty, condition) and the variant barcode. They go into the product Markdown as a Details section, and into Product JSON-LD as `material`, `countryOfOrigin`, `gtin`, `itemCondition` and `additionalProperty`.

Tested against Vercel Shop template 2026.10.0: typecheck, lint, build and runtime checks.

## Before you run it

In the Shopify admin, open FeedProof → Settings → Headless storefront → Connect storefront. That shows the `FEEDPROOF_INGEST_KEY` once, and makes the metafields readable by the Storefront API. The code builds without the key; it just doesn't report.

## Usage

Run `/enable-feedproof` in Claude Code from the storefront repo.

## License

MIT

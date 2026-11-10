---
name: scrapfly-sdk-typescript
description: >-
  Web scraping from TypeScript or JavaScript with the Scrapfly SDK
  (`scrapfly-sdk`). Use when the user asks to scrape a URL, bypass anti-bot
  protection, render JavaScript, capture screenshots, or extract structured
  data from Node.js, Deno, or Bun.
---

# Scrapfly TypeScript SDK

Use the `scrapfly-sdk` package to scrape web pages from TypeScript/JavaScript
with proxy rotation, anti-bot bypass (Unblocker), JavaScript rendering, screenshots,
and AI extraction. Runs on Node.js, Deno, Bun, and edge runtimes
(Cloudflare Workers, AWS Lambda).

## When to use

- Scraping web pages (HTML, JSON, text, markdown) from a TS/JS codebase
- Bypassing anti-bot protections (Cloudflare, DataDome, PerimeterX, Kasada)
- Rendering JavaScript-heavy pages with a headless browser
- Capturing screenshots or extracting structured data
- Concurrent scraping of many URLs

## Setup

```bash
npm install scrapfly-sdk
# Deno / JSR: import from 'jsr:@scrapfly/scrapfly-sdk'
export SCRAPFLY_API_KEY="your-api-key"
```

Get your API key at [scrapfly.io](https://scrapfly.io).

## Basic scrape

```typescript
import { ScrapflyClient, ScrapeConfig } from 'scrapfly-sdk';

const client = new ScrapflyClient({ key: process.env.SCRAPFLY_API_KEY });

const result = await client.scrape(
    new ScrapeConfig({ url: 'https://web-scraping.dev/products' }),
);

console.log(result.result.content);
```

## Anti-bot bypass, geo-targeting, JS rendering

`ScrapeConfig` uses snake_case option keys, matching the API.

```typescript
const result = await client.scrape(
    new ScrapeConfig({
        url: 'https://web-scraping.dev/products',
        proxy_pool: 'unblocker',            // residential proxies + the Unblocker (anti-bot bypass)
        country: 'us',                      // proxy country (ISO 3166-1 alpha-2)
        render_js: true,                    // headless browser rendering
        wait_for_selector: 'div.product',   // requires render_js
        rendering_wait: 5000,               // ms, requires render_js
        format: 'markdown',                 // raw | clean_html | json | markdown | text
    }),
);
```

## Screenshots and extraction

```typescript
import { ScreenshotConfig, ExtractionConfig } from 'scrapfly-sdk';

const shot = await client.screenshot(
    new ScreenshotConfig({ url: 'https://web-scraping.dev/products' }),
);
// shot.image is the binary image data

const html = (await client.scrape(
    new ScrapeConfig({ url: 'https://web-scraping.dev/reviews' }),
)).result.content;

const data = await client.extract(
    new ExtractionConfig({
        body: html,
        content_type: 'text/html',
        extraction_prompt: 'extract the reviews as JSON with fields: review, date, rating, author',
    }),
);
console.log(data.data);
```

## Concurrent scraping

```typescript
const configs = [
    new ScrapeConfig({ url: 'https://web-scraping.dev/products?page=1' }),
    new ScrapeConfig({ url: 'https://web-scraping.dev/products?page=2' }),
];

for await (const result of client.concurrentScrape(configs)) {
    console.log(result.result.content.slice(0, 100));
}
```

## Important notes

- `render_js: true` is required for `rendering_wait`, `wait_for_selector`, and screenshots
- `proxy_pool` picks the proxy pool: `'basic'` (datacenter proxies, Unblocker
  off), `'unblocker'` (residential proxies and the Unblocker),
  `'unblocker_ultra'` (the same, and the only pool a target or an anti-bot
  protection with a price of its own accepts) or `'auto'` (the API picks the
  pool before the request, from the target's profile: the Unblocker on
  datacenter proxies, or the pool the target needs; accepted on every target,
  the ones that require `unblocker_ultra` included, where their price is
  added; priced as the pool it runs on). Each pool sets the Unblocker itself.
  A target or an anti-bot protection with a price of its own answers `basic`
  and `unblocker` with `ERR::PROXY::UNBLOCKER_ULTRA_REQUIRED` (HTTP 422, not
  billed): resend with `proxy_pool: 'unblocker_ultra'`. `ScreenshotConfig`
  takes the same `proxy_pool` from the release after 0.12.0
- `unblocker: true` and its alias `asp` are deprecated: use `proxy_pool`
  (`'unblocker_ultra'` for the Unblocker). The
  [migration guide](https://scrapfly.io/new-pricing#legacy-spellings) covers
  the requests written before the proxy pools, the pool names
  `'public_datacenter_pool'` and `'public_residential_pool'` included. An
  explicitly supplied `asp` wins over `unblocker`
- Use `format: 'markdown'` for clean, LLM-friendly content
- The API key is read from the `key` constructor option (pass `process.env.SCRAPFLY_API_KEY`)
- Read the page body via `result.result.content`

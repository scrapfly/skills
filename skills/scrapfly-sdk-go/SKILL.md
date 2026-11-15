---
name: scrapfly-sdk-go
description: >-
  Web scraping from Go with the Scrapfly SDK
  (`github.com/scrapfly/go-scrapfly`). Use when the user asks to scrape a URL,
  bypass anti-bot protection, render JavaScript, capture screenshots, or
  extract structured data from a Go program.
---

# Scrapfly Go SDK

Use the `go-scrapfly` module to scrape web pages from Go with proxy rotation,
anti-bot bypass (Unblocker), JavaScript rendering, screenshots, and AI extraction.
It ships a built-in `goquery` selector for parsing the response.

## When to use

- Scraping web pages (HTML, JSON, text, markdown) from a Go codebase
- Bypassing anti-bot protections (Cloudflare, DataDome, PerimeterX, Kasada)
- Rendering JavaScript-heavy pages with a headless browser
- Capturing screenshots or extracting structured data

## Setup

```bash
go get github.com/scrapfly/go-scrapfly
export SCRAPFLY_API_KEY="your-api-key"
```

The package name is `scrapfly`. Get your API key at [scrapfly.io](https://scrapfly.io).

## Basic scrape

```go
package main

import (
    "fmt"
    "log"
    "os"

    "github.com/scrapfly/go-scrapfly"
)

func main() {
    client, err := scrapfly.New(os.Getenv("SCRAPFLY_API_KEY"))
    if err != nil {
        log.Fatalf("failed to create client: %v", err)
    }

    apiResponse, err := client.Scrape(&scrapfly.ScrapeConfig{
        URL: "https://web-scraping.dev/product/1",
    })
    if err != nil {
        log.Fatalf("scrape failed: %v", err)
    }

    fmt.Println(apiResponse.Result.Content)
}
```

## Anti-bot bypass, geo-targeting, JS rendering

`ScrapeConfig` fields are Go-idiomatic PascalCase. `ProxyPool` is a typed
string: pass the pool as a string literal, `"basic"`, `"unblocker"`,
`"unblocker_ultra"` or `"auto"`, which every go-scrapfly release accepts.

```go
apiResponse, err := client.Scrape(&scrapfly.ScrapeConfig{
    URL:             "https://web-scraping.dev/product/1",
    ProxyPool:       "unblocker",   // residential proxies + the Unblocker (anti-bot bypass)
    Country:         "us",          // proxy country (ISO 3166-1 alpha-2)
    RenderJS:        true,          // headless browser rendering
    WaitForSelector: "div.product", // requires RenderJS
    RenderingWait:   5000,          // ms, requires RenderJS
    Format:          "markdown",    // raw | clean_html | json | markdown | text
})
```

## Built-in HTML selector

```go
selector, err := apiResponse.Selector() // goquery document
if err != nil {
    log.Fatal(err)
}
fmt.Println("Title:", selector.Find("h3").First().Text())
```

## Screenshots and extraction

```go
shot, err := client.Screenshot(&scrapfly.ScreenshotConfig{
    URL: "https://web-scraping.dev/products",
})

data, err := client.Extract(&scrapfly.ExtractionConfig{
    Body:             []byte(apiResponse.Result.Content),
    ContentType:      "text/html",
    ExtractionPrompt: "extract all product names and prices",
})
```

## Important notes

- `RenderJS: true` is required for `RenderingWait`, `WaitForSelector`, and screenshots
- `ProxyPool: "basic"` is datacenter proxies with the Unblocker off,
  `"unblocker"` residential proxies with the Unblocker, `"unblocker_ultra"` the
  same and the only pool a target or an anti-bot protection with a price of its
  own accepts, and `"auto"` lets the API pick the pool before the request, from
  the target's profile: the Unblocker on datacenter proxies, or the pool the
  target needs (accepted on every target, the ones that require
  `unblocker_ultra` included, where their price is added; priced as the pool
  it runs on). Each pool sets the Unblocker itself. A target or an anti-bot
  protection with a price of its own answers `basic` and `unblocker` with
  `ERR::PROXY::UNBLOCKER_ULTRA_REQUIRED` (HTTP 422, not billed): resend with
  `ProxyPool: "unblocker_ultra"`. `ScreenshotConfig.ProxyPool` comes with the
  go-scrapfly release after v1.6.0
- `Unblocker` (a `*bool`, set with `scrapfly.BoolPtr(true)`, nil = unset) and
  its alias, the plain `bool` field `ASP`, are deprecated: use `ProxyPool`
  (`"unblocker_ultra"` for the Unblocker). The
  [migration guide](https://scrapfly.io/new-pricing#legacy-spellings) covers
  the requests written before the proxy pools, the pool names
  `scrapfly.PublicDataCenterPool` and `scrapfly.PublicResidentialPool`
  included. Both fields go to the API as `asp`; an explicit `ASP: true` wins
  over `Unblocker`
- Use `Format: "markdown"` for clean, LLM-friendly content
- Read the page body via `apiResponse.Result.Content`; parse with `apiResponse.Selector()`

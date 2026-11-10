---
name: scrapfly-browser
description: >-
  Automate cloud browsers using the Scrapfly Cloud Browser API with Python
  Playwright. Covers connection options, sessions, anti-bot fingerprinting,
  and file downloads including streaming large files (up to
  500 MB) out of the session with downloads_stream=1 and the CDP IO domain.
---

# Scrapfly Cloud Browser

Use the Scrapfly Cloud Browser API with Python Playwright to automate remote cloud browsers with built-in proxy rotation, anti-bot fingerprinting, and geo-targeting.

## When to use

- Automating browser interactions on remote cloud browsers (login flows, form filling, navigation)
- Scraping JavaScript-heavy sites that require full browser automation
- Bypassing anti-bot protections with managed browser fingerprinting
- Running Playwright scripts through geo-targeted proxies
- Maintaining persistent browser sessions across multiple connections
- Downloading files through browser interactions

## Setup

```bash
pip install playwright
```

The API key must be provided via environment variable `SCRAPFLY_API_KEY` or passed directly in the connection URL.

## API Reference

**Connection:** WebSocket CDP (Chrome DevTools Protocol)

```
wss://browser.scrapfly.io?api_key=YOUR_API_KEY
```

### Connection Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `api_key` | str | **required** | Scrapfly API key |
| `proxy_pool` | str | `"datacenter"` | Proxy type: `"datacenter"` or `"residential"`. The Web Scraping API words `basic`, `unblocker`, `unblocker_ultra` and `auto` do not apply to the Cloud Browser, which refuses them with `ERR::BROWSER::CONFIG_ERROR` |
| `os` | str | random | OS fingerprint: `"linux"`, `"windows"`, or `"macos"` |
| `country` | str | None | Proxy country (ISO 3166-1 alpha-2, e.g. `"us"`, `"de"`) |
| `session` | str | None | Session ID for browser state persistence |
| `auto_close` | str | `"true"` | Close session on disconnect. Set `"false"` to keep alive |
| `timeout` | int | 900 | Max session duration in seconds (900-1800) |
| `debug` | str | `"false"` | Enable session video recording |
| `downloads_stream` | str | None | `"1"` returns large downloads as IO stream handles instead of inline base64 |

### Building the Connection URL

```python
import os
from urllib.parse import urlencode

params = {
    "api_key": os.environ["SCRAPFLY_API_KEY"],
    "proxy_pool": "datacenter",
    "os": "linux",
}
BROWSER_WS = f"wss://browser.scrapfly.io?{urlencode(params)}"
```

## Examples

### Basic page navigation

```python
from playwright.sync_api import sync_playwright
import os

API_KEY = os.environ["SCRAPFLY_API_KEY"]
BROWSER_WS = f"wss://browser.scrapfly.io?api_key={API_KEY}&proxy_pool=datacenter&os=linux"

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto("https://web-scraping.dev")
        print("Title:", page.title())
        print("Content:", page.content()[:500])
    finally:
        browser.close()
```

### Async Playwright

```python
from playwright.async_api import async_playwright
import asyncio
import os

API_KEY = os.environ["SCRAPFLY_API_KEY"]
BROWSER_WS = f"wss://browser.scrapfly.io?api_key={API_KEY}&proxy_pool=datacenter&os=linux"

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.connect_over_cdp(BROWSER_WS)
        try:
            context = browser.contexts[0]
            page = context.pages[0] if context.pages else await context.new_page()

            await page.goto("https://web-scraping.dev")
            print("Title:", await page.title())
        finally:
            await browser.close()

asyncio.run(main())
```

### Login flow with form filling

```python
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto("https://web-scraping.dev/login")
        page.fill("input[name=username]", "user123")
        page.fill("input[name=password]", "password")
        page.click("button[type=submit]")
        page.wait_for_selector("div#secret-message")

        print("Logged in:", page.url)
    finally:
        browser.close()
```

### Take a screenshot

```python
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto("https://web-scraping.dev")
        page.screenshot(path="screenshot.png", full_page=True)
    finally:
        browser.close()
```

### Scrape data with selectors

```python
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto("https://web-scraping.dev/products")
        page.wait_for_selector("div.product")

        products = page.query_selector_all("div.product")
        for product in products:
            name = product.query_selector("a").inner_text()
            price = product.query_selector(".price").inner_text()
            print(f"{name}: {price}")
    finally:
        browser.close()
```

### Geo-targeted browsing with residential proxy

```python
BROWSER_WS = (
    f"wss://browser.scrapfly.io?"
    f"api_key={API_KEY}"
    f"&proxy_pool=residential"
    f"&country=de"
    f"&os=windows"
)

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto("https://web-scraping.dev")
        print("Page content from DE proxy:", page.title())
    finally:
        browser.close()
```

### Persistent sessions (multi-step workflows)

Uses the same login credentials and selectors as the [Login flow with form filling](#login-flow-with-form-filling) example.

```python
LOGIN_URL = "https://web-scraping.dev/login"
USERNAME = "user123"
PASSWORD = "password"

# Step 1: Login and keep session alive
BROWSER_WS = (
    f"wss://browser.scrapfly.io?"
    f"api_key={API_KEY}"
    f"&session=my-session"
    f"&auto_close=false"
    f"&proxy_pool=datacenter"
)

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto(LOGIN_URL)
        page.fill("input[name=username]", USERNAME)
        page.fill("input[name=password]", PASSWORD)
        page.click("button[type=submit]")
        page.wait_for_selector("div#secret-message")
        print("Login complete, session preserved")
    finally:
        browser.close()  # Disconnects but session stays alive

# Step 2: Reconnect to same session (still authenticated with same credentials)
BROWSER_WS = (
    f"wss://browser.scrapfly.io?"
    f"api_key={API_KEY}"
    f"&session=my-session"
    f"&proxy_pool=datacenter"
)

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto(LOGIN_URL)
        page.wait_for_selector("div#secret-message")
        print("Accessed protected page (still logged in):", page.title())
    finally:
        browser.close()
```

### Infinite scroll scraping

```python
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto("https://web-scraping.dev/testimonials")
        page.wait_for_selector("span.rating")

        all_items = []
        for _ in range(10):  # Scroll 10 times
            items = page.query_selector_all("span.rating")
            all_items = items
            page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
            page.wait_for_timeout(2000)

        print(f"Collected {len(all_items)} reviews")
    finally:
        browser.close()
```

### Execute JavaScript in page

```python
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        page.goto("https://web-scraping.dev")

        # Extract data using JavaScript
        data = page.evaluate("""
            () => {
                return {
                    title: document.title,
                    links: Array.from(document.querySelectorAll('a')).map(a => ({
                        text: a.innerText,
                        href: a.href,
                    })),
                };
            }
        """)
        print(f"Found {len(data['links'])} links")
    finally:
        browser.close()
```

### Multi-page navigation

```python
with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()

        urls = [
            "https://web-scraping.dev/products",
            "https://web-scraping.dev/products?page=2",
            "https://web-scraping.dev/products?page=3",
        ]

        results = []
        for url in urls:
            page.goto(url)
            results.append({"url": url, "content": content})

        print(f"Scraped {len(results)} pages")
    finally:
        browser.close()
```

## File Downloads

A download triggered in the page (a click behind a login, an anti-bot flow) is
captured by the session and read back over CDP with the `ScrapiumBrowser.*`
commands. They are browser-level: send them on a CDP session.

| Command | Params | Returns |
|---------|--------|---------|
| `ScrapiumBrowser.hasDownloads` | none | `{"result": true\|false}` |
| `ScrapiumBrowser.getDownloadsMetadatas` | none | `{"metadata": {"<filename>": <bytes>}}` — sizes only, no content, no state |
| `ScrapiumBrowser.getDownloadMetadatas` | `filename` | `{"size": <bytes>}` for one file |
| `ScrapiumBrowser.getDownload` | `filename` (required), `delete` (optional) | `{"data": "<base64>"}` — one file, content carried once |
| `ScrapiumBrowser.getDownloads` | `delete` (optional) | every file's content, carried twice: `files` (by name) and `files_by_guid` |

### Wait before you read

Nothing queues the read behind the download. Listen for `Browser.downloadProgress`
with `state == "completed"`, or poll `getDownloadsMetadatas` until the file is
there and its size has stopped moving. Calling `getDownloads` straight after the
click returns `{}` and your lookup raises `KeyError`.

```python
import base64
import time

def wait_for_download(cdp, filename, timeout=60):
    """Poll until the file exists and its size is stable across two reads."""
    deadline = time.monotonic() + timeout
    last = -1
    while time.monotonic() < deadline:
        size = cdp.send("ScrapiumBrowser.getDownloadsMetadatas")["metadata"].get(filename)
        if size is not None and size == last:
            return size
        last = size if size is not None else -1
        time.sleep(0.5)
    raise TimeoutError(f"{filename} did not finish downloading")


with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()
        cdp = context.new_cdp_session(page)

        page.goto("https://web-scraping.dev/file-download")
        page.click("#download-btn")

        size = wait_for_download(cdp, "download-sample.pdf")

        # One file by name: content is carried once, so the reply is ~1.34x the file.
        result = cdp.send("ScrapiumBrowser.getDownload", {"filename": "download-sample.pdf"})
        open("download-sample.pdf", "wb").write(base64.b64decode(result["data"]))
    finally:
        browser.close()
```

Downloaded files are cleaned up when the session ends. Pass `delete: True` to
`getDownload` / `getDownloads` to reclaim disk during a long-running session.

### Large files: size the reply before you ask for it

`getDownloads` answers with the whole payload inside **one** CDP message, and a
WebSocket message above the client's limit is a protocol error, not a slow read:
the client closes with code `1009` and the session dies with it, surfacing as
`Target closed` / `Session closed`.

- `getDownloads` reply ≈ **2.7x** the file — base64 costs a third and the content
  is carried under both `files` and `files_by_guid`.
- `getDownload` reply ≈ **1.34x** the file. Prefer it when you know the name.
- **Server-side cap: 500 MB per file.** An oversized file is never encoded, and
  **the call it appears in fails**: `getDownloads` answers with
  `download too large: "export.csv" (620000000 bytes) exceeds the 524288000 byte limit per file`
  and returns *nothing*, including the small files that were fine.
  `getDownloadsMetadatas` still lists the oversized file, and `getDownload` still
  serves the others by name — that pair is the recovery path.
- Call `getDownloadsMetadatas` first and branch on the real size. Past a few tens
  of megabytes, stream instead of inlining.

### Stream large downloads with `downloads_stream=1`

Add `downloads_stream=1` to the Cloud Browser WebSocket URL. It is a connection
option: nothing later in the session can turn it on. `getDownloads` then answers
with a smaller shape — each file's content **once** under `files_by_guid`, with
`guid_names` mapping GUID to filename, and anything above ~8 MB of base64 handed
back under `streams` as an IO handle instead of inline content:

```json
{
  "files": {},
  "files_by_guid": {"9F2C1A4E-...": "JVBERi0xLjQKJe..."},
  "guid_names": {"9F2C1A4E-...": "invoice.pdf", "B71D0E88-...": "export.csv"},
  "streams": {
    "B71D0E88-...": {"stream": "scrapfly-download:1", "size": 91250688, "name": "export.csv"}
  }
}
```

A `stream` value is an ordinary IO handle, drained with `IO.read` / `IO.close`:

```python
import base64
from urllib.parse import urlencode
from playwright.sync_api import sync_playwright

params = {"api_key": API_KEY, "proxy_pool": "datacenter", "downloads_stream": "1"}
BROWSER_WS = f"wss://browser.scrapfly.io?{urlencode(params)}"

CHUNK = 8 * 1024 * 1024  # base64 characters, not raw bytes (32 MB is the cap)


def drain(cdp, handle, destination):
    """Read a scrapfly-download: handle to eof. Sequential only, no offset."""
    encoded = []
    try:
        while True:
            read = cdp.send("IO.read", {"handle": handle, "size": CHUNK})
            if read.get("data"):
                encoded.append(read["data"])
            if read.get("eof"):
                break
    finally:
        try:
            cdp.send("IO.close", {"handle": handle})
        except Exception:
            pass

    # Chunks are cut on base64 group boundaries, so they concatenate as text
    # and decode once.
    with open(destination, "wb") as out:
        out.write(base64.b64decode("".join(encoded)))


with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(BROWSER_WS)
    try:
        context = browser.contexts[0]
        page = context.pages[0] if context.pages else context.new_page()
        cdp = context.new_cdp_session(page)

        page.goto("https://web-scraping.dev/file-download")
        page.click("#download-btn")
        wait_for_download(cdp, "download-sample.pdf")

        result = cdp.send("ScrapiumBrowser.getDownloads")

        # Small files still arrive inline, large ones as handles. Handle both.
        for guid, b64 in result.get("files_by_guid", {}).items():
            open(result["guid_names"][guid], "wb").write(base64.b64decode(b64))
        for guid, stream in result.get("streams", {}).items():
            drain(cdp, stream["stream"], stream["name"])
    finally:
        browser.close()
```

Rules for download handles (they differ from Chromium's own IO streams):

- **Reads are sequential.** Passing `offset` is an error: bytes are released as
  they are read.
- **`size` counts base64 characters here**, not raw bytes. Chunks are capped at
  32 MB and default to 4 MB when no `size` is sent.
- **Loop until `eof` is true.** An empty `data` without `eof` is not the end.
- **Always `IO.close`**, including on the error path. An abandoned handle is
  reclaimed after 120 seconds, and reaching `eof` releases it, but closing is the
  contract.
- Only `scrapfly-download:<n>` handles are served by the session. `blob:` handles,
  `Page.printToPDF` streams and every other IO handle reach Chromium untouched.

### No download at all: stream a URL out of the page

When the URL is known and the browser download machinery is not needed, fetch it
in the page and stream the Blob. Nothing large ever crosses the wire, and the
500 MB cap does not apply on this path.

```python
CHUNK = 10 * 1024 * 1024  # raw bytes here; the reply inflates ~1.34x

evaluated = cdp.send("Runtime.evaluate", {
    "expression": f"fetch({url!r}).then(r => r.blob())",
    "awaitPromise": True,
})
resolved = cdp.send("IO.resolveBlob", {"objectId": evaluated["result"]["objectId"]})
handle = "blob:" + resolved["uuid"]
# Drain with IO.read / IO.close, decoding each chunk when base64Encoded is true.
```

`Page.printToPDF` takes `transferMode: "ReturnAsStream"` and returns a handle the
same way.

**Do not use `Network.getResponseBody` for a large body.** Chromium evicts big
response bodies from the inspector cache before you can ask
(`Request content was evicted from inspector cache`), and for a text content type
it returns the body already decoded, which silently corrupts binary payloads.

## Anti-Bot Features

The Cloud Browser automatically manages browser fingerprinting:

- TLS/JA3 fingerprints matching real browsers
- User-Agent, device metrics, timezone, locale (based on `country`)
- WebGL and Canvas fingerprint emulation
- HTTP/2 fingerprints

## Important Notes

- **Always use `connect_over_cdp()`** (not `connect()`) to connect to the Cloud Browser
- **Always use `browser.contexts[0]`** to get the pre-configured context. Do NOT create new contexts - the Cloud Browser provides a context with proper fingerprinting
- **Always close the browser in a `try/finally` block** to stop billing. Forgetting to close means continuous charges
- **Do NOT set fingerprint overrides** (user agent, viewport, timezone, etc.) - they are managed automatically
- Sessions expire after 1 hour of inactivity
- Maximum session duration is 30 minutes (1800 seconds)
- Maximum file download size is 500 MB per file, and one oversized file fails the
  whole `getDownloads` call. Past a few tens of megabytes use `downloads_stream=1`
  and drain the IO handle, see [File Downloads](#file-downloads)
- Start with `proxy_pool=datacenter`, upgrade to `residential` only when needed for anti-bot bypass
- Use `debug=true` for session video recording when troubleshooting

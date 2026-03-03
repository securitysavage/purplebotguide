# Web Bots & Scrapers Codex v1.0 — Purple Team Edition

**Purpose:** Teach a junior cybersecurity professional to write Python-based web bots and scrapers from zero to operational, with a dual focus: how to build them (red) and how to detect/defend against them (blue).

**Prerequisites:** Basic Python familiarity (variables, functions, loops, conditionals). Comfort with the Linux command line. A willingness to read HTML source and HTTP headers.

**Why This Matters for Purple Team:** Adversaries use scrapers for OSINT reconnaissance, credential harvesting, data exfiltration, and attack surface mapping. Defenders who can build and operate scrapers understand the traffic patterns, evasion techniques, and fingerprints that show up in WAF logs, SIEM alerts, and network captures. You can't write a good detection rule for something you've never operated.

---

## Section 1: The Web — How It Actually Works Under the Hood

Before you write a single line of scraping code, you need to understand what happens when a browser requests a web page. Every concept in this codex builds on this foundation.

## 1.1 The HTTP Request-Response Cycle

When you type a URL into a browser, here's what happens:

```text
YOUR BROWSER                          WEB SERVER
     │                                     │
     │──── DNS lookup (URL → IP) ─────────>│
     │                                     │
     │──── TCP handshake (SYN/ACK) ───────>│
     │                                     │
     │──── TLS handshake (if HTTPS) ──────>│
     │                                     │
     │──── HTTP GET request ──────────────>│
     │     (includes headers, cookies)     │
     │                                     │
     │<─── HTTP response ─────────────────-│
     │     (status code, headers, body)    │
     │                                     │
     │──── Browser renders HTML ───────────│
     │     (fetches CSS, JS, images)       │
     │                                     │
     │──── JavaScript executes ────────────│
     │     (may fetch MORE data via AJAX)  │
     └─────────────────────────────────────┘
```

**Why this matters for scraping:** Simple scrapers (Requests + BeautifulSoup) only do steps 1-6. They get the raw HTML but never execute JavaScript. That means if a site loads its content via JavaScript (AJAX calls, React/Vue rendering), a simple scraper sees an empty page. You'll need a browser automation tool (Playwright, Selenium) for those sites.

## 1.2 HTTP Methods You'll Use

```text
METHOD   PURPOSE                    SCRAPING USE CASE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET      Retrieve a resource        Fetching pages, images, files
POST     Submit data                Login forms, search queries,
                                    API calls with parameters
HEAD     Get headers only           Check if page exists / size
                                    before downloading
PUT      Update a resource          Rarely used in scraping
DELETE   Remove a resource          Rarely used in scraping
OPTIONS  Check allowed methods      API reconnaissance
```

## 1.3 HTTP Headers — Your Bot's Identity Card

Every HTTP request includes headers that tell the server about the client. This is where most bots get caught.

```text
CRITICAL HEADERS FOR SCRAPING:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

User-Agent:     Identifies the browser/client. Default Python
                Requests UA is "python-requests/2.31.0" — this
                is an INSTANT red flag. Always set a real UA.

Accept:         Tells server what content types you want.
                Real browsers send complex Accept headers.
                Bots often omit this or send "*/*".

Accept-Language: Real browsers declare language preferences.
                Missing = bot indicator.

Accept-Encoding: Declares compression support (gzip, br).
                Real browsers always send this.

Referer:        Where the request came from. If you're hitting
                page 5 of results without ever visiting page 1,
                the missing Referer is suspicious.

Cookie:         Session tokens, auth tokens, tracking tokens.
                Essential for maintaining logged-in sessions.

Connection:     Usually "keep-alive" for real browsers.

Sec-Fetch-*:    Modern browsers send Sec-Fetch-Dest,
                Sec-Fetch-Mode, Sec-Fetch-Site headers.
                Missing = headless browser indicator.
```

**Purple team insight:** On the blue side, these headers are exactly what your WAF and log analysis should be checking. A request with `User-Agent: python-requests/2.28.1` hitting your login page 500 times in a minute? That's a scraper. A request claiming to be Chrome 124 but missing `Sec-Fetch-Dest`? That's a poorly configured headless browser.

## 1.4 HTTP Status Codes You'll Encounter

```text
CODE  MEANING              SCRAPING IMPLICATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
200   OK                   Success — you got the page
301   Moved Permanently    Follow the redirect (most libs do auto)
302   Found (temp redirect) Follow it, but the original URL is valid
403   Forbidden            You've been detected/blocked
404   Not Found            Page doesn't exist
429   Too Many Requests    Rate limited — slow down
500   Internal Server Error Server-side problem (not your fault)
503   Service Unavailable  Server overloaded or maintenance
```

**Key pattern:** If you're getting 200s and suddenly start getting 403s or 429s, you've been detected. Back off. Change your approach.

## 1.5 HTML Structure — What You're Parsing

HTML is a tree of nested elements. Every scraping task is about navigating this tree to find the data you want.

```html
<html>
  <head><title>Target Page</title></head>
  <body>
    <div id="content">
      <h1 class="title">Product Name</h1>
      <span class="price">$49.99</span>
      <div class="description">
        <p>This is the product description.</p>
      </div>
      <ul class="features">
        <li>Feature 1</li>
        <li>Feature 2</li>
      </ul>
    </div>
  </body>
</html>
```

**Selectors** are how you target specific elements:

```text
CSS SELECTORS (used by BeautifulSoup, Playwright):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
h1              Tag name (all h1 elements)
.price          Class name (class="price")
#content        ID (id="content")
div.description Tag + class combined
ul.features li  Nested: li elements inside ul.features
div > p         Direct child only (not deeply nested)
[href]          Has attribute
a[href*="login"] Attribute contains value

XPATH (used by Scrapy, lxml, Playwright):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
//h1                    All h1 elements
//span[@class='price']  span with class="price"
//div[@id='content']    div with id="content"
//ul/li                 li children of ul
//a[contains(@href, 'login')]  Link containing "login"
//text()                Extract text content
```

## 1.6 JavaScript-Rendered Content (The SPA Problem)

Modern web apps often load content dynamically using JavaScript frameworks (React, Vue, Angular). When you `GET` the page, the HTML body might look like this:

```html
<div id="root"></div>
<script src="/static/js/app.bundle.js"></script>
```

All the actual content gets injected by JavaScript AFTER the page loads. A simple HTTP request sees only the empty `<div>`. This is why browser automation tools (Playwright, Selenium) exist — they run an actual browser that executes JavaScript.

**How to tell if a site uses JS rendering:** Right-click → View Page Source. If the content you want isn't in the raw HTML, it's JavaScript-rendered. Compare with the browser's DevTools Elements tab, which shows the rendered DOM.

## 1.7 APIs — The Scraper's Best Friend

Before scraping HTML, check if the site has an API. Many sites load their data via API calls that return clean JSON — much easier to parse than HTML.

```text
HOW TO FIND HIDDEN APIs:
━━━━━━━━━━━━━━━━━━━━━━━
1. Open browser DevTools (F12)
2. Go to the Network tab
3. Filter by "XHR" or "Fetch"
4. Navigate the site normally
5. Watch for API calls returning JSON data
6. Right-click → Copy as cURL
7. Reproduce in Python with requests

EXAMPLE — what you might find:
  GET https://api.example.com/products?page=1&limit=20
  Response: {"products": [{"name": "Widget", "price": 49.99}, ...]}
```

This is often dramatically easier than parsing HTML, and it's how most professional scrapers work. The HTML is the presentation layer — the API is the data layer.

---

## Section 2: The Python Scraping Stack — Choosing Your Tools

There is no single "best" scraping tool. The right tool depends on the target. This section maps the entire landscape so you can match tools to targets.

## 2.1 The Decision Tree

```text
START HERE: What kind of site are you scraping?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Is the content in the raw HTML source?
  │
  ├─ YES → Is it a simple, one-page scrape?
  │         ├─ YES → Requests + BeautifulSoup
  │         └─ NO (multi-page crawl) → Scrapy
  │
  └─ NO (JavaScript-rendered) →
      │
      ├─ Does the site have a JSON API? (check DevTools Network tab)
      │   └─ YES → Requests (call the API directly)
      │
      └─ NO (must render JS) →
          │
          ├─ Need to interact with the page? (click, scroll, fill forms)
          │   └─ YES → Playwright (preferred) or Selenium
          │
          └─ Just need rendered HTML?
              └─ Playwright in minimal mode
```

## 2.2 Tier 1: Requests + BeautifulSoup (The Foundation)

This is where you start. These two libraries handle 60-70% of all scraping tasks.

```text
REQUESTS                          BEAUTIFULSOUP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
What:  HTTP client library        What:  HTML/XML parser
Does:  Sends HTTP requests,       Does:  Navigates HTML tree,
       receives responses                finds elements by
                                         tag/class/id/CSS
Can't: Execute JavaScript         Can't: Fetch web pages
       Render pages                      (it only parses)
       Handle browser cookies
       automatically

TOGETHER: Requests fetches the page, BeautifulSoup parses it.
```

**Install:**

```bash
pip install requests beautifulsoup4 lxml
```

**Your First Scraper:**

```python
import requests
from bs4 import BeautifulSoup

# Step 1: Fetch the page
url = "https://quotes.toscrape.com/"
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                  "AppleWebKit/537.36 (KHTML, like Gecko) "
                  "Chrome/124.0.0.0 Safari/537.36"
}
response = requests.get(url, headers=headers)

# Step 2: Check we got a good response
if response.status_code != 200:
    print(f"Failed: {response.status_code}")
    exit()

# Step 3: Parse the HTML
soup = BeautifulSoup(response.text, "lxml")

# Step 4: Extract data
quotes = soup.find_all("div", class_="quote")
for quote in quotes:
    text = quote.find("span", class_="text").get_text()
    author = quote.find("small", class_="author").get_text()
    print(f"{author}: {text}")
```

**Key BeautifulSoup Methods:**

```python
# Finding elements
soup.find("tag")                   # First match
soup.find_all("tag")               # All matches
soup.find("tag", class_="name")    # By class
soup.find("tag", id="name")        # By ID
soup.find("tag", attrs={"data-id": "123"})  # By attribute
soup.select("css > selector")      # CSS selector (returns list)
soup.select_one("css > selector")  # CSS selector (first match)

# Extracting data
element.get_text()                 # Text content
element.get_text(strip=True)       # Text, whitespace stripped
element["href"]                    # Attribute value
element.get("href", "")            # Attribute with default
element.find_parent("div")         # Navigate up the tree
element.find_next_sibling("p")     # Next sibling element
```

## 2.3 Tier 2: Playwright (Browser Automation)

When the site requires JavaScript rendering or interactive behavior (login forms, infinite scroll, button clicks), you need a real browser.

Playwright is the modern standard — faster, more reliable, and less detectable than Selenium.

**Install:**

```bash
pip install playwright
playwright install chromium
```

**Basic Playwright Scraper:**

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    # Launch a headless browser
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()

    # Navigate to the page (waits for load automatically)
    page.goto("https://quotes.toscrape.com/js/")

    # Wait for content to render
    page.wait_for_selector(".quote")

    # Extract data from the rendered page
    quotes = page.query_selector_all(".quote")
    for quote in quotes:
        text = quote.query_selector(".text").inner_text()
        author = quote.query_selector(".author").inner_text()
        print(f"{author}: {text}")

    browser.close()
```

**Key Playwright Actions:**

```python
# Navigation
page.goto(url)                          # Navigate to URL
page.go_back()                          # Browser back
page.reload()                           # Refresh page

# Waiting (critical for dynamic content)
page.wait_for_selector("css-selector")  # Wait for element
page.wait_for_load_state("networkidle") # Wait for network quiet
page.wait_for_timeout(2000)             # Wait 2 seconds

# Interaction
page.click("css-selector")             # Click element
page.fill("input#username", "myuser")  # Type into input
page.select_option("select#country", "US")  # Dropdown
page.keyboard.press("Enter")           # Keyboard
page.mouse.wheel(0, 500)               # Scroll down

# Extraction
page.inner_text("selector")            # Text content
page.get_attribute("selector", "href") # Attribute
page.content()                         # Full rendered HTML
page.query_selector_all("selector")    # Multiple elements

# Screenshots (useful for debugging & recon)
page.screenshot(path="screenshot.png")
page.screenshot(path="full.png", full_page=True)

# Network interception (see hidden APIs)
page.on("response", lambda resp:
    print(resp.url) if "api" in resp.url else None)
```

## 2.4 Tier 3: Scrapy (Industrial-Scale Crawling)

Scrapy is a full framework for large-scale, multi-page crawling. It's overkill for simple scrapes but essential when you need to crawl thousands of pages with structured pipelines.

```text
WHEN TO USE SCRAPY:
━━━━━━━━━━━━━━━━━━
✦ Crawling an entire site (follow all links)
✦ Scraping thousands of pages with the same structure
✦ Need built-in rate limiting, retries, and politeness
✦ Want structured data pipelines (clean → validate → store)
✦ Building a reusable, maintainable spider

WHEN NOT TO USE SCRAPY:
━━━━━━━━━━━━━━━━━━━━━━━
✦ Quick one-off scrapes (use Requests + BS4)
✦ JavaScript-rendered content (Scrapy doesn't render JS
  natively — needs Playwright middleware)
✦ Learning your first scraper (steep learning curve)
```

**Install and Create a Project:**

```bash
pip install scrapy
scrapy startproject recon_spider
cd recon_spider
scrapy genspider target_spider example.com
```

**Basic Scrapy Spider:**

```python
# recon_spider/spiders/target_spider.py
import scrapy

class TargetSpider(scrapy.Spider):
    name = "target_spider"
    start_urls = ["https://quotes.toscrape.com/"]

    # Politeness settings (in settings.py)
    custom_settings = {
        "DOWNLOAD_DELAY": 2,           # 2 sec between requests
        "CONCURRENT_REQUESTS": 1,      # One at a time
        "ROBOTSTXT_OBEY": True,        # Respect robots.txt
        "USER_AGENT": "Mozilla/5.0 ... Chrome/124.0",
    }

    def parse(self, response):
        for quote in response.css("div.quote"):
            yield {
                "text": quote.css("span.text::text").get(),
                "author": quote.css("small.author::text").get(),
                "tags": quote.css("a.tag::text").getall(),
            }

        # Follow pagination
        next_page = response.css("li.next a::attr(href)").get()
        if next_page:
            yield response.follow(next_page, self.parse)
```

**Run:** `scrapy crawl target_spider -o output.json`

## 2.5 Tool Comparison Matrix

```text
                    Requests+BS4   Playwright    Scrapy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Learning curve      Low            Medium        High
JavaScript support  No             Yes           No*
Speed               Fast           Slow          Fast
Memory usage        Low            High          Medium
Anti-bot evasion    Manual         Better**      Manual
Scale               Small-med      Small-med     Large
Built-in crawling   No             No            Yes
Data pipelines      No             No            Yes
Async support       No***          Yes           Yes
Best for            Quick scrapes  JS sites      Crawling

*  Scrapy can add Playwright via scrapy-playwright middleware
** Playwright runs a real browser, harder to fingerprint
*** httpx library adds async to the Requests workflow
```

---

## Section 3: Practical Scraping Patterns — From Simple to Advanced

This section provides working patterns for real-world scraping scenarios, progressing from basic to complex. Each pattern includes the red team use and the blue team detection angle.

## 3.1 Pattern: Paginated Data Collection

Most sites spread data across multiple pages. You need to follow pagination links or increment page numbers.

```python
import requests
from bs4 import BeautifulSoup
import time
import random

BASE_URL = "https://quotes.toscrape.com/page/{}/"
headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
           "AppleWebKit/537.36 Chrome/124.0 Safari/537.36"}

all_quotes = []
for page_num in range(1, 11):
    url = BASE_URL.format(page_num)
    response = requests.get(url, headers=headers)

    if response.status_code != 200:
        print(f"Stopped at page {page_num}: {response.status_code}")
        break

    soup = BeautifulSoup(response.text, "lxml")
    quotes = soup.find_all("div", class_="quote")

    if not quotes:
        break  # No more content

    for q in quotes:
        all_quotes.append({
            "text": q.find("span", class_="text").get_text(),
            "author": q.find("small", class_="author").get_text(),
        })

    # CRITICAL: Random delay between requests
    delay = random.uniform(1.5, 4.0)
    print(f"Page {page_num}: {len(quotes)} quotes. "
          f"Sleeping {delay:.1f}s...")
    time.sleep(delay)

print(f"Total: {len(all_quotes)} quotes collected.")
```

```text
🔴 RED TEAM USE:
  Enumerate employee profiles, job postings, product
  catalogs, or any paginated data source.

🔵 BLUE TEAM DETECTION:
  Sequential page access pattern (page/1, page/2, page/3...)
  from same IP with regular intervals. Look for:
  - Same User-Agent across all requests
  - No Referer headers between pages
  - No CSS/JS/image requests (only HTML)
  - Access pattern faster than human reading speed
  - Requests at 2-4 second intervals (suspiciously regular
    even with randomization)
```

## 3.2 Pattern: Form Submission and Login

Many targets require authentication. This means submitting login forms and maintaining session cookies.

```python
import requests
from bs4 import BeautifulSoup

# Use a Session to persist cookies across requests
session = requests.Session()
session.headers.update({
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                  "AppleWebKit/537.36 Chrome/124.0 Safari/537.36"
})

# Step 1: Get the login page (to capture CSRF token)
login_page = session.get("https://quotes.toscrape.com/login")
soup = BeautifulSoup(login_page.text, "lxml")
csrf_token = soup.find("input", {"name": "csrf_token"})["value"]

# Step 2: Submit login form
login_data = {
    "csrf_token": csrf_token,
    "username": "testuser",
    "password": "testpass",
}
response = session.post(
    "https://quotes.toscrape.com/login",
    data=login_data
)

# Step 3: Now use the authenticated session
protected_page = session.get("https://quotes.toscrape.com/")
# session.cookies now contains the auth cookie
```

```text
🔴 RED TEAM USE:
  Credential stuffing attacks, authenticated OSINT collection,
  accessing member-only content during engagements.

🔵 BLUE TEAM DETECTION:
  - Multiple login attempts from same IP with different creds
    (credential stuffing)
  - Login followed immediately by rapid page access
    (no human browse pattern after login)
  - Missing CSRF token requests before POST (lazy bots)
  - Event ID correlation: failed logins → successful login →
    rapid automated browsing
```

## 3.3 Pattern: API Interception with Playwright

Instead of parsing HTML, intercept the underlying API calls that the JavaScript makes.

```python
from playwright.sync_api import sync_playwright
import json

collected_data = []

def handle_response(response):
    """Capture API responses containing JSON data."""
    if "api" in response.url and response.status == 200:
        try:
            data = response.json()
            collected_data.append({
                "url": response.url,
                "data": data
            })
            print(f"Captured: {response.url}")
        except Exception:
            pass  # Not JSON, skip

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()

    # Register response interceptor
    page.on("response", handle_response)

    # Navigate and interact normally
    page.goto("https://example-spa.com")
    page.wait_for_load_state("networkidle")

    # Scroll to trigger lazy loading
    for _ in range(5):
        page.mouse.wheel(0, 500)
        page.wait_for_timeout(1000)

    browser.close()

# Now you have clean API data
for item in collected_data:
    print(json.dumps(item, indent=2))
```

```text
🔴 RED TEAM USE:
  Discover undocumented APIs, capture API schemas, find
  authentication tokens, map the full API surface area
  of a target application.

🔵 BLUE TEAM DETECTION:
  - API calls without preceding page navigation
  - API rate patterns that don't match normal user behavior
  - Missing correlation between page views and API calls
  - Unusual API endpoints being accessed in sequence
```

## 3.4 Pattern: Data Export and Storage

Collected data needs to go somewhere useful.

```python
import json
import csv
from datetime import datetime

data = [
    {"name": "Target Corp", "email": "info@target.com", "role": "Admin"},
    {"name": "Jane Doe", "email": "jane@target.com", "role": "Engineer"},
]

# JSON (preserves structure, good for nested data)
with open("output.json", "w") as f:
    json.dump(data, f, indent=2)

# CSV (good for spreadsheets and data analysis)
with open("output.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=data[0].keys())
    writer.writeheader()
    writer.writerows(data)

# Timestamped filenames (for recurring scrapes)
timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
filename = f"scrape_results_{timestamp}.json"
```

## 3.5 Pattern: Error Handling and Resilience

Production scrapers must handle failures gracefully. Networks drop, servers error, sites change structure.

```python
import requests
from requests.exceptions import (
    ConnectionError, Timeout, HTTPError, TooManyRedirects
)
import time

def fetch_with_retry(url, headers, max_retries=3, backoff=2):
    """Fetch a URL with exponential backoff on failure."""
    for attempt in range(max_retries):
        try:
            response = requests.get(
                url, headers=headers, timeout=10
            )
            response.raise_for_status()  # Raise on 4xx/5xx
            return response

        except Timeout:
            print(f"  Timeout on attempt {attempt + 1}")
        except ConnectionError:
            print(f"  Connection failed on attempt {attempt + 1}")
        except HTTPError as e:
            if e.response.status_code == 429:
                wait = backoff ** (attempt + 2)
                print(f"  Rate limited. Waiting {wait}s...")
                time.sleep(wait)
            elif e.response.status_code == 403:
                print(f"  Blocked (403). Aborting.")
                return None
            else:
                print(f"  HTTP {e.response.status_code}")

        # Exponential backoff
        wait = backoff ** attempt
        time.sleep(wait)

    print(f"  Failed after {max_retries} retries: {url}")
    return None
```

---

## Section 4: OSINT Reconnaissance — Purple Team Scraping Use Cases

This is where scraping meets security operations. These are the use cases that justify learning this skill as part of a purple team mission.

## 4.1 Attack Surface Mapping

Scrape publicly available information about your own organization to find what an adversary would find.

```text
WHAT TO SCRAPE (on your own org, with authorization):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ Job postings — reveal internal tech stack, tools, and
  team structures. "Must know Splunk, CrowdStrike, and
  Palo Alto" tells an adversary your entire security stack.

✦ Employee directories / LinkedIn — org chart, reporting
  structure, who to target for phishing.

✦ GitHub/GitLab repos — leaked credentials, internal IPs,
  API keys in commit history, infrastructure documentation.

✦ Pastebin / paste sites — leaked credentials, configs,
  code snippets referencing internal systems.

✦ DNS records / subdomains — map the external footprint.
  Combine with tools like subfinder, amass, dnsx.

✦ SSL/TLS certificates — Certificate Transparency logs
  reveal subdomains (crt.sh is scrapable).

✦ Social media — employee posts revealing office photos
  (badge designs, screen contents), travel patterns,
  conference attendance.

✦ Shodan / Censys — exposed services, banners, versions.
  Both have APIs; scraping the web UI is unnecessary.
```

## 4.2 Credential Exposure Monitoring

Build a scraper that monitors paste sites and breach databases for your organization's domains.

```python
# EXAMPLE: Check if emails from your domain appear in breach data
# This uses the Have I Been Pwned API (requires API key)
import requests
import time

API_KEY = "your-hibp-api-key"
DOMAIN_EMAILS = [
    "admin@yourcompany.com",
    "ciso@yourcompany.com",
    "helpdesk@yourcompany.com",
]

headers = {
    "hibp-api-key": API_KEY,
    "User-Agent": "PurpleTeam-CredMonitor/1.0",
}

for email in DOMAIN_EMAILS:
    url = f"https://haveibeenpwned.com/api/v3/breachedaccount/{email}"
    response = requests.get(url, headers=headers)

    if response.status_code == 200:
        breaches = response.json()
        print(f"⚠️  {email}: found in {len(breaches)} breaches")
        for b in breaches:
            print(f"    - {b['Name']} ({b['BreachDate']})")
    elif response.status_code == 404:
        print(f"✅ {email}: no breaches found")
    else:
        print(f"❓ {email}: status {response.status_code}")

    time.sleep(1.5)  # HIBP rate limit: 1 req per 1.5 sec
```

## 4.3 Threat Intelligence Feed Collection

Scrape publicly available threat intel feeds and IOC sources for your SIEM/SOAR ingestion.

```text
SCRAPABLE THREAT INTEL SOURCES:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ abuse.ch (URLhaus, MalwareBazaar, ThreatFox)
  — Structured CSV/JSON feeds. Direct download, no scraping needed.

✦ AlienVault OTX — API-based. Free tier available.

✦ VirusTotal — API-based. Free tier: 4 requests/min.

✦ Shodan — API-based. Free tier: 100 results/month.

✦ crt.sh — Certificate transparency. Scrapable or API.

✦ CVE databases — NIST NVD has a REST API.

✦ Vendor advisories — Many are HTML pages that need scraping
  (Microsoft MSRC, Cisco PSIRT, etc.)
```

**Purple team value:** Build automated pipelines that scrape these sources, normalize the IOCs, and feed them into your SIEM. This is how you build detection content from publicly available intelligence.

## 4.4 Phishing Infrastructure Detection

Scrape Certificate Transparency logs to find domains that impersonate your organization.

```python
import requests
import json

def check_ct_logs(domain):
    """Search crt.sh for certificates matching a domain pattern."""
    url = f"https://crt.sh/?q=%25{domain}%25&output=json"
    response = requests.get(url, timeout=30)

    if response.status_code != 200:
        print(f"Failed to query crt.sh: {response.status_code}")
        return []

    certs = response.json()
    suspicious = []

    for cert in certs:
        name = cert.get("name_value", "")
        # Look for lookalikes (typosquatting, homoglyphs)
        if domain not in name:
            # This cert mentions our domain but isn't ours
            suspicious.append({
                "common_name": name,
                "issuer": cert.get("issuer_name", ""),
                "not_before": cert.get("not_before", ""),
                "not_after": cert.get("not_after", ""),
            })

    return suspicious

# Check for certificates impersonating your domain
results = check_ct_logs("yourcompany.com")
for r in results:
    print(f"⚠️  {r['common_name']} "
          f"(issued: {r['not_before']})")
```

---

## Section 5: Anti-Bot Detection — Know What You're Up Against

To build effective scrapers AND effective defenses, you need to understand the multi-layered detection systems that modern sites deploy. This is the arms race.

## 5.1 Detection Layers (From Basic to Advanced)

```text
LAYER 1: REQUEST-LEVEL ANALYSIS (easiest to bypass)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ User-Agent string inspection
  — Default library UAs ("python-requests", "curl")
  — Outdated browser versions
  — Impossible OS/browser combinations

✦ HTTP header validation
  — Missing Accept, Accept-Language, Accept-Encoding
  — Wrong header order (browsers send headers in
    specific orders; Python sends alphabetically)
  — Missing Sec-Fetch-* headers

✦ Rate limiting
  — Too many requests per minute from one IP
  — Fixed intervals between requests (no human variance)

LAYER 2: IP REPUTATION (moderate difficulty)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ Datacenter IP detection (AWS, GCP, Azure, Hetzner)
✦ Known proxy/VPN IP ranges
✦ Geolocation anomalies (IP in Germany, Accept-Language
  set to en-US)
✦ Historical reputation (IP previously flagged for abuse)

LAYER 3: BROWSER FINGERPRINTING (harder to bypass)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ navigator.webdriver property (True for automation tools)
✦ Canvas fingerprinting (how the browser renders graphics)
✦ WebGL fingerprinting (GPU rendering characteristics)
✦ Audio context fingerprinting
✦ Plugin and extension enumeration
✦ Screen resolution, color depth, timezone
✦ Font enumeration
✦ JavaScript execution behavior

LAYER 4: BEHAVIORAL ANALYSIS (hardest to bypass)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ Mouse movement patterns (bots don't move the mouse)
✦ Scroll behavior (bots don't scroll naturally)
✦ Click patterns (bots click instantly, humans hesitate)
✦ Navigation patterns (bots skip the homepage)
✦ Session duration (bots are too fast)
✦ Resource loading (bots skip CSS/images/fonts)

LAYER 5: TLS FINGERPRINTING (very hard to bypass)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ TLS ClientHello message fingerprint (JA3/JA4)
✦ Cipher suite order
✦ Supported extensions
✦ Key share groups
✦ Python's TLS fingerprint is DIFFERENT from Chrome's
  — This alone can identify your scraper

LAYER 6: CAPTCHA / CHALLENGE (active defense)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✦ reCAPTCHA v2/v3 (Google)
✦ Cloudflare Turnstile
✦ hCaptcha
✦ Custom JavaScript challenges
✦ Proof-of-work challenges
```

## 5.2 Major Anti-Bot Providers You'll Encounter

```text
PROVIDER       DIFFICULTY    WHAT THEY DO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Cloudflare     Medium-High   JS challenge, Turnstile CAPTCHA,
                             bot score, WAF rules, IP rep
Akamai         High          Bot Manager, fingerprinting,
                             behavioral analysis
DataDome       High          Real-time ML detection, device
                             fingerprinting, behavioral
PerimeterX     High          Human Challenge, behavioral
(now HUMAN)                  biometrics, code integrity
Imperva        Medium-High   Advanced Bot Protection,
(Incapsula)                  client classification
AWS WAF        Medium        Bot Control managed rules,
                             ML-based detection, rate limiting
Sucuri         Medium        WAF + CDN, basic bot detection
```

## 5.3 Evasion Techniques (Red Team Perspective)

Understanding these helps you both attack and defend. Every evasion technique has a corresponding detection opportunity.

```text
TECHNIQUE                WHAT IT DOES              DETECTION ANGLE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
User-Agent rotation      Cycles through real UAs    Same IP with
                                                    changing UAs
                                                    is suspicious

Proxy rotation           Changes IP per request     Residential IPs
                         (datacenter, residential,  harder to detect;
                         mobile)                    datacenter = easy

Random delays            Adds jitter between        Statistical analysis
                         requests (1-10s random)    of request timing

Header spoofing          Sends full browser-like    Header order still
                         header sets                differs from real
                                                    browsers

Headless browser         Runs real Chrome/Firefox   navigator.webdriver,
stealth plugins          with automation patched    missing plugins,
                         (undetected-chromedriver,  canvas anomalies
                         playwright-stealth)

Session rotation         New browser profile per    Cookie/session
                         N requests                 discontinuity

Referer chain            Simulates natural nav      Referer chain that
                         (homepage → category →     doesn't match actual
                         product)                   site structure

Mouse/scroll sim         Fake human interaction     Too-perfect movement
                         patterns                   patterns, missing
                                                    micro-corrections
```

## 5.4 Blue Team Detection Playbook

This is the defensive half. When you're wearing the blue hat, here's what to look for in your logs.

```text
INDICATOR                          LOG SOURCE           DETECTION RULE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Known bot User-Agents              Web server logs      String match:
  python-requests, curl,                                python-requests,
  wget, scrapy, HttpClient                              Go-http-client,
                                                        etc.

No CSS/JS/image requests           Web server logs      HTML requests
  (only HTML fetched)                                   without associated
                                                        asset requests

Sequential URL patterns            Web server logs      Regex: page numbers
  /page/1, /page/2, /page/3                            incrementing
                                                        sequentially

High request rate from             WAF / rate limiter   Threshold: >30
  single IP                                             req/min to same
                                                        endpoint

Requests outside business          Web server logs      Time-based filter:
  hours (3 AM local time)                               requests at unusual
                                                        hours with no prior
                                                        session

Missing Referer on deep            Web server logs      Deep URL accessed
  pages                                                 without homepage or
                                                        nav page Referer

TLS fingerprint mismatch           WAF / TLS logs       JA3/JA4 hash doesn't
                                                        match claimed UA
                                                        (Python TLS ≠ Chrome)

HeadlessChrome in UA               Web server logs      String: HeadlessChrome

Identical timing between           Web server logs      Standard deviation
  requests                                              of inter-request
                                                        time is near zero

Geographic impossibility           GeoIP + logs         Same session from
                                                        two countries in
                                                        seconds
```

---

## Section 6: Legal, Ethical, and Operational Boundaries

This section is non-negotiable. Skip it and you risk your job, your clearance, and your freedom.

## 6.1 The Legal Landscape (US Focus)

```text
LAW / PRECEDENT              WHAT IT MEANS FOR YOU
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CFAA (Computer Fraud         The big one. "Unauthorized access"
and Abuse Act)               to a computer system. Key question:
                             did you have permission?

Van Buren v. US (2021)       Supreme Court NARROWED the CFAA.
                             Accessing data you're authorized to
                             see for an unauthorized purpose is
                             NOT a CFAA violation. This helped
                             define "exceeds authorized access."

hiQ v. LinkedIn (2022)       Ninth Circuit: scraping PUBLIC data
                             does not violate the CFAA. Landmark
                             case for scraping legality. BUT:
                             this only applies to truly public
                             data. Behind a login = different.

Meta v. Bright Data (2024)   Scraping content behind contractual
                             restrictions may be a breach, even
                             if publicly visible. ToS matters.

DMCA Section 1201            Anti-circumvention. Bypassing
                             technical protection measures
                             (CAPTCHAs, login walls, anti-bot)
                             could trigger DMCA liability. X Corp
                             has alleged DMCA claims against
                             scrapers for proxy use.

GDPR (EU)                    Scraping personal data of EU
                             residents requires lawful basis.
                             Fines up to €20M or 4% global
                             revenue. Even public data can be
                             "personal data" under GDPR.

CCPA (California)            Similar to GDPR for California
                             residents. Applies if you scrape
                             PII.

robots.txt                   NOT legally binding in most
                             jurisdictions. BUT: ignoring it
                             undermines "good faith" arguments.
                             In Texas, ignoring robots.txt could
                             support a DMCA claim.

ToS (Terms of Service)       Violating ToS doesn't auto-trigger
                             CFAA liability, but CAN result in
                             civil lawsuits under contract law.
```

## 6.2 The Purple Team Rules of Engagement

```text
ALWAYS:
━━━━━━
✦ Get written authorization before scraping ANY target
  that isn't your own infrastructure or a public training
  site (like quotes.toscrape.com, books.toscrape.com,
  httpbin.org)

✦ Scrape your OWN organization's external footprint
  (with approval from leadership and legal)

✦ Respect robots.txt even when you don't have to — it
  demonstrates good faith

✦ Rate limit your requests — never hammer a target

✦ Document everything: what you scraped, when, why,
  what you found, what you did with the data

✦ Delete scraped data when the engagement ends

✦ Use a descriptive User-Agent for authorized testing:
  "CompanyName-PurpleTeam/1.0 (authorized-assessment)"

NEVER:
━━━━━━
✦ Scrape behind login walls without explicit authorization

✦ Scrape PII (names, emails, SSNs, etc.) and store it
  without a legitimate need and data handling plan

✦ Bypass anti-bot measures on targets you don't own
  (this can trigger CFAA / DMCA liability)

✦ Scrape sites so aggressively that you cause a DoS
  condition (even accidentally)

✦ Use scraped data for purposes beyond the authorized
  scope of the engagement

✦ Share scraped data outside the authorized team
```

## 6.3 Training and Practice Sites

These sites are specifically built for scraping practice. Use them freely.

```text
SITE                                  WHAT IT OFFERS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
quotes.toscrape.com                   Static quotes, pagination,
                                      login form, tags. Perfect
                                      for Requests + BS4.

quotes.toscrape.com/js/               Same content but JavaScript-
                                      rendered. Perfect for
                                      Playwright practice.

books.toscrape.com                    Fake bookstore with catalog,
                                      categories, ratings, pricing.

httpbin.org                           HTTP request/response testing.
                                      Echoes back your headers,
                                      IP, cookies.

webscraper.io/test-sites              Multiple test sites: e-commerce,
                                      tables, AJAX loading.

scrapethissite.com                    Tutorials + practice sandboxes
                                      with increasing difficulty.

toscrape.com                          Hub for all Scrapy practice
                                      sites.
```

---

## Section 7: Building Your Lab Environment

## 7.1 Python Environment Setup

```bash
# Create a dedicated virtual environment
python3 -m venv ~/scraping-lab
source ~/scraping-lab/bin/activate

# Core libraries
pip install requests beautifulsoup4 lxml

# Browser automation
pip install playwright
playwright install chromium

# Framework (when you're ready)
pip install scrapy

# Utilities
pip install httpx          # Async HTTP client
pip install parsel         # Scrapy's selector library standalone
pip install fake-useragent # Rotate User-Agents
pip install rich           # Pretty terminal output
pip install pandas         # Data analysis
```

## 7.2 Project Structure

```text
scraping-lab/
├── venv/                   # Virtual environment
├── scrapers/
│   ├── basic/              # Requests + BS4 scrapers
│   ├── browser/            # Playwright scrapers
│   ├── crawlers/           # Scrapy projects
│   └── osint/              # Purple team OSINT tools
├── output/
│   ├── json/               # JSON exports
│   ├── csv/                # CSV exports
│   └── screenshots/        # Playwright screenshots
├── utils/
│   ├── headers.py          # Realistic header sets
│   ├── delays.py           # Rate limiting helpers
│   └── storage.py          # Data export functions
├── detection/
│   ├── log_analyzer.py     # Blue team log analysis
│   └── sample_logs/        # Sample web server logs
└── README.md               # Document everything
```

## 7.3 Homelab Detection Practice

Since you have a Proxmox environment, you can build both sides of the purple team exercise:

```text
PURPLE TEAM HOMELAB SETUP:
━━━━━━━━━━━━━━━━━━━━━━━━━
1. Deploy a web server VM (Nginx or Apache on RHEL)
   with a simple site to scrape

2. Enable detailed access logging:
   log_format detailed '$remote_addr - $remote_user '
     '[$time_local] "$request" $status $body_bytes_sent '
     '"$http_referer" "$http_user_agent" '
     '$request_time $upstream_response_time';

3. Run your scraper against it

4. Analyze the logs — can you distinguish your bot from
   a real browser? What gives it away?

5. Improve the scraper to be less detectable

6. Improve the detection rules to catch the improved scraper

7. Iterate. This IS the purple team cycle.

OPTIONAL ADDITIONS:
✦ ModSecurity WAF in front of the web server
✦ Suricata/Zeek on the network segment
✦ Forward logs to a SIEM (Wazuh, ELK, Splunk Free)
✦ Write detection rules for each scraper technique
```

---

## Section 8: The Learning Path — 8-Week Progression

## 8.1 Phase 1: Foundations (Weeks 1-2)

```text
WEEK 1: HTTP and HTML Fundamentals
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Read Section 1 of this codex (HTTP, HTML, selectors)
[ ] Install Python venv and core libraries (Section 7.1)
[ ] Use browser DevTools: inspect elements, view Network
    tab, view page source on 5 different sites
[ ] Use httpbin.org to understand request/response headers
[ ] Write your first Requests + BS4 scraper against
    quotes.toscrape.com (Section 2.2)
[ ] Export results to JSON and CSV (Section 3.4)
[ ] Read Section 6 (Legal/Ethical) — this is not optional

WEEK 2: Core Scraping Patterns
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Build a paginated scraper for books.toscrape.com
    (collect all 1000 books across 50 pages)
[ ] Implement random delays and proper error handling
    (Sections 3.1 and 3.5)
[ ] Practice CSS selectors and XPath on 3 different
    practice sites
[ ] Learn to use requests.Session() for cookie persistence
[ ] Build a form-submitting scraper (Section 3.2)
[ ] Start documenting what you build (README per project)
```

## 8.2 Phase 2: Browser Automation (Weeks 3-4)

```text
WEEK 3: Playwright Fundamentals
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Install Playwright and chromium (Section 2.3)
[ ] Scrape quotes.toscrape.com/js/ (JS-rendered version)
[ ] Learn page.wait_for_selector() and networkidle
[ ] Practice clicking, filling forms, scrolling
[ ] Take screenshots during scraping (debugging tool)
[ ] Intercept API calls using page.on("response")
    (Section 3.3)

WEEK 4: Advanced Browser Automation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Find a SPA practice site — use DevTools to discover
    hidden APIs, then scrape the API directly
[ ] Handle infinite scroll (scroll → wait → check for
    new content → repeat)
[ ] Handle popup modals and cookie banners
[ ] Run Playwright in headed mode (headless=False) to
    debug visually
[ ] Compare headless vs headed fingerprints using
    bot detection test sites (bot.sannysoft.com,
    browserleaks.com)
```

## 8.3 Phase 3: OSINT and Purple Team (Weeks 5-6)

```text
WEEK 5: OSINT Scraping Tools
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Build a crt.sh certificate transparency monitor
    for a domain you own (Section 4.4)
[ ] Build a job posting scraper for tech stack
    reconnaissance (Section 4.1)
[ ] Use the HIBP API to check breach exposure for
    test email addresses (Section 4.2)
[ ] Explore abuse.ch threat feeds — download and parse
    URLhaus CSV data

WEEK 6: Detection and Defense
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Deploy a web server on your homelab (Section 7.3)
[ ] Run your scrapers against it and collect access logs
[ ] Write grep/awk one-liners to identify bot traffic
[ ] Study the anti-bot detection layers (Section 5.1)
[ ] Write 5 detection rules for your own scraper's
    traffic patterns
[ ] Read the blue team detection playbook (Section 5.4)
    and map each indicator to your log analysis
```

## 8.4 Phase 4: Advanced Topics (Weeks 7-8)

```text
WEEK 7: Scrapy and Scale
━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Build a Scrapy spider for a practice site (Section 2.4)
[ ] Configure politeness settings (delay, concurrent reqs,
    robots.txt compliance)
[ ] Use Scrapy pipelines to clean and store data
[ ] Handle pagination with response.follow()
[ ] Export to multiple formats (JSON, CSV, JSONL)

WEEK 8: Capstone — Full Purple Team Exercise
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Pick a scraping target (practice site or your own
    authorized infrastructure)
[ ] Build a comprehensive scraper that:
    - Handles pagination
    - Manages sessions/cookies
    - Implements rate limiting
    - Exports structured data
    - Handles errors gracefully
[ ] Run it against your homelab web server
[ ] Analyze the logs from the blue team side
[ ] Write a detection rule set (WAF rules, SIEM alert)
    that would catch your scraper
[ ] Document the full exercise: methodology, findings,
    red team TTPs, blue team detections
[ ] Present findings to your battle buddy
```

---

## Section 9: Quick Reference Cards

## 9.1 Requests Cheat Sheet

```python
import requests

# GET request
r = requests.get(url, headers=h, params={"key": "val"}, timeout=10)

# POST request (form data)
r = requests.post(url, data={"user": "x", "pass": "y"})

# POST request (JSON)
r = requests.post(url, json={"key": "value"})

# Session (persists cookies)
s = requests.Session()
s.headers.update({"User-Agent": "..."})
s.get(url)  # cookies carry forward

# Response attributes
r.status_code      # 200, 403, 429, etc.
r.text             # Response body as string
r.content          # Response body as bytes
r.json()           # Parse JSON response
r.headers          # Response headers (dict)
r.cookies          # Response cookies
r.url              # Final URL (after redirects)
r.elapsed          # Time for response
r.raise_for_status()  # Raise exception on 4xx/5xx
```

## 9.2 BeautifulSoup Cheat Sheet

```python
from bs4 import BeautifulSoup

soup = BeautifulSoup(html, "lxml")

# Find elements
soup.find("tag")                        # First match
soup.find_all("tag")                    # All matches
soup.find("tag", class_="name")         # By class
soup.find("tag", id="name")             # By ID
soup.find("tag", attrs={"data-x": "y"}) # By attribute
soup.select("css > selector")           # CSS (list)
soup.select_one("css > selector")       # CSS (first)

# Extract data
el.get_text(strip=True)                 # Text content
el["href"]                              # Attribute
el.get("href", "default")              # Attr with default
el.find_parent("div")                   # Navigate up
el.find_next_sibling("p")              # Next sibling
el.find_all("a", href=True)            # Elements with attr

# Check existence
if soup.find("div", class_="error"):
    print("Error element found")
```

## 9.3 Playwright Cheat Sheet

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    ctx = browser.new_context(
        user_agent="Mozilla/5.0 ...",
        viewport={"width": 1920, "height": 1080},
    )
    page = ctx.new_page()

    # Navigate
    page.goto(url, wait_until="networkidle")
    page.wait_for_selector("selector")

    # Interact
    page.click("selector")
    page.fill("input", "text")
    page.keyboard.press("Enter")
    page.mouse.wheel(0, 500)

    # Extract
    text = page.inner_text("selector")
    attr = page.get_attribute("selector", "href")
    html = page.content()
    els = page.query_selector_all("selector")

    # Debug
    page.screenshot(path="debug.png")
    page.pause()  # Opens inspector (headed mode only)

    browser.close()
```

## 9.4 One-Liner Log Analysis (Blue Team)

```bash
# Top 20 User-Agents hitting your server
awk -F'"' '{print $6}' access.log | sort | uniq -c | sort -rn | head -20

# Requests per IP (find high-volume scrapers)
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -20

# Find python-requests / curl / wget UAs
grep -i 'python-requests\|curl\|wget\|scrapy\|go-http' access.log

# Requests per minute from a specific IP
grep "192.168.1.100" access.log | awk '{print $4}' | \
  cut -d: -f1-3 | uniq -c | sort -rn

# Pages hit without CSS/JS/image requests (bot indicator)
awk '$7 ~ /\.html|\/[^.]*$/ {print $1}' access.log | \
  sort | uniq -c | sort -rn

# Find sequential page access patterns
grep 'page/[0-9]' access.log | awk '{print $1, $7}' | sort

# Requests at unusual hours (midnight to 5am)
awk -F'[/: ]' '$5 >= 0 && $5 <= 5 {print}' access.log | head -20
```

---

## Section 10: Common Mistakes and How to Avoid Them

```text
MISTAKE: Not setting a User-Agent.
FIX: Always set a realistic, current browser User-Agent.
  The default "python-requests/2.x" is immediately blocked
  by any competent WAF. Use a real Chrome/Firefox UA string.

MISTAKE: No delay between requests.
FIX: Always add random delays. time.sleep(random.uniform(1, 4))
  at minimum. Faster than that and you risk rate limiting,
  IP banning, or causing a DoS condition on the target.

MISTAKE: Ignoring robots.txt.
FIX: Read it first. If it says Disallow, respect it unless
  you have explicit written authorization for a security
  assessment. Even then, document that you checked.

MISTAKE: Not using a Session object.
FIX: requests.Session() persists cookies, headers, and TCP
  connections across requests. Without it, you lose state
  between requests and trigger detection (no session cookie).

MISTAKE: Scraping HTML when an API exists.
FIX: Always check DevTools Network tab for XHR/Fetch calls
  first. JSON APIs are cleaner, faster, and more reliable
  than HTML parsing. Many SPAs have undocumented APIs.

MISTAKE: Not handling errors.
FIX: Wrap every request in try/except. Handle 403 (blocked),
  429 (rate limited), connection timeouts, and HTML structure
  changes gracefully. A scraper that crashes at 3 AM doesn't
  collect data.

MISTAKE: Hardcoding selectors without checking.
FIX: Websites change their HTML structure. Today's
  div.product-card might be div.item-card tomorrow. Add
  checks: if not soup.find("expected-element"), alert.

MISTAKE: Storing PII without a plan.
FIX: Before scraping any personal data (names, emails,
  phone numbers), define: why you need it, how you'll
  protect it, when you'll delete it, and who authorized it.

MISTAKE: Running scrapers from your corporate network.
FIX: Use a dedicated assessment network/VM. Your corporate
  IP getting blacklisted by a site you're scraping is
  an operational security failure.

MISTAKE: Not documenting your work.
FIX: Every scraper should have: target URL, authorization
  reference, data collected, date range, methodology, and
  findings. This is your evidence trail.
```

---

## Section 11: Where to Go Next

```text
AFTER COMPLETING THIS CODEX:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Intermediate:
[ ] Learn async scraping with httpx + asyncio
[ ] Study Scrapy middleware (proxy rotation, Playwright
    integration)
[ ] Explore the Scrapling library (modern, anti-bot aware)
[ ] Build automated OSINT pipelines that feed your SIEM
[ ] Study browser fingerprinting in depth
    (browserleaks.com, bot.sannysoft.com)

Advanced:
[ ] Learn to detect and analyze bot traffic with
    ModSecurity / Coraza WAF rules
[ ] Build custom Suricata/Zeek signatures for scraper
    traffic
[ ] Study TLS fingerprinting (JA3/JA4 hashes) and how
    to detect client spoofing
[ ] Contribute to SpiderFoot or other OSINT automation
    tools
[ ] Take SANS SEC587 (Advanced OSINT) if budget allows

Ongoing:
[ ] Subscribe to abuse.ch feeds for your SIEM
[ ] Monitor your org's external footprint quarterly
[ ] Update your scraping toolkit as libraries evolve
[ ] Share detection rules with your security team
[ ] Practice on CTF challenges that include web scraping
    components (HackTheBox, TryHackMe OSINT rooms)
```

---

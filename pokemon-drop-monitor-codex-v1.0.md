# Pokemon Card Drop Monitor Codex v1.0

**Purpose:** Build a Python-based stock monitoring system that watches retailer product pages for Pokemon TCG restocks and sends instant notifications via Discord, SMS, or push — giving you a human-speed advantage without crossing legal or ethical lines.

**Prerequisites:** Completed (or concurrent with) the Web Bots & Scrapers Codex v1.0. Basic Python. A Discord account. A phone that receives text messages.

**What This Project Teaches (Scraping Codex Skills Applied):**
Requests + BeautifulSoup (Codex Section 2.2), HTTP headers and status codes (Codex Section 1.2-1.4), rate limiting and random delays (Codex Section 3.1), error handling and resilience (Codex Section 3.5), API discovery via DevTools (Codex Section 1.7), session management (Codex Section 3.2), scheduling and long-running processes, webhook integrations (Discord, Twilio), configuration management, logging and operational monitoring.

---

## Section 1: Why a Monitor and Not a Bot — The Ethics Lecture You Need

## 1.1 The Line

There is a bright, clear line between monitoring and botting. You need to understand it before you write a single line of code, because crossing it can cost you your career in cybersecurity.

```text
MONITORING (what we're building):          BOTTING (what we're NOT building):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━         ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Checks a public product page               Automates the checkout process
Reads the stock status                      Fills cart, enters payment, submits
Sends YOU a notification                    Completes purchase without human
YOU click Buy with your hands               Bypasses queue systems and CAPTCHAs
Respects rate limits                        Hammers endpoints at machine speed
Uses a polite User-Agent                    Spoofs identity to evade detection
One request every few minutes               Hundreds of requests per second
```

## 1.2 Why Purchase Bots Are a Career-Ender for Security Professionals

This isn't hypothetical. This is your professional reality.

```text
LEGAL RISK:
━━━━━━━━━━━
Every major retailer (Pokemon Center, Target, Walmart, Best Buy,
Amazon, GameStop) explicitly prohibits automated purchasing in
their Terms of Service.

Bypassing anti-bot measures (Cloudflare, Akamai, queue systems,
CAPTCHAs) can trigger:
  - CFAA liability ("exceeds authorized access")
  - DMCA Section 1201 (anti-circumvention)
  - State computer fraud statutes
  - Civil lawsuits under contract law (ToS violation)

The BOTS Act (Better Online Tickets Sales Act, 2016) specifically
criminalizes using bots to circumvent security measures on
retail/ticket sites. While originally focused on tickets, its
principles are being extended.

CAREER RISK:
━━━━━━━━━━━━
You work in cybersecurity. Your job is to DEFEND systems from
exactly this kind of automated abuse. Getting caught running
purchase bots is the professional equivalent of a firefighter
committing arson. It's not just illegal — it's a betrayal of
the trust that makes your career possible.

ETHICAL RISK:
━━━━━━━━━━━━
The reason you can't buy Pokemon cards at retail is BECAUSE
other people run purchase bots. Building another one doesn't
fix the problem — it deepens the arms race. Every new bot
makes it harder for every human. If everyone stands up at the
concert, nobody sees better.
```

## 1.3 Why Monitoring is Legal and Ethical

```text
WHAT MAKES MONITORING ACCEPTABLE:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. You're accessing publicly available product pages
   (no login required, no paywall)

2. You're reading information the retailer intentionally
   displays to the public (price, stock status)

3. You're not circumventing any security measures
   (no CAPTCHA solving, no queue bypassing)

4. You're making requests at a frequency a human could
   reasonably match (once every 2-5 minutes, not 100/sec)

5. You're not automating any purchasing action

6. The hiQ v. LinkedIn precedent supports accessing
   public data

7. You're identifying your scraper honestly (or at
   minimum, not actively deceiving detection systems)

THIS IS FUNCTIONALLY IDENTICAL TO:
  - Refreshing a product page in your browser every few minutes
  - Using a browser extension that checks for page changes
  - Subscribing to a stock alert newsletter
  - Following a restock Twitter/X account

You're just automating the "check" part, not the "buy" part.
```

## 1.4 The Hacker Ethic — A Word About Culture

```text
There is a decades-long tradition in hacker culture of fighting
for the free flow of information. From the Jargon File to the
EFF, from Aaron Swartz to the cypherpunks, the principle has
always been: information wants to be free.

But "information wants to be free" was never "do whatever you
want with someone else's systems." The hacker ethic also says:

  "Do not damage or harm other people's data or systems."
  "Do not use a computer to steal."
  "Promote access to information, yes — but do no harm."

A purchase bot doesn't free information. It uses speed and
automation to take a LIMITED PHYSICAL RESOURCE (Pokemon cards)
away from other humans. That's not liberation — that's
scalping with extra steps.

The CFAA was originally written too broadly, and hackers have
spent decades pushing back against its overreach. That fight
matters. But you weaken that fight when you give prosecutors
real examples of people using automation to cause actual harm.
Every script kiddie running a sneaker bot makes it harder for
the EFF to argue that the CFAA criminalizes legitimate
security research.

Be on the right side of the line. Monitor. Alert. Let your
human hands do the buying.
```

---

## Section 2: Architecture — How the Monitor Works

## 2.1 System Design

```text
┌─────────────────────────────────────────────────────┐
│                  POKEMON DROP MONITOR               │
│                                                     │
│  ┌─────────┐    ┌──────────┐    ┌──────────────┐    │
│  │ Product |───>│  Stock   │───>│ Notification │    │
│  │ Checker │    │ Detector │    │   Sender     │    │
│  └─────────┘    └──────────┘    └──────────────┘    │
│       │              │               │              │
│       │              │          ┌────┴─────┐        │
│       ▼              ▼          │          │        │
│  ┌─────────┐   ┌──────────┐  ┌─┴──┐  ┌───┴───┐      │
│  │ Retailer│   │  State   │  │Disc│  │Twilio │      │
│  │  Configs│   │  Store   │  │cord│  │ SMS   │      │
│  └─────────┘   └──────────┘  └────┘  └───────┘      │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │              Scheduler (loop)               |    │
│  │   Check every N minutes with random jitter  │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │              Logger / Audit Trail           │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

## 2.2 Component Responsibilities

```text
COMPONENT          RESPONSIBILITY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Product Checker    Fetches the product page or API endpoint.
                   Parses the HTML/JSON for stock status.
                   Returns: in_stock (bool), price, product name.

Stock Detector     Compares current state to previous state.
                   Detects transitions: OUT_OF_STOCK -> IN_STOCK.
                   Prevents duplicate alerts (don't alert twice
                   for the same restock).

Notification       Sends alerts via configured channels:
Sender             Discord webhook, Twilio SMS, or both.
                   Includes: product name, price, direct URL.

Retailer Configs   Per-retailer settings: URL patterns, CSS
                   selectors for stock status, rate limits,
                   headers, and any quirks.

State Store        Tracks last-known status per product.
                   Simple JSON file (no database needed).

Scheduler          Runs checks on interval with random jitter.
                   Handles the main loop and graceful shutdown.

Logger             Records every check, state change, error,
                   and notification. Your audit trail.
```

## 2.3 Data Flow for a Single Check

```text
1. Scheduler fires → time to check Product X

2. Product Checker:
   a. Reads config for Product X (URL, selectors, headers)
   b. Sends GET request to product page
   c. Parses response for stock indicator
   d. Returns: {"in_stock": True, "price": "$49.99",
                "name": "Prismatic Evolutions ETB",
                "url": "https://..."}

3. Stock Detector:
   a. Reads last-known state from state store
   b. Compares: was OUT_OF_STOCK, now IN_STOCK
   c. State changed! Mark as transition event.
   d. Updates state store with new status + timestamp.

4. Notification Sender:
   a. Formats alert message with product details
   b. Sends to Discord webhook
   c. Sends SMS via Twilio
   d. Logs notification sent

5. Logger records everything:
   [2026-03-03 14:32:17] CHECK pokemon-center/prismatic-etb
   [2026-03-03 14:32:18] STATUS CHANGE: OUT_OF_STOCK -> IN_STOCK
   [2026-03-03 14:32:18] ALERT SENT: Discord + SMS
```

---

## Section 3: Building the Monitor — Step by Step

## 3.1 Project Setup

```bash
# Create project directory
mkdir -p ~/pokemon-monitor
cd ~/pokemon-monitor

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install requests beautifulsoup4 lxml python-dotenv

# Optional: for SMS alerts
pip install twilio

# Create project structure
mkdir -p configs logs
touch monitor.py config.py notifier.py checker.py state.py
touch .env .gitignore configs/products.json
```

**.gitignore** (never commit secrets):

```text
venv/
.env
*.pyc
__pycache__/
logs/
state.json
```

## 3.2 Configuration — .env and products.json

**.env** (your secrets — NEVER commit this):

```bash
# Discord
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN

# Twilio (optional — for SMS alerts)
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_FROM_NUMBER=+1234567890
TWILIO_TO_NUMBER=+1987654321

# Monitor settings
CHECK_INTERVAL_MIN=3
CHECK_INTERVAL_MAX=6
LOG_LEVEL=INFO
```

**configs/products.json** (what to monitor):

```json
{
  "products": [
    {
      "id": "pokemon-center-prismatic-etb",
      "name": "Prismatic Evolutions ETB",
      "url": "https://www.pokemoncenter.com/product/290-85956",
      "retailer": "pokemon-center",
      "enabled": true
    },
    {
      "id": "target-prismatic-etb",
      "name": "Prismatic Evolutions ETB (Target)",
      "url": "https://www.target.com/p/pokemon-prismatic-evolutions-etb/-/A-12345678",
      "retailer": "target",
      "enabled": true
    },
    {
      "id": "walmart-prismatic-booster",
      "name": "Prismatic Evolutions Booster Bundle (Walmart)",
      "url": "https://www.walmart.com/ip/Pokemon-Prismatic-Evolutions/123456789",
      "retailer": "walmart",
      "enabled": false
    }
  ]
}
```

## 3.3 The Configuration Module — config.py

```python
"""
config.py — Load environment variables and product configs.
"""
import os
import json
from dotenv import load_dotenv

load_dotenv()


class Config:
    """Central configuration loaded from .env and products.json."""

    # Discord
    DISCORD_WEBHOOK_URL = os.getenv("DISCORD_WEBHOOK_URL", "")

    # Twilio (optional)
    TWILIO_ACCOUNT_SID = os.getenv("TWILIO_ACCOUNT_SID", "")
    TWILIO_AUTH_TOKEN = os.getenv("TWILIO_AUTH_TOKEN", "")
    TWILIO_FROM_NUMBER = os.getenv("TWILIO_FROM_NUMBER", "")
    TWILIO_TO_NUMBER = os.getenv("TWILIO_TO_NUMBER", "")

    # Timing (seconds)
    CHECK_INTERVAL_MIN = int(os.getenv("CHECK_INTERVAL_MIN", "3")) * 60
    CHECK_INTERVAL_MAX = int(os.getenv("CHECK_INTERVAL_MAX", "6")) * 60

    # Logging
    LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO")

    # User-Agent — honest identification
    USER_AGENT = (
        "PokemonDropMonitor/1.0 "
        "(personal stock alert; contact: your-email@example.com)"
    )

    @staticmethod
    def load_products(path="configs/products.json"):
        """Load product list from JSON config."""
        with open(path, "r") as f:
            data = json.load(f)
        return [p for p in data["products"] if p.get("enabled", True)]
```

**Note the User-Agent:** We identify ourselves honestly. This is not a browser pretending to be Chrome. It's a personal monitoring tool that says what it is. This is the ethical choice — and for a low-frequency monitor (one request every few minutes), it rarely causes issues. If a retailer blocks this UA, they've made their wishes clear and you should respect that.

## 3.4 The State Store — state.py

```python
"""
state.py — Track last-known stock status per product.
Uses a simple JSON file. No database needed for this scale.
"""
import json
import os
from datetime import datetime


STATE_FILE = "state.json"


def load_state():
    """Load the state file, or return empty dict if missing."""
    if not os.path.exists(STATE_FILE):
        return {}
    with open(STATE_FILE, "r") as f:
        return json.load(f)


def save_state(state):
    """Write state to disk."""
    with open(STATE_FILE, "w") as f:
        json.dump(state, f, indent=2)


def check_transition(product_id, current_in_stock):
    """
    Compare current stock status against last known.
    Returns:
        "restock"    — was out of stock, now in stock (ALERT!)
        "still_in"   — was in stock, still in stock
        "sold_out"   — was in stock, now out of stock
        "still_out"  — was out of stock, still out of stock
        "first_check"— no previous state recorded
    """
    state = load_state()
    previous = state.get(product_id, {})
    was_in_stock = previous.get("in_stock", None)

    # Update state
    state[product_id] = {
        "in_stock": current_in_stock,
        "last_checked": datetime.now().isoformat(),
        "last_changed": (
            datetime.now().isoformat()
            if was_in_stock != current_in_stock
            else previous.get("last_changed", datetime.now().isoformat())
        ),
    }
    save_state(state)

    # Determine transition
    if was_in_stock is None:
        return "first_check"
    elif not was_in_stock and current_in_stock:
        return "restock"
    elif was_in_stock and current_in_stock:
        return "still_in"
    elif was_in_stock and not current_in_stock:
        return "sold_out"
    else:
        return "still_out"
```

## 3.5 The Product Checker — checker.py

This is the core scraping logic. Each retailer has different HTML structure, so we use a strategy pattern — one function per retailer.

```python
"""
checker.py — Fetch product pages and determine stock status.

Each retailer gets its own check function because HTML structures
differ. Add new retailers by writing a new check_<retailer>()
function and registering it in RETAILER_CHECKERS.
"""
import requests
from bs4 import BeautifulSoup
import logging

logger = logging.getLogger(__name__)


def make_request(url, user_agent):
    """
    Make a polite GET request with proper headers.
    Returns the Response object, or None on failure.
    """
    headers = {
        "User-Agent": user_agent,
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9",
        "Accept-Language": "en-US,en;q=0.9",
        "Accept-Encoding": "gzip, deflate, br",
        "Connection": "keep-alive",
    }
    try:
        response = requests.get(url, headers=headers, timeout=15)
        response.raise_for_status()
        return response
    except requests.exceptions.Timeout:
        logger.warning(f"Timeout fetching {url}")
    except requests.exceptions.HTTPError as e:
        status = e.response.status_code
        if status == 403:
            logger.warning(f"Blocked (403) by {url} — respect this")
        elif status == 429:
            logger.warning(f"Rate limited (429) by {url} — slow down")
        else:
            logger.warning(f"HTTP {status} from {url}")
    except requests.exceptions.ConnectionError:
        logger.warning(f"Connection failed to {url}")
    except Exception as e:
        logger.error(f"Unexpected error fetching {url}: {e}")
    return None


# ─── RETAILER-SPECIFIC CHECKERS ─────────────────────────────

def check_generic(url, user_agent):
    """
    Generic checker — looks for common stock indicators in HTML.
    Works as a fallback when no retailer-specific checker exists.

    Looks for:
      - "Add to Cart" / "Add to Bag" buttons (in stock)
      - "Out of Stock" / "Sold Out" / "Unavailable" text
      - "Notify Me" buttons (out of stock)
    """
    response = make_request(url, user_agent)
    if response is None:
        return None  # Check failed, don't update state

    soup = BeautifulSoup(response.text, "lxml")
    page_text = soup.get_text().lower()

    # Positive indicators (in stock)
    in_stock_signals = [
        "add to cart",
        "add to bag",
        "buy now",
        "in stock",
    ]

    # Negative indicators (out of stock)
    out_of_stock_signals = [
        "out of stock",
        "sold out",
        "currently unavailable",
        "notify me when available",
        "notify me",
        "coming soon",
    ]

    # Check for out-of-stock first (more specific)
    for signal in out_of_stock_signals:
        if signal in page_text:
            return {"in_stock": False, "signal": signal}

    # Check for in-stock signals
    for signal in in_stock_signals:
        if signal in page_text:
            return {"in_stock": True, "signal": signal}

    # Ambiguous — couldn't determine status
    logger.info(f"Could not determine stock status for {url}")
    return None


def check_pokemon_center(url, user_agent):
    """
    Pokemon Center specific checker.

    Pokemon Center uses a queue system for high-demand drops.
    When the queue is active, the product page may redirect
    to a waiting room. This is NOT the same as "in stock."

    Status indicators:
      - "Add to Cart" button present = in stock
      - "Out of Stock" text = sold out
      - Queue/waiting room redirect = drop is active but
        you need to join the queue manually
    """
    response = make_request(url, user_agent)
    if response is None:
        return None

    # Check for queue redirect
    if "queue" in response.url.lower() or "waiting" in response.url.lower():
        logger.info(f"Pokemon Center queue active for {url}")
        return {"in_stock": True, "signal": "queue-active",
                "note": "Queue is live — go join manually!"}

    soup = BeautifulSoup(response.text, "lxml")
    page_text = soup.get_text().lower()

    if "out of stock" in page_text or "sold out" in page_text:
        return {"in_stock": False, "signal": "sold-out"}

    # Look for add-to-cart button
    add_btn = soup.find("button", string=lambda t: t and "add to" in t.lower())
    if add_btn:
        return {"in_stock": True, "signal": "add-to-cart"}

    # Check for "Notify Me" (definitely out of stock)
    if "notify me" in page_text:
        return {"in_stock": False, "signal": "notify-me"}

    return check_generic(url, user_agent)


def check_target(url, user_agent):
    """
    Target specific checker.

    Target heavily uses JavaScript rendering, so a simple GET
    may not show stock status. However, Target's product API
    can sometimes be called directly.

    Tip: Use browser DevTools (Network tab, filter XHR) on a
    Target product page to find the API endpoint that returns
    stock data as JSON. That's far more reliable than parsing
    HTML.
    """
    # Try the HTML approach first
    result = check_generic(url, user_agent)
    if result is not None:
        return result

    # Target often requires JS rendering for stock status.
    # If generic check is ambiguous, log it.
    logger.info(
        f"Target stock status ambiguous for {url} — "
        f"consider using Playwright or finding the JSON API"
    )
    return None


def check_walmart(url, user_agent):
    """
    Walmart specific checker.

    Walmart product pages include stock data in embedded JSON-LD
    structured data, which is available in the raw HTML without
    JavaScript rendering.
    """
    response = make_request(url, user_agent)
    if response is None:
        return None

    soup = BeautifulSoup(response.text, "lxml")

    # Try JSON-LD structured data first
    import json
    for script in soup.find_all("script", type="application/ld+json"):
        try:
            data = json.loads(script.string)
            if isinstance(data, dict):
                avail = data.get("offers", {}).get("availability", "")
                if "InStock" in avail:
                    return {"in_stock": True, "signal": "json-ld-instock"}
                elif "OutOfStock" in avail:
                    return {"in_stock": False, "signal": "json-ld-outofstock"}
        except (json.JSONDecodeError, AttributeError):
            continue

    # Fall back to generic text search
    return check_generic(url, user_agent)


# ─── CHECKER REGISTRY ───────────────────────────────────────

RETAILER_CHECKERS = {
    "pokemon-center": check_pokemon_center,
    "target": check_target,
    "walmart": check_walmart,
    "generic": check_generic,
}


def check_product(product, user_agent):
    """
    Check a single product's stock status.

    Args:
        product: dict from products.json
        user_agent: string

    Returns:
        dict with keys: in_stock, signal, product (merged)
        or None if check failed
    """
    retailer = product.get("retailer", "generic")
    checker_fn = RETAILER_CHECKERS.get(retailer, check_generic)

    result = checker_fn(product["url"], user_agent)

    if result is not None:
        result["product_id"] = product["id"]
        result["product_name"] = product["name"]
        result["product_url"] = product["url"]
        result["retailer"] = retailer

    return result
```

## 3.6 The Notification Sender — notifier.py

```python
"""
notifier.py — Send alerts via Discord webhook and/or Twilio SMS.
"""
import requests
import logging
from datetime import datetime

logger = logging.getLogger(__name__)


def send_discord_alert(webhook_url, product_name, product_url,
                       price=None, signal=None, note=None):
    """
    Send a rich embed to a Discord channel via webhook.

    Discord webhook rate limit: 30 requests per 60 seconds.
    We're sending maybe 1-2 per restock event, so no issue.
    """
    if not webhook_url:
        logger.warning("Discord webhook URL not configured")
        return False

    # Build the embed
    description = f"**{product_name}** appears to be in stock!"
    if note:
        description += f"\n\n{note}"
    if signal:
        description += f"\n\nDetection signal: `{signal}`"

    embed = {
        "title": "IN STOCK ALERT",
        "description": description,
        "url": product_url,
        "color": 15844367,  # Gold color
        "timestamp": datetime.utcnow().isoformat(),
        "fields": [],
        "footer": {"text": "Pokemon Drop Monitor v1.0"},
    }

    if price:
        embed["fields"].append({
            "name": "Price",
            "value": price,
            "inline": True,
        })

    embed["fields"].append({
        "name": "Action",
        "value": f"[Go Buy It Now]({product_url})",
        "inline": True,
    })

    payload = {
        "username": "Pokemon Restock Alert",
        "embeds": [embed],
    }

    try:
        response = requests.post(
            webhook_url,
            json=payload,
            timeout=10,
        )
        if response.status_code == 204:
            logger.info(f"Discord alert sent for {product_name}")
            return True
        else:
            logger.warning(
                f"Discord webhook returned {response.status_code}: "
                f"{response.text}"
            )
            return False
    except Exception as e:
        logger.error(f"Discord alert failed: {e}")
        return False


def send_sms_alert(config, product_name, product_url, note=None):
    """
    Send an SMS alert via Twilio.

    Requires: pip install twilio
    Requires: Twilio account with verified phone number.
    Free tier: limited to verified numbers only.
    """
    if not config.TWILIO_ACCOUNT_SID:
        return False  # Twilio not configured, skip silently

    try:
        from twilio.rest import Client
    except ImportError:
        logger.warning("Twilio library not installed (pip install twilio)")
        return False

    body = f"RESTOCK ALERT: {product_name} is IN STOCK!\n{product_url}"
    if note:
        body += f"\n{note}"

    try:
        client = Client(config.TWILIO_ACCOUNT_SID, config.TWILIO_AUTH_TOKEN)
        message = client.messages.create(
            body=body,
            from_=config.TWILIO_FROM_NUMBER,
            to=config.TWILIO_TO_NUMBER,
        )
        logger.info(f"SMS alert sent for {product_name} (SID: {message.sid})")
        return True
    except Exception as e:
        logger.error(f"SMS alert failed: {e}")
        return False


def send_alerts(config, product_name, product_url,
                price=None, signal=None, note=None):
    """Send alerts on all configured channels."""
    results = {}
    results["discord"] = send_discord_alert(
        config.DISCORD_WEBHOOK_URL,
        product_name, product_url, price, signal, note,
    )
    results["sms"] = send_sms_alert(
        config, product_name, product_url, note,
    )
    return results
```

## 3.7 The Main Monitor — monitor.py

```python
#!/usr/bin/env python3
"""
monitor.py — Pokemon Card Drop Monitor

Main entry point. Runs the check loop with random jitter,
handles graceful shutdown, and coordinates all components.

Usage:
    python monitor.py              # Run the monitor
    python monitor.py --once       # Single check (for testing)
    python monitor.py --test-alert # Send a test notification
"""
import sys
import time
import random
import signal
import logging
from datetime import datetime

from config import Config
from checker import check_product
from state import check_transition
from notifier import send_alerts


# ─── LOGGING SETUP ──────────────────────────────────────────

def setup_logging(level_name):
    """Configure logging to both console and file."""
    level = getattr(logging, level_name.upper(), logging.INFO)

    formatter = logging.Formatter(
        "[%(asctime)s] %(levelname)s %(name)s: %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
    )

    # Console handler
    console = logging.StreamHandler()
    console.setLevel(level)
    console.setFormatter(formatter)

    # File handler (daily log)
    date_str = datetime.now().strftime("%Y-%m-%d")
    file_handler = logging.FileHandler(
        f"logs/monitor_{date_str}.log"
    )
    file_handler.setLevel(logging.DEBUG)  # Always log everything to file
    file_handler.setFormatter(formatter)

    root = logging.getLogger()
    root.setLevel(logging.DEBUG)
    root.addHandler(console)
    root.addHandler(file_handler)

    return logging.getLogger(__name__)


# ─── GRACEFUL SHUTDOWN ──────────────────────────────────────

running = True

def handle_shutdown(signum, frame):
    """Handle Ctrl+C gracefully."""
    global running
    running = False
    print("\nShutting down gracefully...")


signal.signal(signal.SIGINT, handle_shutdown)
signal.signal(signal.SIGTERM, handle_shutdown)


# ─── MAIN CHECK CYCLE ───────────────────────────────────────

def run_check_cycle(config, logger):
    """Run a single check across all configured products."""
    products = config.load_products()
    logger.info(f"Checking {len(products)} product(s)...")

    for product in products:
        logger.debug(f"Checking: {product['name']}")

        result = check_product(product, config.USER_AGENT)

        if result is None:
            logger.debug(f"  Could not determine status for {product['name']}")
            continue

        # Check for state transition
        transition = check_transition(
            product["id"], result["in_stock"]
        )

        logger.info(
            f"  {product['name']}: "
            f"{'IN STOCK' if result['in_stock'] else 'Out of Stock'} "
            f"({result.get('signal', 'unknown')}) "
            f"[{transition}]"
        )

        # Send alerts on restock transition
        if transition == "restock":
            logger.info(f"  *** RESTOCK DETECTED: {product['name']} ***")
            send_alerts(
                config,
                product_name=result["product_name"],
                product_url=result["product_url"],
                signal=result.get("signal"),
                note=result.get("note"),
            )

        elif transition == "sold_out":
            logger.info(f"  Product sold out: {product['name']}")

        elif transition == "first_check":
            if result["in_stock"]:
                logger.info(f"  First check — already in stock: {product['name']}")
                # Optionally alert on first check if in stock
                send_alerts(
                    config,
                    product_name=result["product_name"],
                    product_url=result["product_url"],
                    signal=result.get("signal"),
                    note="First check — was already in stock when monitoring started",
                )

        # Small delay between products (politeness)
        time.sleep(random.uniform(1.0, 3.0))


# ─── ENTRY POINT ────────────────────────────────────────────

def main():
    config = Config()
    logger = setup_logging(config.LOG_LEVEL)

    logger.info("=" * 50)
    logger.info("Pokemon Drop Monitor v1.0 starting")
    logger.info(f"Check interval: {config.CHECK_INTERVAL_MIN // 60}-"
                f"{config.CHECK_INTERVAL_MAX // 60} minutes")
    logger.info("=" * 50)

    # Handle command-line flags
    if "--test-alert" in sys.argv:
        logger.info("Sending test alert...")
        send_alerts(
            config,
            product_name="TEST PRODUCT",
            product_url="https://example.com",
            signal="test-signal",
            note="This is a test alert. If you see this, notifications work!",
        )
        return

    if "--once" in sys.argv:
        run_check_cycle(config, logger)
        return

    # Main loop
    while running:
        try:
            run_check_cycle(config, logger)
        except Exception as e:
            logger.error(f"Error in check cycle: {e}", exc_info=True)

        # Random wait between cycles
        wait = random.uniform(
            config.CHECK_INTERVAL_MIN,
            config.CHECK_INTERVAL_MAX,
        )
        logger.info(f"Next check in {wait / 60:.1f} minutes")

        # Sleep in small increments so we can catch shutdown signal
        end_time = time.time() + wait
        while running and time.time() < end_time:
            time.sleep(1)

    logger.info("Monitor stopped.")


if __name__ == "__main__":
    main()
```

---

## Section 4: Notification Setup Guides

## 4.1 Discord Webhook Setup (Free, Recommended)

Discord is the fastest free notification method. Alerts appear on your phone instantly if you have Discord mobile notifications enabled.

```text
SETUP STEPS:
━━━━━━━━━━━━

1. Open Discord and go to (or create) a server.

2. Create a channel for alerts:
   #pokemon-restock-alerts

3. Click the gear icon next to the channel name
   (Edit Channel).

4. Go to "Integrations" -> "Webhooks" -> "New Webhook".

5. Name it "Pokemon Monitor" and click "Copy Webhook URL".

6. Paste the URL into your .env file:
   DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...

7. On your phone, open Discord app:
   - Long-press the #pokemon-restock-alerts channel
   - Notification Settings -> "All Messages"
   This ensures your phone buzzes on every alert.

8. Test it:
   python monitor.py --test-alert

   You should see a gold embed in your Discord channel.
```

## 4.2 Twilio SMS Setup (Paid, Optional)

SMS alerts hit your phone even when you don't have Discord open. Twilio's free trial gives you limited credits to test.

```text
SETUP STEPS:
━━━━━━━━━━━━

1. Sign up at twilio.com (free trial includes ~$15 credit).

2. Get your Account SID and Auth Token from the Console
   Dashboard.

3. Get a Twilio phone number (Console -> Phone Numbers ->
   Buy a Number). SMS-capable numbers start around $1/month.

4. On the free trial, you must verify any number you want
   to send TO. Go to Console -> Verified Caller IDs and
   add your personal phone number.

5. Add credentials to .env:
   TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   TWILIO_AUTH_TOKEN=your_auth_token_here
   TWILIO_FROM_NUMBER=+1XXXXXXXXXX
   TWILIO_TO_NUMBER=+1XXXXXXXXXX

6. Install the Twilio library:
   pip install twilio

7. Test it:
   python monitor.py --test-alert
```

## 4.3 Extending to Other Notification Channels

The notifier module is designed to be extended. Some ideas:

```text
CHANNEL             HOW TO ADD IT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pushover            REST API, similar to Discord webhook.
                    $5 one-time purchase for the app.
                    Excellent for push notifications.

ntfy.sh             Free, open-source push notification
                    service. Dead simple REST API:
                    requests.post("https://ntfy.sh/your-topic",
                        data="Product is in stock!")

Email (SMTP)        Python's smtplib built-in module.
                    Slower than Discord/SMS but universal.

Telegram Bot        Free via Telegram Bot API. Create a bot
                    with @BotFather, get the token, POST to
                    api.telegram.org.

Slack Webhook       Nearly identical to Discord webhook
                    structure. Good for team use.

Desktop Toast       Use plyer library for cross-platform
                    desktop notifications. Good if you're
                    sitting at your computer.
```

---

## Section 5: Retailer-Specific Intelligence

## 5.1 Pokemon Center

```text
RESTOCK PATTERNS:
━━━━━━━━━━━━━━━━
- Restocks are irregular. In late 2025, roughly 1.5 major
  restocks per week were observed.
- No advance warning from official channels. Items go live
  without announcement.
- Queue system activates during high-demand drops. Once the
  queue is up, you join manually in your browser.
- The official support page states: "Customer Support does
  not have information on when or if out of stock items will
  be restocked."

MONITORING STRATEGY:
  - Check every 3-5 minutes during peak hours
  - Peak hours vary, but weekday mornings (ET) are common
  - Monitor the "Back in Stock" collection page AND
    individual product pages
  - Watch for queue activation as a restock signal

TECHNICAL NOTES:
  - Pokemon Center uses a React-based SPA
  - Some stock data may require JavaScript rendering
  - The queue system (powered by Queue-it or similar)
    redirects to a waiting room URL
  - Checking the raw HTML may miss JS-rendered stock buttons
  - For better results, investigate the site's API calls
    using browser DevTools (Network tab -> XHR filter)

RESPECT THE QUEUE:
  Pokemon Center explicitly warns: "Opening Pokemon Center
  in multiple tabs, using alternate links, or attempting
  any workarounds will not help you bypass the virtual
  queue, and those actions may result in other technical
  errors." Respect this. Your monitor tells you the queue
  is active. You join it like a human.
```

## 5.2 Target

```text
RESTOCK PATTERNS:
━━━━━━━━━━━━━━━━
- Online inventory often updates weekday mornings
- Local stores restock overnight or early morning
- Online availability can flip without notice

MONITORING STRATEGY:
  - Target product pages are JavaScript-heavy
  - Simple HTTP GET may not reveal stock status
  - Better approach: find the internal product API
    (check DevTools Network tab for /redsky/ API calls)
  - Monitor both online availability and in-store pickup

TECHNICAL NOTES:
  - Target uses heavy JavaScript rendering
  - The generic checker may return ambiguous results
  - Consider upgrading to Playwright for Target pages
  - Or find the XHR/Fetch API endpoint that returns
    stock data as JSON (much more reliable)
```

## 5.3 Walmart

```text
RESTOCK PATTERNS:
━━━━━━━━━━━━━━━━
- Inventory updates throughout the day
- Third-party Marketplace sellers complicate listings —
  monitor "Sold & shipped by Walmart" specifically
- Restocks often cluster around announcements and events

MONITORING STRATEGY:
  - Walmart includes structured data (JSON-LD) in the page
    HTML, which often contains availability info
  - The checker already parses JSON-LD (check_walmart)
  - Look for Schema.org "InStock" / "OutOfStock" values
  - Also check for the "Add to cart" button as fallback

TECHNICAL NOTES:
  - JSON-LD is available without JavaScript rendering
  - This makes Walmart one of the easiest retailers to
    monitor with Requests + BeautifulSoup
  - Be cautious about Marketplace vs Walmart-sold items
```

## 5.4 Amazon, Best Buy, GameStop

```text
GENERAL NOTES:
━━━━━━━━━━━━━━
These retailers have more aggressive anti-bot detection
(especially Amazon). For a personal monitor checking once
every few minutes, you may be fine. But be aware:

Amazon:
  - CAPTCHA challenges are common for automated requests
  - Product pages are complex with multiple sellers
  - Consider using the Amazon Product Advertising API
    (legitimate, rate-limited, requires affiliate account)
  - Or use Keepa/CamelCamelCamel alerts (existing services)

Best Buy:
  - JavaScript-heavy pages
  - Anti-bot detection is aggressive
  - May need Playwright for reliable checking

GameStop:
  - Stock status sometimes embedded in page source
  - Watch for exclusive bundles and pre-order windows
  - Less aggressive anti-bot than Amazon/Best Buy

FOR ALL OF THESE:
  If the retailer blocks your monitor (403 response),
  STOP. Do not try to bypass their detection. They have
  communicated that they don't want automated access.
  Use an existing alert service (TrackaLacker, RestockR,
  community Discord servers) for those retailers instead.
```

---

## Section 6: Running the Monitor — Operations

## 6.1 Testing the Setup

```bash
# Activate your virtual environment
source ~/pokemon-monitor/venv/bin/activate
cd ~/pokemon-monitor

# Make the logs directory
mkdir -p logs

# Test notifications work
python monitor.py --test-alert

# Run a single check cycle
python monitor.py --once

# Start the monitor (Ctrl+C to stop)
python monitor.py
```

## 6.2 Running as a Persistent Service

The monitor needs to run continuously to catch restocks. Here are your options:

## Option A: tmux session (simple, good for homelab)**

```bash
# Start a tmux session
tmux new -s pokemon-monitor

# Run the monitor
cd ~/pokemon-monitor
source venv/bin/activate
python monitor.py

# Detach: Ctrl+B, then D
# Reattach later: tmux attach -t pokemon-monitor
```

## Option B: systemd service (production, auto-restart)**

```ini
# /etc/systemd/system/pokemon-monitor.service
[Unit]
Description=Pokemon Card Drop Monitor
After=network.target

[Service]
Type=simple
User=your-username
WorkingDirectory=/home/your-username/pokemon-monitor
ExecStart=/home/your-username/pokemon-monitor/venv/bin/python monitor.py
Restart=on-failure
RestartSec=60

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/home/your-username/pokemon-monitor

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable pokemon-monitor
sudo systemctl start pokemon-monitor
sudo journalctl -u pokemon-monitor -f  # Watch logs
```

## Option C: cron job (check on schedule, simplest)**

```bash
# Run every 5 minutes via cron
*/5 * * * * cd /home/user/pokemon-monitor && venv/bin/python monitor.py --once >> logs/cron.log 2>&1
```

## 6.3 Monitoring the Monitor

```bash
# Check if it's running
ps aux | grep monitor.py

# Watch the logs in real time
tail -f ~/pokemon-monitor/logs/monitor_$(date +%Y-%m-%d).log

# Check the state file
cat ~/pokemon-monitor/state.json | python3 -m json.tool

# Count checks today
grep -c "Checking" ~/pokemon-monitor/logs/monitor_$(date +%Y-%m-%d).log
```

---

## Section 7: Common Problems and Solutions

```text
PROBLEM: All checks return "Could not determine stock status"
CAUSE:   The retailer's page structure changed, or JavaScript
         rendering is required.
FIX:     Open the product URL in a browser. Right-click ->
         View Page Source. Search for "Add to Cart" or "Out
         of Stock". If the text isn't in the raw source,
         you need Playwright. Or find the JSON API via
         DevTools (Network tab -> XHR).

PROBLEM: Getting 403 Forbidden on every request
CAUSE:   The retailer is blocking your requests.
FIX:     First, check if your User-Agent is being blocked.
         If switching to a browser UA helps temporarily, the
         site has basic UA filtering. But consider: if they're
         actively blocking you, respect that. Use a community
         alert service for that retailer instead.
         DO NOT start rotating UAs or proxies to evade
         detection — that crosses the line from monitoring
         to botting.

PROBLEM: Getting 429 Too Many Requests
CAUSE:   You're checking too frequently.
FIX:     Increase CHECK_INTERVAL_MIN and CHECK_INTERVAL_MAX
         in your .env file. 5-10 minute intervals are usually
         safe. If you're hitting 429 at 5 minutes, go to 10.

PROBLEM: Discord webhook returns 429
CAUSE:   Discord rate limiting (30 messages per 60 seconds).
FIX:     Unless you have 30+ products restocking simultaneously
         (unlikely), this shouldn't happen. If it does, add a
         small delay between webhook calls.

PROBLEM: Monitor crashes overnight
CAUSE:   Network interruption, DNS failure, or unhandled error.
FIX:     Use the systemd service with Restart=on-failure.
         Or wrap the main loop in broader exception handling
         (which monitor.py already does).

PROBLEM: False alerts (says in stock, but it's not)
CAUSE:   HTML structure changed, or the stock signal is
         from a third-party seller / different product variant.
FIX:     Review the detection signal in the log. Refine the
         retailer-specific checker. For example, Walmart
         might show "In Stock" for a Marketplace seller
         while the Walmart-sold version is still out of stock.

PROBLEM: Missing restocks (was in stock, no alert)
CAUSE:   Check interval too long. The product restocked and
         sold out between checks.
FIX:     Shorten the interval (but stay polite — 2-3 min
         minimum). For high-demand drops, the window can be
         as short as 60 seconds. You may want a dedicated
         "high alert" mode with shorter intervals for
         announced drop dates.

PROBLEM: Twilio SMS not sending
CAUSE:   On free trial, you can only send to verified numbers.
FIX:     Go to Twilio Console -> Verified Caller IDs and
         verify the phone number you're sending to.
```

---

## Section 8: Upgrade Paths — Where to Take This Next

## 8.1 Immediate Improvements

```text
[ ] Add Playwright support for JavaScript-rendered pages
    (Target, Best Buy). Use the Playwright checker from
    the Scraping Codex Section 2.3.

[ ] Add API-based checking for retailers with discoverable
    APIs. Use DevTools Network tab to find JSON endpoints.

[ ] Add price tracking — monitor price changes, not just
    stock status. Alert on price drops.

[ ] Add multiple notification delays — immediate alert for
    first restock, then cooldown (don't alert again for
    the same product for 30 minutes).

[ ] Add a "high alert" mode with shorter intervals for
    known drop dates (set via command-line flag or config).
```

## 8.2 Community Features

```text
[ ] Share your Discord server with friends who also collect.
    One monitor, many humans getting alerts.

[ ] Add a web dashboard (Flask/FastAPI) showing current
    status of all monitored products.

[ ] Track historical restock data — when do restocks happen
    most often? Build a pattern database over time.
```

## 8.3 Purple Team Tie-In

```text
This project reinforces real skills from the Scraping Codex:

SKILL EXERCISED            CODEX SECTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HTTP requests & headers    1.2, 1.3
HTML parsing (BS4)         2.2
Error handling & retries   3.5
Rate limiting              3.1
API discovery              1.7
Session management         3.2
Webhook integrations       New (Discord, Twilio)
systemd services           New (ops)
Log analysis               9.4
Legal/ethical boundaries   6.1, 6.2

EXTEND IT FURTHER FOR PURPLE TEAM:
  - Set up your homelab web server with a fake product page
  - Run your monitor against it
  - Analyze the access logs — can you detect your monitor?
  - Write Suricata/WAF rules that would flag it
  - This is the same purple team cycle from Codex Section 7.3
```

---

## Section 9: Existing Services Worth Knowing About

You don't have to build everything yourself. These services exist and are worth using alongside or instead of your custom monitor, especially for retailers that block automated requests.

```text
SERVICE              TYPE         COST        NOTES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TrackaLacker         App/Discord  Free +      Push alerts via mobile app,
                                  premium     Discord, email, SMS. Claims
                                              fastest detection times.

RestockR             App          Free +      Monitors Target, Walmart,
                                  premium     Amazon, Pokemon Center.

Visualping           Web          Freemium    Generic page change monitor.
                                              Works for any URL.

r/PokemonRestocks    Reddit       Free        Community-sourced alerts.
                                              Slower but free and reliable.

r/PKMNTCGDeals       Reddit       Free        Broader deals community.

Twitter/X Accounts   Twitter      Free        @PokemonStockUS and similar.
                                              Fast but requires Twitter.

Community Discord    Discord      Free        Multiple free servers with
Servers                                       restock bots. Search for
                                              "Pokemon restock alerts".
```

**The smart play:** Use your custom monitor for retailers you can reliably check (Walmart's JSON-LD is great), and use community services for retailers with aggressive anti-bot (Amazon, Best Buy). Belt and suspenders.

---

## Section 10: Quick Reference — The Complete File Set

Here's every file in the project and its line count for reference:

```text
pokemon-monitor/
├── .env                    # Your secrets (never commit)
├── .gitignore              # Excludes venv, .env, logs, state
├── monitor.py              # Main entry point and scheduler
├── config.py               # Configuration loader
├── checker.py              # Retailer-specific stock checkers
├── state.py                # State tracking (JSON file)
├── notifier.py             # Discord + Twilio notification senders
├── configs/
│   └── products.json       # Product URLs and metadata
├── logs/
│   └── monitor_YYYY-MM-DD.log  # Daily log files
└── state.json              # Auto-generated, tracks stock status
```

---

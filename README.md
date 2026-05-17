# How to Handle 429 Errors When Scraping (Without Losing Your Mind or Your Data)

You're halfway through a 50,000-URL job, your script has been huming along for an hour, and then it happens: every single response comes back `429 Too Many Requests`. Your queue stalls. Your data has gaps. You start Googling fixes at 1 AM.

I've been there more times than I'd like to admit. After years of building scrapers that need to run reliably at scale, I've landed on a handful of approaches that actually work — and one that basically eliminated429s from my workflow entirely.

Let me walk through what's happening under the hood, DIY fixes worth trying, and when it makes sense to hand the problem off to infrastructure that's purpose-built for it.

## What a 429 Actually Means (and Why Sites Are Getting Stricter)

HTTP 429 is the server's way of saying "you're sending too many requests in short a window." It's a rate-limit enforcement response defined in RFC 6585.

The thing most people miss: the threshold isn't universal. One site might tolerate 60 requests per minute from a single IP. Another might flag you after 5 requests in 10 seconds. E-commerce sites, search engines, and social platforms have gotten dramatically more aggressive with rate limiting over the past couple of years. Many now combine IP fingerprinting, TLS fingerprinting, and behavioral analysis before they even send a 429 — sometimes they just silently serve you garbage data or CAPTCHAs instead.

The `Retry-After` header (when present) tells you how long to wait. Most scrapers ignore it. Don't be most scrapers.

## DIY Fixes That Actually Help

### Exponential Backoff with Jitter

The single most important pattern. When you get a 429, don't just wait a flat 5 seconds and retry. Double your wait time on each consecutive failure, and add a random jitter so you're not hammering the server in synchronized bursts.

```python
import time
import random
import requests

def fetch_with_backoff(url, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url)
        if response.status_code == 200:
            return response
        if response.status_code == 429:
            wait = (2 ** attempt) + random.uniform(0, 1)
            retry_after = response.headers.get("Retry-After")
            if retry_after:
                wait = max(wait, float(retry_after))
            time.sleep(wait)
    return None
```

This works for small jobs. It falls apart when you're running thousands of concurrent requests because you're still burning through a single IP's rate-limit budget.

### Request Throttling and Concurrency Caps

Before you even hit a 429, slow down proactively. Set a per-domain request rate (I usually start at 1 request per 2 seconds for unknown sites) and cap your concurrent connections.

```python
import asyncio
from aiohttp import ClientSession, TCPConnector

semaphore = asyncio.Semaphore(10)  # max 10 concurrent requests

async def throttled_fetch(session, url):
    async with semaphore:
        async with session.get(url) as resp:
            await asyncio.sleep(1.5)  # breathing room between requests
            return await resp.text()
```

The downside: your50,000-URL job now takes days instead of hours.

### Rotating Proxies (the Manual Way)

Distributing requests across multiple IPs is the most direct way to multiply your effective rate limit. If a site allows60 req/min per IP and you have 100 proxies, your theoretical ceiling is 6,000 req/min.

In practice, it's messier. Cheap datacenter proxies get detected and blocked fast. Residential proxies are more reliable but expensive and inconsistent. You end up building a proxy health-check layer, a rotation scheduler, a ban-detection system… and suddenly you're maintaining infrastructure instead of scraping data.

I ran my own proxy pool for about eight months. The maintenance overhead ate more time than the actual scraping projects.

### Respect robots.txt Crawl-Delay

Some sites specify a `Crawl-delay` directive in their robots.txt. It's not part of the HTTP spec, but honoring it often keeps you under the radar longer than ignoring it.

## When DIY Stops Making Sense

Here's the pattern I kept hitting: I'd build a scraper with backoff, throttling, and proxy rotation. It'd work great for two weeks. Then the target site would update their anti-bot stack, and I'd spend a weekend debugging why my residential proxies were suddenly all getting429s or CAPTCHAs.

The math eventually stopped working. My time debugging anti-bot measures was worth more than paying for a service that handles it automatically.

## Offloading 429 Handling to ScraperAPI

ScraperAPI sits between your code and the target site. You send a normal GET request to their endpoint, they handle proxy rotation, retry logic, CAPTCHA solving, and browser rendering on their end. You get back the HTML (or JSON) with a 200, or you don't get charged.

The part that's directly relevant to 429 errors: ScraperAPI automatically retries failed requests — including 429s — for up to 60 seconds using different IPs and request profiles. You only pay for successful responses. So a 429 on their end never becomes a 429 on your end.

Here's what the integration looks like:

```python
import requests

API_KEY = "your_scraperapi_key"

def scrape(url):
    params = {
        "api_key": API_KEY,
        "url": url,
        "render": "false
    }
    response = requests.get("http://api.scraperapi.com", params=params)
    return response.text
```

That's it. No backoff logic, no proxy management, no retry code. The 429 problem becomes someone else's infrastructure problem.

I switched my main production scrapers over about a year ago. The jobs that used to fail15-20% of the time due to rate limits now complete with 98%+ success rates, and I haven't touched the retry logic since.

## ScraperAPI Plans at a Glance

| Plan | Monthly Credits | Concurrent Threads | Geotargeting | Monthly Price | Link |
| --- | --- | --- | --- | --- | --- |
| Free | 5,000 | 5 | Limited | $0 | [Start free — no card required](https://www.scraperapi.com/signup?fp_ref=coupons&sub1=table) |
| Hobby | 100,000 | 20 | US, EU | $49 ($29/mo annual) | [Get the Hobby plan](https://www.scraperapi.com/?fp_ref=coupons&sub1=table) |
| Startup | 500,000 | 50 | US, EU, Asia | $149 ($99/mo annual) | [Get the Startup plan](https://www.scraperapi.com/?fp_ref=coupons&sub1=table) |
| Business | 3,000,000 | 100 | All geos | $499 ($299/mo annual) | [Get the Business plan](https://www.scraperapi.com/?fp_ref=coupons&sub1=table) |
| Enterprise | 8,000,000+ | 200+ | All geos + dedicated support | Custom | [Talk to sales](https://www.scraperapi.com/?fp_ref=coupons&sub1=table) |

For most people dealing with 429 errors on mid-size scraping jobs (under 500K pages/month), the Startup plan hits the sweet spot —50 concurrent threads is enough to keep jobs fast without neding to think about throttling.

## Combining DIY + API: A Hybrid Approach

You don't have to go all-in on either approach. What I actually run in production is a hybrid:

1. **First pass**: hit the target directly with polite throttling (1 req/2s). Cheap and fast for sites that don't rate-limit agressively.
2. **On 429 or block**: route the failed URL through ScraperAPI as a fallback.

```python
import requests
import time

SCRAPER_API_KEY = "your_key"

def smart_fetch(url):
    # Try direct first
    resp = requests.get(url, timeout=10)
    if resp.status_code == 200:
        return resp.text

    # Fallback to ScraperAPI on 429 or 403
    if resp.status_code in (429, 403):
        api_resp = requests.get(
            "http://api.scraperapi.com",
            params={"api_key": SCRAPER_API_KEY, "url": url}
        )
        if api_resp.status_code == 200:
            return api_resp.text

    return None
```

This keeps your API credit usage low while guaranteeing you don't lose data to rate limits.

## FAQ

**What exactly triggers a 429 when scraping?**
Too many requests from the same IP (or fingerprint) within the site's rate-limit window. The threshold varies wildly — some sites are generous at hundreds per minute, others will flag you after a handful of rapid requests.

**Will rotating proxies alone fix 429 errors?**
They help a lot, but they're not bulletproof. Sites increasingly fingerprint beyond IP address — looking at TLS signatures, header order, and request patterns. Proxies buy you more headroom, but you still need proper request spacing and header randomization.

**Does ScraperAPI charge me for failed requests?**
No. You only get billed for requests that return a successful response. If ScraperAPI can't get through after its internal retry cycle, you don't pay for that request. 👉 [Check the free tier to test it yourself](https://www.scraperapi.com/signup?fp_ref=coupons&sub1=faq)

**How fast can I scrape without hitting rate limits?**
There's no universal answer. Start conservative (1-2 requests per second per domain), monitor for 429s, and scale up gradually. Or use a service like ScraperAPI that handles the pacing automatically across massive proxy pool.

**Is it legal to scrape a site that sends 429 responses?**
A 429 is a technical rate limit, not a legal prohibition. Legality depends on what you're scraping, how you use the data, and the site's terms of service. The 429 itself just means "slow down," not "go away forever."

## The Short Version

429 errors are a solvable problem. For small jobs, exponential backoff and basic throttling get the job done. For anything at scale — or anything you need to run reliably without babysitting — offloading retry logic and proxy management to a dedicated service saves real time.

I burned months building and maintaining my own anti-429 infrastructure before accepting that it wasn't a good use of my hours. If you're at that crossroads now, the free tier is enough to test whether it actually fixes your completion rates before committing any money.

👉 [Grab 5,000 free API credits and test it against your worst429 targets](https://www.scraperapi.com/signup?fp_ref=coupons&sub1=footer)

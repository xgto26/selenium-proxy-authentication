# selenium proxy: how to wire authenticated, rotating residential IPs into Chrome, Firefox and headless Python scripts

You add `--proxy-server`, run the script, open `httpbin.org/ip`, and the page shows your own IP address. Or it works on your laptop and dies at 2 a.m. on the server because the proxy the browser was bound to is gone. Selenium's proxy story is awkward in a specific way: the proxy is attached to the browser process at launch, not to a request object you can swap between calls. That single design detail decides which proxies you can use, how you rotate them, and how badly things break when they fail.

This guide walks through the setups that actually hold up, where Selenium's proxy handling breaks, and what a residential proxy plan costs once you start running real workloads.

## Why a proxy in Selenium behaves nothing like a proxy in `requests`

With `requests`, proxies are a dict. Swap it, retry, move on. Auth is a URL string. Rotating per request costs you nothing.

In Selenium, three things change:

- **The proxy is set when the browser starts.** Changing it mid-run means either building a new driver or using `selenium-wire`, which patches the proxy in place. Starting a fresh Chrome instance costs roughly 2–3 seconds per cycle, so "rotate the proxy every request" is not a free strategy here.
- **The browser can't answer a proxy auth prompt.** Chrome and Firefox accept `--proxy-server=host:port` for open proxies, but username/password proxies need an extension, a local forwarder, or `selenium-wire`. A silent 407 shows up as a failed page load, not a helpful error.
- **The IP isn't the only thing a site sees.** A browser leaks several dozen signals beyond IP. Residential IPs help with the reputation half of the problem; they don't fix a headless Chrome that reports `navigator.webdriver` or an old user agent.

That last point matters for plan selection. If a site blocks you with a datacenter IP in the first place, a residential IP is usually the fix. If it blocks you because the fingerprint looks automated, no proxy tier rescues that.

## The three ways to attach a proxy, ranked by when they're useful

### 1. Chrome/Firefox flags: the fast path for open proxies

python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

opts = Options()
opts.add_argument("--proxy-server=http://HOST:PORT")
opts.add_argument("--headless=new")

driver = webdriver.Chrome(options=opts)
driver.get("https://httpbin.org/ip")
print(driver.find_element("tag name", "body").text)


Firefox does the same through a preference dict:

python
from selenium.webdriver import Firefox, FirefoxOptions

opts = FirefoxOptions()
opts.set_preference("network.proxy.type", 1)
opts.set_preference("network.proxy.http", "HOST")
opts.set_preference("network.proxy.http_port", PORT)
opts.set_preference("network.proxy.ssl", "HOST")
opts.set_preference("network.proxy.ssl_port", PORT)

driver = Firefox(options=opts)


This is the cleanest setup and the one most tutorials stop at. It only works when the proxy endpoint doesn't ask for credentials — which is exactly why IP whitelisting and local port forwarding are worth caring about.

### 2. Extension-based auth: works, with caveats

Chrome can't take user/pass through the command line, so the common workaround is packaging a small extension that answers the auth challenge. It works, but under `--headless=new` extension service workers can fail silently — no exception, no log, just an empty page. If your proxy auth must run through an extension, running the browser under a virtual display on the server is more predictable than pure headless.

### 3. Local forwarder or `selenium-wire`: the setup that scales

Two variants of the same idea:

- **A local port forwarder.** Something outside the browser listens on `127.0.0.1:PORT` and forwards traffic to an authenticated upstream proxy. Selenium only ever sees a plain `--proxy-server=http://127.0.0.1:PORT`. Credentials never enter the browser, and rotation can happen behind the scenes.
- **`selenium-wire`.** A drop-in replacement for the Selenium webdriver module that accepts a proxy with credentials and lets you reassign it mid-run:

python
from seleniumwire import webdriver

driver = webdriver.Chrome(seleniumwire_options={
    "proxy": {
        "http": "http://user:pass@host:port",
        "https": "http://user:pass@host:port",
    }
})
driver.get("https://httpbin.org/ip")
driver.proxy = {"http": "http://user:pass@host2:port"}  # rotate without relaunching


The trade-off: `selenium-wire` records every request and response in memory by default. Long scraping runs need the capture limit trimmed, or the process grows until something kills it.

## Sticky or rotating? The choice most guides get backwards

Proxy marketing pushes rotation. Selenium work usually benefits from the opposite.

A single modern page pulls in dozens of subresources from the same host. If each request leaves from a different IP, you look like a bot to any half-decent analytics layer, and anything session-based — carts, logins, paginated flows — falls apart immediately. A sticky session that holds one IP for a set number of minutes, with rotation between sessions, is the default that survives contact with real sites.

Rotate per request when the unit of work genuinely is a single isolated request and the site doesn't keep state. Don't rotate mid-page.

## What 9Proxy's two models mean for a Selenium script

9Proxy is a residential proxy platform advertising 20M+ residential IPs across 90+ countries, with targeting down to country, state, city, ZIP and ISP, and both HTTP/HTTPS and SOCKS5 endpoints. It sells two distinct products, and they behave very differently in an automation script.

**Residential proxies by GB** are the natural fit for Selenium on a server:

- Pay per gigabyte consumed, not per IP. You can generate unlimited endpoints.
- Auth by sub-user username/password, or by IP whitelisting for machines with a fixed address.
- Sticky sessions with a configurable duration, or rotating sessions that change IP per request.
- Everything is managed from the dashboard — no desktop software on the machine running the browser.
- Traffic is valid for 180 days, unlimited on Enterprise plans.

Practically, you create a sub-user with a username and password, pick a location, choose sticky or rotating, and the Proxy Generator hands you a connection string in a format you select (`user:pass:host:port`, `host:port:user:pass`, and so on), plus ready-made code samples to paste into your script. That connection string drops straight into the `selenium-wire` options above, or into a forwarder.

**Residential proxies by IP** work differently and this is a detail worth reading before you buy:

- You pay per IP with unlimited bandwidth on that IP. Unused IPs never expire.
- Setup requires the 9Proxy desktop app, which does local port forwarding — optionally with proxy authentication.
- An individual IP stays alive for a few hours up to around 24 hours. Rotation isn't automatic; the company offers an Auto Rotation Proxy that switches at intervals you set on selected ports.

The app-based model is actually convenient in one specific case: port forwarding means your Selenium script points at `127.0.0.1:PORT` with no auth flag and no extension, which sidesteps the whole authentication problem. The catch is that the app has to run on the same machine as the browser, which gets awkward on headless cloud boxes, and a long-running session can outlive its underlying IP.

Short version: **GB-based for servers, containers and cloud runs; IP-based when each job moves a lot of data through a small number of stable identities.**

## Full plan list and current pricing

Two notes before the numbers. GB-based pricing is unchanged; IP-based and bundle pricing went up on 1 June 2026 — the company's own announcement confirms that split, and third-party reviews from mid-2026 reflect it. Everything below is a one-time balance purchase rather than a monthly subscription.

### IP-based residential packages (unlimited bandwidth per IP)

| Package | Effective price per IP | Total | Purchase |
| --- | --- | --- | --- |
| 100 IPs | $0.24 | $24 | [ Get 100 residential IPs for $24](https://bit.ly/9-Proxy) |
| 500 IPs | $0.144 | $72 | [ Get 500 IPs for $72](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus IPs | $0.084 | $126 | [ Get 1,000 IPs plus 500 bonus for $126](https://bit.ly/9-Proxy) |
| 2,500 IPs | $0.084 | $210 | [ Get 2,500 IPs for $210](https://bit.ly/9-Proxy) |
| 5,000 IPs | $0.072 | $360 | [ Get 5,000 IPs for $360](https://bit.ly/9-Proxy) |
| 15,000 IPs | $0.048 | $720 | [ Get 15,000 IPs for $720](https://bit.ly/9-Proxy) |
| 25,000 IPs | $0.035 | $863 | [ Get 25,000 IPs for $863](https://bit.ly/9-Proxy) |
| 50,000 IPs | $0.029 | $1,438 | [ Get 50,000 IPs for $1,438](https://bit.ly/9-Proxy) |
| 100,000 IPs (business) | $0.023 | $2,300 | [ Get 100,000 IPs for $2,300](https://bit.ly/9-Proxy) |
| 200,000 IPs (business) | $0.021 | $4,140 | [ Get 200,000 IPs for $4,140](https://bit.ly/9-Proxy) |
| 500,000 IPs (business) | $0.018 | $8,625 | [ Get 500,000 IPs for $8,625](https://bit.ly/9-Proxy) |

### GB-based residential packages

| Package | Price per GB | Total | Validity | Purchase |
| --- | --- | --- | --- | --- |
| 5 GB | $3.00 | $15 | 180 days | [ Get 5 GB for $15](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | $2.10 | $105 | 180 days | [ Get 50 GB plus 5 GB bonus for $105](https://bit.ly/9-Proxy) |
| 100 GB | $1.50 | $150 | 180 days | [ Get 100 GB for $150](https://bit.ly/9-Proxy) |
| 200 GB | $1.00 | $200 | 180 days | [ Get 200 GB for $200](https://bit.ly/9-Proxy) |
| 1,000 GB | $0.80 | $800 | 180 days | [ Get 1,000 GB for $800](https://bit.ly/9-Proxy) |
| 2,000 GB | $0.75 | $1,500 | 180 days | [ Get 2,000 GB for $1,500](https://bit.ly/9-Proxy) |
| 3,000 GB (Enterprise) | $0.72 | $2,160 | Unlimited | [ Get 3,000 GB Enterprise for $2,160](https://bit.ly/9-Proxy) |
| 6,000 GB (Enterprise) | $0.70 | $4,200 | Unlimited | [ Get 6,000 GB Enterprise for $4,200](https://bit.ly/9-Proxy) |
| 10,000 GB (Enterprise) | $0.68 | $6,800 | Unlimited | [ Get 10,000 GB Enterprise for $6,800](https://bit.ly/9-Proxy) |

Enterprise plans also add team mode (one owner plus up to five members), no-expiration bandwidth sharing inside the team, per-member traffic controls and activity logs.

### Bundle packages (IPs + bandwidth)

| Bundle | Contents | Price | Purchase |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [ Get the Starter bundle for $30](https://bit.ly/9-Proxy) |
| Popular | 1,500 IPs + 50 GB | $180 | [ Get the Popular bundle for $180](https://bit.ly/9-Proxy) |
| Pro | 5,000 IPs + 500 GB | $720 | [ Get the Pro bundle for $720](https://bit.ly/9-Proxy) |

Payment options include bank cards, crypto (USDT, BTC, ETH, LTC, DOGE and others), Alipay, Apple Pay and Google Pay. Reviews from mid-2026 mention an extra 5% off or a 5% credit bonus on selected payment methods, and 9Proxy advertises a 5% discount for users who sign up through a referral invite code — which is exactly what the link in the tables above applies. Pricing moves, so confirm the number on the checkout page before you commit.

## Doing the cost math before you pick a model

Take a mid-sized Selenium job: 100,000 page loads in a month, roughly 200 KB of transferred data per page after assets are blocked. That's about 20 GB.

- On the GB model, the 50 GB + 5 GB package at $105 covers it with room to spare, and you never think about IP counts.
- On the IP model, 100 IPs at $24 gives you unlimited bandwidth on those IPs. If each IP survives even four hours and you rotate on a schedule, 100 IPs deliver several hundred sessions per day. For a job that doesn't need new IPs constantly, that beats the GB price by a wide margin.

Flip the workload and the answer flips too. If your script deliberately touches 50,000 distinct IPs for geo-checking, the IP model becomes a five-figure purchase while the GB model stays cheap. **IP model when bandwidth is heavy and identities are few; GB model when identities are many and bandwidth is light.** That single sentence covers most purchase decisions.

One more factor: the 180-day window on GB traffic is generous but not infinite. If it takes you a year to burn 200 GB, you're buying 180 days of validity, not 200 GB forever.

## Failure modes worth checking before you blame the proxy

- **The flag never applied.** Always load an IP-echo endpoint first. If it shows your real address, nothing downstream matters.
- **Credentials with special characters.** An `@` or `:` inside a password breaks the URL form. URL-encode it, or switch to whitelisting.
- **Headless plus extension.** Silent auth failures under `--headless=new` are common; virtual display instead of headless is the usual workaround.
- **Wrong timeout.** `set_page_load_timeout()` covers the whole page load, not the proxy handshake. A dead proxy burns the entire timeout before failing. Test reachability with a lightweight `requests` call before you build the driver.
- **`--ignore-certificate-errors` masking everything.** It hides genuine TLS problems as well as the harmless ones.
- **One driver reused forever.** With residential IPs that live hours, a browser bound to a dead IP hangs rather than errors. Sessions need a lifespan, and so does the driver behind them.
- **`selenium-wire` memory growth.** Set capture limits on long runs.

## Where 9Proxy isn't the right pick

Worth saying plainly. 9Proxy sells residential IPs; there's no mobile network product and datacenter proxies were still listed as "coming soon". If you need carrier-grade mobile IPs for app testing, or tens of millions of tokens of exotic city-level targeting, you're shopping in a different bracket. Traffic-creator's June 2026 price comparison puts 9Proxy around $1.30/GB blended — cheaper than most, but its Tier 3 success rates (70–85% in their testing of budget providers generally) trail premium options that charge several times more. For Tier 1 and Tier 2 targets, budget providers are typically fine.

And if the site you're scraping doesn't need JavaScript rendering, drop Selenium entirely. `requests` or `httpx` will outrun it by a wide margin, and then proxy rotation becomes a one-line dict swap instead of a browser lifecycle problem. Selenium is for pages that genuinely require a rendered DOM.

If your work does need a browser and a residential IP, the practical setup is: create a 9Proxy account, generate a sub-user for the script, choose sticky sessions, and paste the connection string into `selenium-wire` — [👉 Create a 9Proxy account and generate your first sticky endpoint](https://bit.ly/9-Proxy). Start with a small GB package to measure your real bandwidth per 1,000 page loads, then move to an IP package once you know your actual consumption. Guessing between the two models is how people ove

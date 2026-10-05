# curl proxy authentication: the flags that work, the 407s that don't, and keeping credentials out of your shell history

Two different things get called "authentication" in a curl command, and confusing them is the single most common reason a proxy request comes back with `407 Proxy Authentication Required`.

One is authentication to the website you're scraping. The other is authentication to the proxy standing in front of it. curl keeps them completely separate, which is why it has two nearly identical flags that differ by letter case. Get that straight and most of the weirdness disappears.

Here's the short version of what this article covers: which flag does what, how to pass credentials without poisoning your shell history, why a proxy that works over `http://` suddenly fails with `curl: (56) CONNECT tunnel failed, response 407` when you point it at an HTTPS URL, and how to pick a proxy service whose authentication model doesn't fight you.

## Which flag authenticates what

| Flag | What it authenticates you to | What breaks if you mix them up |
| --- | --- | --- |
| `-x`, `--proxy http://host:port` | Nothing. It just says where the request goes. | Connection goes direct, proxy sits unused, your IP leaks |
| `-U`, `--proxy-user user:pass` | The proxy server | Proxy rejects you with a 407, or the proxy credentials never get sent |
| `-u`, `--user user:pass` | The destination website | Proxy password travels to the target site in an `Authorization` header |
| `--proxy-header "Name: value"` | Extra headers sent to the proxy only | Header goes to the origin server instead |

The `-U` versus `-u` trap is worth repeating because it's genuinely easy to hit. Type `curl -u user:pass -x http://proxy:8080 https://example.com` and curl dutifully sends those credentials to `example.com` as site credentials, while telling the proxy nothing. You get a 407. Then you retry with the same wrong flag and get the same 407, and start blaming the provider.

Capital `-U` and lowercase `-u` are not aliases. They go to different machines.

## Three ways to hand curl the credentials

curl has more than one syntax for proxy credentials, and each has a different exposure profile.

**Option 1: separate flag, credentials kept out of the URL.**

bash
curl -x http://proxy.example:8080 \
     -U "username:password" \
     https://httpbin.org/ip


**Option 2: credentials embedded in the proxy URL.**

bash
curl -x "http://username:password@proxy.example:8080" \
     "https://httpbin.org/ip"


Both work. Option 2 is shorter, which is exactly why people copy it into scripts and then leave it there.

**Option 3: environment variables.**

bash
export https_proxy="http://username:password@proxy.example:8080"
curl https://httpbin.org/ip


No flags at all in the command, which is convenient until you forget the variable is still set and wonder why a later `curl` call is routing through a proxy in another country.

Whichever you pick, quote the whole argument. A password containing `@`, `#`, `&` or `:` will be mangled by the shell before curl ever sees it, and the failure looks like nothing at all rather than a clean error.

## When Basic isn't what the proxy wants

By default, curl uses Basic authentication for proxies, which base64-encodes `user:password` and sends it. Fine for a proxy connection you control over TLS; less ideal over plain HTTP, where base64 is only barely better than plaintext.

Some proxies, mostly corporate ones, want something else. The `407` response includes a `Proxy-Authenticate` header listing what the proxy actually supports, and curl can be told to speak it:

bash
curl -x http://proxy.example:8080 -U "user:pass" \
     --proxy-ntlm https://example.com

curl -x http://proxy.example:8080 -U "user:pass" \
     --proxy-digest https://example.com

curl -x http://proxy.example:8080 -U "user:pass" \
     --proxy-negotiate https://example.com


If you don't know and don't want to care, `--proxy-anyauth` lets curl negotiate the scheme itself, at the cost of an extra round trip or two on the first request. That's a reasonable default when you're pointing curl at an enterprise proxy you didn't configure.

Commercial proxy providers are a different story. Most of them use plain username/password Basic auth over their gateway, so `-U` or credentials in the URL is all you need.

## SOCKS5 behaves differently, in one way that matters

SOCKS proxies accept credentials the same way (`-U`, or embedded in the URL), but the scheme prefix changes what happens to DNS. Compare:

bash
curl -x socks5://user:pass@proxy.example:1080 https://example.com
curl -x socks5h://user:pass@proxy.example:1080 https://example.com


The trailing `h` in `socks5h` means hostname resolution happens at the proxy. Without it, your machine resolves the domain locally and hands the proxy an IP. If your target is geo-restricted or your local DNS is being filtered, `socks5` can fail or produce the wrong result while looking perfectly authenticated. If you're using SOCKS to reach a site that blocks your region, use `socks5h`.

## Curl 407 on HTTP vs HTTPS: two different errors for one cause

This one confuses people because the error message changes depending on the URL scheme, even though the underlying problem is identical.

Against an `http://` target, a proxy authentication failure comes back as a normal-looking `407` response:


> GET /get HTTP/1.1
< HTTP/1.1 407 Proxy Authentication Required


Against an `https://` target, curl has to first open a tunnel with a `CONNECT` method. If the proxy rejects that with a 407, the tunnel never exists, so there's no HTTP response for curl to hand you. You get this instead:


> CONNECT example.com:443 HTTP/1.1
< HTTP/1.1 407 Proxy Authentication Required
* CONNECT tunnel failed, response 407
curl: (56) CONNECT tunnel failed, response 407


Same cause, same fix. Don't go hunting for TLS problems.

## Troubleshooting table

| What you see | Most likely cause | Fix |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Missing or wrong proxy credentials | Add `-U user:pass`; confirm you didn't use `-u` |
| `curl: (56) CONNECT tunnel failed, response 407` | Same 407, seen through an HTTPS CONNECT tunnel | Fix credentials, not TLS |
| Works with `-v` manually, fails in a script | Credentials not reaching the command, or a stale `https_proxy` in the environment | `env \| grep -i proxy`, then re-set explicitly |
| Authentication succeeds, target returns 403 | Proxy auth worked, the site rejected the proxy IP | Switch proxy type or rotate the session |
| Auth works over HTTP target, fails over HTTPS target | Special characters breaking the URL form | Use `-U` instead of embedding credentials in the URL |
| Sudden 407 after weeks of nothing | Traffic exhausted, IP whitelist entry changed, or credentials rotated | Check the dashboard, regenerate credentials |
| Credentials look right, still 407 | Auth scheme mismatch (Digest/NTLM wanted, Basic sent) | Read the `Proxy-Authenticate` header, add `--proxy-anyauth` |

Two habits make all of this faster. First, `-v` on every failing request — verbose output shows whether the failure is DNS, TCP connect, the proxy auth exchange, or the tunnel, instead of leaving you guessing from a three-digit code. Second, `--noproxy` or a `no_proxy` environment variable when you need a specific request to bypass the proxy, especially for `localhost` and internal hosts. A proxy configuration applied globally that reroutes internal traffic is a bad afternoon.

## Habits that keep curl jobs alive

Once authentication works, the next failure is usually operational rather than cryptographic.

bash
curl -x "http://user:pass@proxy.example:8080" \
     --connect-timeout 10 \
     --max-time 30 \
     --retry 3 --retry-delay 2 \
     -o output.html \
     "https://example.com/data"


A few things worth wiring in deliberately:

- **Timeouts on both connect and total.** A proxy that accepts the tunnel and then stalls will hang your script for as long as the OS allows.
- **Retries with a pause.** Rotating proxies fail transiently; three retries absorbs most of it without hammering the endpoint.
- **Fail closed, not open.** If proxy variables go unset in production, curl will happily connect directly, and your requests arrive from the wrong IP wearing the wrong identity. Explicitly set the proxy per command rather than relying on inherited environment variables for jobs where the exit IP matters.
- **Don't `-v` in CI with credentials in the URL.** Verbose output plus a URL containing a password equals a password in your build logs. Use `-U` and let the log redact the argument, or run verbose locally only.
- **Rotate sessions on purpose.** Rotating or sticky sessions are usually controlled by the username string or a session parameter, so a fresh session ID per job is a config change, not a code change.

## Where the credentials come from, and why the billing model matters here

All of the above assumes you have a working proxy to authenticate against. This is where a lot of first-time curl users burn an afternoon: buying a subscription-based proxy plan for a script that makes forty requests, then discovering the monthly bucket expired before the debugging was done.

DataImpulse takes the pay-as-you-go route instead, which fits curl-style work reasonably well. Residential traffic starts at **$1/GB**, datacenter at **$0.50/GB**, mobile at **$2/GB**, and premium residential at **$5/GB**. There's no subscription, and unused traffic doesn't expire — so a top-up you use for testing this week still works next month. You can start with $5 of residential traffic and find out whether your target actually tolerates the IPs before committing to anything larger.

Authentication-wise, the dashboard supports both username/password credentials and IP whitelisting, so `curl -U` works out of the box, and the whitelist option lets you skip credentials entirely for requests coming from a fixed server. Protocols cover HTTP, HTTPS and SOCKS5, and country targeting is included at the base rate, with city, ZIP and ASN targeting as paid add-ons. That last detail matters for curl work: if you only need a different country, you don't pay extra.

The pool is 90M+ IPs across 195+ locations, rotating and sticky sessions are both available (datacenter sticky sessions run up to 30 minutes), and support is human and 24/7 rather than a bot. The provider publishes a 99.51% success rate, which is their own number rather than an independent measurement — useful as a reference point, not as a guarantee.

## All current DataImpulse plans

Every plan below is pay-as-you-go with non-expiring traffic and no subscription requirement. Prices are the published rates at the time of writing and can change.

| Proxy type | Entry package | Volume tiers | Rate | Billing | Get started |
| --- | --- | --- | --- | --- | --- |
| Residential | $5 / 5 GB | 1 TB at $0.80/GB; 5 TB at $0.70/GB | $1/GB | Pay-as-you-go, traffic never expires | [Start with $5 of residential traffic](https://bit.ly/dataimPulse) |
| Datacenter | $5 / 10 GB | 100 GB for $50; 1 TB for $450 ($0.45/GB); 5 TB+ from $2,250 | $0.50/GB | Pay-as-you-go, traffic never expires | [Check datacenter proxy plans](https://dataimpulse.com/datacenter-proxies/?aff=86938) |
| Mobile (4G/5G) | $5 / 2.5 GB | 25 GB for $50; 1 TB for $1,600 ($1.60/GB); 5 TB+ from $8,000 | $2/GB | Pay-as-you-go, traffic never expires | [See mobile proxy pricing](https://bit.ly/dataimPulse) |
| Premium residential | $5/GB pay-as-you-go | High-speed pool, dedicated account manager, 99.9% uptime, all targeting included | $5/GB | Pay-as-you-go, traffic never expires | [Compare premium residential plans](https://dataimpulse.com/premium-residential-proxies/?aff=86938) |

For a curl script that just needs a different exit IP now and then, the datacenter tier at $0.50/GB is the cheap choice, with the caveat that datacenter ranges get blocked by anything with decent bot detection. Residential is the middle ground at $1/GB. Mobile is the expensive option you reach for only when a target genuinely requires a cellular IP.

## Matching the proxy type to what you're actually doing

- **SERP checks and public pages through curl** — datacenter. Fast, cheap, and most reference pages aren't defended.
- **Price monitoring on e-commerce sites** — residential. Retail anti-bot systems are specifically tuned to flag datacenter ranges.
- **Ad verification and geo-testing** — residential with country targeting included, and city targeting if the campaign is city-level.
- **Mobile app endpoints and hardest targets** — mobile proxies, but only for the requests that need them. Paying $2/GB to fetch a static page you could get at $0.50/GB is a self-inflicted budget problem.
- **Fixed-IP, high-volume jobs** — premium residential, where the account manager and the tighter uptime commitment start to matter more than per-GB price.

You can mix and match under a single account, which is practical: point most of your curl calls at the cheap tier and route only the defended ones through residential.

## Things people search after their first 407

**Does curl send proxy credentials on the first request?** Yes, when you specify a scheme explicitly with `-U`, curl inserts the `Proxy-Authorization` header on the initial request rather than waiting for a 407 challenge. With `--proxy-anyauth` it probes first instead.

**Why does the same command work in Postman but not in curl?** Almost always the URL-embedded credentials. Postman handles the encoding and the app sends the credentials separately; a raw shell passes the string through untouched.

**Can I use a proxy without a username and password at all?** Yes, if the provider supports IP whitelisting. You authorize your server's IP in the dashboard and then `curl -x http://proxy:port https://example.com` works with no credentials. The tradeoff is that the access is tied to that IP, so it breaks the moment you move servers or run from a laptop on a dynamic connection.

**Is it safe to put the proxy password in a URL?** It's functional but leaky. Shell history, process listings and verbose logs can all capture it. `-U` keeps it out of the URL; a `.curlrc` or a properly permissioned environment variable keeps it out of the command line.

**What does `curl: (5)` mean?** curl couldn't resolve the proxy hostname. That's a DNS or typo problem, not an authentication one — check the gateway address before you touch credentials.

## The one-command sanity check

Before you integrate anything into a larger script, run one request and read the output:

bash
curl -v -x "http://user:pass@proxy.example:8080" https://api.ipify.org


If you see the proxy's IP returned, the credentials, the transport and the session are all working. If you see a 407, it's the credentials. If you see a 403 or a captcha page, the credentials are fine and the IP is the problem — that's a proxy type question, not a curl question. Getting those three outcomes apart early saves a lot of aimless flag-swapping, and everything after that is just picking the tier that doesn't make your per-request cost embarrassing.

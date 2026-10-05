# wget proxy: Set http_proxy, https_proxy and .wgetrc, verify rotation, and fix the failures wget never warns you about

You export `http_proxy`, run wget, and it downloads straight from your own IP. No error. No warning. The file arrives, the exit code is 0, and nothing tells you the proxy was ignored.

That's the single most common way a wget proxy setup fails, and it's almost never the proxy's fault. Wget reads proxy settings from four different places, it only trusts one spelling of the environment variables, and it has no SOCKS5 support at all. Get any of those wrong and wget quietly does the wrong thing.

This guide covers the four configuration methods and how they override each other, the authentication quirks that break long downloads, how to point wget at a rotating residential gateway with working code, and how to diagnose a failed request from the exit code alone.

## The four places wget looks for proxy settings

Wget checks command-line options, your user config, the system config, and environment variables — in that order of authority. Later sources in the list lose to earlier ones.

| Priority | Where it lives | Scope | Beats |
| --- | --- | --- | --- |
| 1 (highest) | `-e` flags on the command line | Single command | Everything |
| 2 | `~/.wgetrc` | Your user account | System config and env vars |
| 3 | `/etc/wgetrc` | Every user on the machine | Env vars only |
| 4 (lowest) | `http_proxy` / `https_proxy` env vars | Current shell session | Nothing |

If you want to know which config file wget actually reads, check the `Wgetrc:` line in `wget --version` output. Homebrew builds often land on `/usr/local/etc/wgetrc`, most Linux package managers use `/etc/wgetrc`, and Windows builds vary enough that guessing is a waste of time.

The escape hatch is `--no-proxy`, which bypasses every proxy setting regardless of where it came from:

bash

wget --no-proxy https://internal-server.company.com/report.pdf



That's more useful than it sounds. On a shared server where a sysadmin already set a proxy in `/etc/wgetrc`, you don't have to edit system files to reach an internal host once.

## Method 1: environment variables (lowercase, or they do nothing)

This is the default way wget discovers a proxy, and it's where most people get burned:

bash

export http_proxy=http://gw.dataimpulse.com:823

export https_proxy=http://gw.dataimpulse.com:823

export ftp_proxy=http://gw.dataimpulse.com:823

export no_proxy=localhost,127.0.0.1,.internal.company.com



Wget reads **lowercase** variable names only. `HTTP_PROXY` in capitals is silently ignored — no warning, no error, wget resolves the target directly. This bites people in CI pipelines constantly, because secret managers and GitHub Actions conventions store variables in uppercase, and the runner then exposes them to the shell in uppercase too. If your step definition defines `HTTP_PROXY`, rename it.

Two more details worth knowing:

- Include the `http://` scheme even for `https_proxy`. The proxy endpoint itself is HTTP; wget opens a `CONNECT` tunnel through it to reach the HTTPS target. Recent wget versions will prepend the scheme for you and log `Prepended http:// to ...`, but relying on that is asking for version-specific behaviour.

- `no_proxy` is comma-separated, and a leading dot (`.company.com`) covers all subdomains.

Quick sanity check before you start a long job — this fetches your apparent IP through the proxy:

bash

wget -qO- -e use_proxy=on -e https_proxy=http://gw.dataimpulse.com:823 \

https://api.ipify.org/



If that prints your own IP, the proxy was not used and you should stop and fix it before queueing a 40 GB download.

## Method 2: ~/.wgetrc, the right place for credentials

Environment variables leak. They show up in `env` dumps, in process environments, and in shell history. A config file with locked permissions is better:

ini

use_proxy = on

http_proxy = http://USERNAME:PASSWORD@gw.dataimpulse.com:823

https_proxy = http://USERNAME:PASSWORD@gw.dataimpulse.com:823

no_proxy = localhost,127.0.0.1

proxy_user = USERNAME

proxy_password = PASSWORD



bash

chmod 600 ~/.wgetrc



Note the syntax difference: `.wgetrc` uses spaces around `=`, environment variables don't. Also note that `use_proxy = on/off` is the toggle to keep in mind — `off` disables proxying while leaving your settings intact, which is easier than commenting lines out and back in.

If you need a deterministic config path — Windows, containers, CI — pass it explicitly instead of hoping wget finds the right file:

bash

wget --config=./.wgetrc -qO- https://api.ipify.org/



## Method 3: one-off flags with -e

When you only need a proxy for a single request, override everything from the command line:

bash

wget -e use_proxy=on \

-e https_proxy=http://gw.dataimpulse.com:823 \

https://example.com/dataset.zip



Use `on` for the boolean. `wget: use_proxy: Invalid boolean 'true'; use 'on' or 'off'` is the error you'll get if you try `true` or `yes` on a build that doesn't accept it — `on` works everywhere and matches the manual.

One correction worth making, because it's copied around a lot: you'll see tutorials using `wget --proxy=http://host:port`. That isn't the documented form. `--proxy` takes `on` or `off`. The proxy address goes in `http_proxy`/`https_proxy`, either as an environment variable or via `-e`.

## Proxy authentication: Basic only, and where your password ends up

Wget implements exactly one proxy authentication scheme: HTTP Basic. There is no NTLM, no Kerberos, no digest, no PAC-file support. Corporate networks using PAC scripts or NTLM will fail here, and the fix isn't a wget flag — it's a local relay like Cntlm (or `px` on Windows) that speaks NTLM upstream and presents Basic auth to wget on `127.0.0.1`.

For normal username/password proxies, you have two options:

bash

# Embedded in the proxy URL

wget -e use_proxy=on \

-e http_proxy=http://USER:PASS@gw.dataimpulse.com:823 \

https://example.com/file.zip

# Or with dedicated flags

wget --proxy-user=USER --proxy-password=PASS \

-e use_proxy=on -e http_proxy=http://gw.dataimpulse.com:823 \

https://example.com/file.zip



The `--proxy-user` and `--proxy-password` flags override anything embedded in the URL. Both approaches leak something somewhere: passwords on the command line are visible to `ps` on a shared box, and URL-embedded credentials end up in shell history unless you disable history or read the password with `read -rs`.

If your password contains `@`, `:`, `/` or `?`, percent-encode it before putting it in a proxy URL. Otherwise the URL parser splits at the wrong character and you get an authentication failure that looks like a wrong password.

> Basic auth is base64, not encryption. Over a plain HTTP connection to the proxy, your proxy credentials are effectively in the clear on the local network segment. On a shared or untrusted network, that matters.

## The SOCKS5 problem nobody mentions

GNU Wget 1.x supports HTTP, HTTPS and FTP proxies. The word "SOCKS" does not appear in its manual. There is no `--socks5` flag, no `socks_proxy` variable, and no `.wgetrc` directive for it. If a tutorial shows you `wget --socks5-hostname=host:port`, that command does not exist in the tool you're running.

If you have a SOCKS5-only endpoint, you have three realistic options:

1. **Use curl instead.** It handles `socks4://`, `socks5://` and `socks5h://` natively, and `socks5h` sends the hostname to the proxy for resolution, which avoids leaking DNS queries locally.

2. **Wrap wget in proxychains4**, which intercepts network calls at the library level and routes them through SOCKS. It works, with the caveat that it only affects dynamically linked programs.

3. **Use the provider's HTTP endpoint.** Most commercial services expose both, and the HTTP port is far simpler.

That third option is the practical one. DataImpulse, for example, publishes its rotating gateway on **port 823 for HTTP/HTTPS** and **port 824 for SOCKS5**. For wget you want 823, and the 824 endpoint is there for the tools that can actually use it.

## Pointing wget at a rotating residential gateway

Here's the concrete version. After you [👉 sign up and grab the $5 starter pack](https://bit.ly/dataimPulse), the dashboard gives you a host, a port, a username and a password. The gateway is `gw.dataimpulse.com`, and rotating traffic runs on 823:

bash

wget -qO- \

-e use_proxy=on \

-e https_proxy=http://USERNAME:PASSWORD@gw.dataimpulse.com:823 \

https://api.ipify.org/



Run it twice. With a rotating endpoint you should see a different exit IP each time. If you see the same IP twice, check whether you've accidentally hit a sticky port.

Country targeting goes in the username, using the `__cr.{country_code}` suffix documented in DataImpulse's own docs:

bash

# German exit IPs

export https_proxy="http://USERNAME__cr.de:PASSWORD@gw.dataimpulse.com:823"

# Pinning requests to a specific session label, still on port 823

export https_proxy="http://USERNAME__cr.us;sessid.42:PASSWORD@gw.dataimpulse.com:823"



Sticky connections use ports **10000 to 20000** instead of 823. The rotation interval is configurable from 1 to 120 minutes, with roughly 30 minutes as the realistic average. That average is a function of how residential pools work: these are real user connections, and when the peer goes offline the session rotates to the next available IP whether you wanted it to or not. If your download job needs a fixed IP for over half an hour, plan for a mid-transfer rotation rather than assuming stability.

Putting it in `.wgetrc` for repeated jobs:

ini

use_proxy = on

http_proxy = http://USERNAME__cr.de:PASSWORD@gw.dataimpulse.com:823

https_proxy = http://USERNAME__cr.de:PASSWORD@gw.dataimpulse.com:823

proxy_user = USERNAME__cr.de

proxy_password = PASSWORD

no_proxy = localhost,127.0.0.1



Authentication on DataImpulse works either with username/password or with an IP allowlist, so on a fixed server you can skip credentials entirely and whitelist the machine's IP in the dashboard.

## What DataImpulse's four proxy types cost

DataImpulse runs a pay-as-you-go model: no subscription, and purchased traffic doesn't expire. Prices below come from the published plan tiers.

| Proxy type | Plan | Traffic | Price | Per GB |
| --- | --- | --- | --- | --- |
| Residential | Intro | 5 GB | $5 | $1.00 |
| Residential | Basic | 50 GB | $50 | $1.00 |
| Residential | Advanced | 1 TB | $800 | $0.80 |
| Residential | Custom | 5 TB+ | From $4,000 | Negotiable |
| Datacenter | Intro | 10 GB | $5 | $0.50 |
| Datacenter | Basic | 100 GB | $50 | $0.50 |
| Datacenter | Advanced | 1 TB | $450 | $0.45 |
| Datacenter | Custom | 5 TB+ | From $2,250 | Negotiable |
| Mobile | Intro | 2.5 GB | $5 | $2.00 |
| Mobile | Basic | 25 GB | $50 | $2.00 |
| Mobile | Advanced | 1 TB | $1,600 | $1.60 |
| Mobile | Custom | 5 TB+ | From $8,000 | Negotiable |
| Premium Residential | Intro | 1 GB | $5 | $5.00 |
| Premium Residential | Basic | 10 GB | $50 | $5.00 |
| Premium Residential | Advanced / Custom | 1 TB+ | From $4,000 | From $4.00 |
| Premium Residential | Custom | 5 TB+ | From $20,000 | Negotiable |

Buy links for every tier point to the same account signup, since plans are configured as a GB quantity on your balance rather than as separate checkout pages: [👉 5 GB residential for $5](https://bit.ly/dataimPulse), [👉 10 GB datacenter for $5](https://bit.ly/dataimPulse), [👉 2.5 GB mobile for $5](https://bit.ly/dataimPulse), [👉 1 GB premium residential for $5](https://bit.ly/dataimPulse), [👉 the 1 TB residential tier at $0.80/GB](https://bit.ly/dataimPulse), [👉 the 1 TB datacenter tier at $0.45/GB](https://bit.ly/dataimPulse), [👉 the 1 TB mobile tier](https://bit.ly/dataimPulse), and [👉 custom 5 TB+ volume pricing](https://bit.ly/dataimPulse).

A few things that aren't in the table rows:

- **Country targeting is included.** City, ZIP and ASN filtering are listed as paid add-ons, and at least one third-party review reports advanced filters billed at roughly double the standard per-GB rate on residential plans. Budget-sensitive jobs should confirm current treatment with support before assuming city-level targeting is free.

- **The minimum spend is $5.** There's no free tier and no free trial. Intro plans come with a 7-day money-back guarantee on card payments, provided you've consumed less than 80% of the traffic. Crypto purchases on Intro plans aren't refundable.

- **Pool and protocols.** 90M+ IPs across 195 countries, sourced through DataImpulse's own opt-in app rather than resold from other providers. HTTP, HTTPS and SOCKS5 all work, and the company publishes a 99.51% success rate.

## Choosing a plan for wget work

Wget isn't browser automation. It pulls files. That changes the maths.

**Datacenter at $0.50/GB** covers anything that doesn't inspect IP reputation — public datasets, documentation mirrors, government open data, CDN testing. A 5 GB datacenter pack is 10 GB of traffic, and at typical HTML page sizes that's a lot of requests.

**Residential at $1/GB** is the one you want when the target blocks server ranges. If you're downloading from sites that gate on IP type, the extra $0.50/GB buys access, not speed. The first 5 GB pack is a reasonable way to measure your own success rate against your actual targets before scaling.

**Mobile at $2/GB** is for targets that specifically distrust non-cellular traffic. If your wget job is pulling app-store assets or mobile-web endpoints, that's the case for it. For plain file downloads it's usually wasted budget.

**Premium residential at $5/GB** is priced for teams whose block rate genuinely costs them money, and it comes with all targeting options and a dedicated account manager. If you're running wget against a target where a 90% success rate isn't good enough, it's worth testing. If you're downloading tarballs, it's overkill.

Traffic never expiring is the quiet advantage for intermittent jobs. Buy 50 GB, use 10 GB this month and the rest in two months, and nothing resets. Subscription providers bill you for the GBs you didn't use, which is why the headline rate isn't the real rate.

## Debugging: read the exit code, not the log

Wget's exit codes are more specific than most CLI tools, and they narrow the problem faster than scrolling through verbose output.

| Code | Meaning | Proxy-related cause |
| --- | --- | --- |
| 1 | Generic error | — |
| 2 | Parse error | Malformed `.wgetrc` or command-line option |
| 3 | File I/O error | Disk or permissions |
| 4 | Network failure | Can't reach the proxy host/port |
| 5 | SSL verification failure | Proxy intercepting TLS, or untrusted CA |
| 6 | Auth failure | Wrong credentials, or an un-updated IP allowlist |
| 7 | Protocol error | Wrong scheme for the target |
| 8 | Server error response | You reached the target; it rejected you |

Lower-numbered codes take precedence when several errors occur at once, so an exit 4 can mask a credentials problem behind it.

`echo $?` after a failing run tells you which of these you're in. Exit 4 usually means the host or port is wrong — verify with `nc -zv gw.dataimpulse.com 823`. Exit 6 means the proxy answered and said no: check the password, and check whether you're relying on IP whitelisting from a machine whose IP changed.

For the silent-ignore case, where everything succeeds but the proxy clearly wasn't used:

bash

# 1. Did I set it in the shell?

env | grep -i proxy

# 2. Is a stale value hiding in a config file?

wget --no-config --spider http://example.com/

# 3. Which config file is being read?

wget --version | grep -i wgetrc



If `--no-config` fixes it, the problem is in a config file, not in your command. Stale proxy lines left behind by a container image or an old admin are a common cause of `Connection refused` errors that point at an IP you've never heard of.

## Making wget behave on real targets

Rotation alone doesn't make a download job look human. Three flags do most of the work:

bash

wget --wait=2 --random-wait \

--user-agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36" \

--tries=3 --timeout=20 \

-c --limit-rate=2m \

-o download.log \

-i urls.txt



- `--random-wait` jitters the delay so your request timing isn't a metronome. `--wait` alone produces a perfectly regular pattern, which is its own signal.

- `--user-agent` matters because wget's default UA is literally `Wget/1.21.2`, and anti-bot rules target it directly.

- `-c` resumes interrupted transfers. With a rotating proxy that occasionally swaps IPs mid-job, resume support turns a failure into a retry instead of a restart.

- `-o` writes a log you can actually review afterwards. Add `-d` when you need to see what the proxy handshake is doing.

One note on crawling: check the target's `robots.txt` before running recursive downloads. Rotating proxies make large-scale fetching technically easy, which doesn't make it permitted.

## FAQ

**Does wget support SOCKS5 proxies?**

No. GNU Wget 1.x handles HTTP, HTTPS and FTP proxies only. Use curl, proxychains4, or an HTTP endpoint from your provider. Wget2 is a separate rewrite and SOCKS5 isn't a standard documented option there either.

**Why is my proxy setting ignored with no error at all?**

Almost always uppercase variable names. Wget reads `http_proxy` and `https_proxy` in lowercase and silently ignores `HTTP_PROXY`.

**Can wget rotate IPs on its own?**

No. It has no built-in rotation. Either use a provider that rotates server-side so every request through the same gateway gets a fresh exit IP, or script a random pick from a proxy list into `-e http_proxy=...` before each call.

**Is `https_proxy` supposed to be an `https://` URL?**

Usually `http://`. The proxy endpoint is HTTP and wget tunnels HTTPS through it via `CONNECT`. Setting `https_proxy=https://...` for an HTTP target throws `Error in proxy URL ...: Must be HTTP`.

**How long can a sticky session last?**

DataImpulse lets you configure 1 to 120 minutes, with about 30 minutes as the average in practice. Residential IPs come from real devices, so a peer going offline rotates the session early.

---

If your wget downloads are landing on your own IP with no warning, the fix is nearly always a capitalised variable name or a stale config line — check those before you change providers. If the configuration is correct and you're still getting blocked, the IP type is the variable, and that's a proxy decision rather than a wget one. The $5 entry point is enough traffic to test your actual targets and find out which one you're dealing with: [👉 start with DataImpulse at $1/GB, traffic that doesn't expire](https://bit.ly/dataimPulse).

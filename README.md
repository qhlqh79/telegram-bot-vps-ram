# Telegram bot VPS: how much RAM does your bot need, polling or webhook, and what 24/7 hosting really costs

A text-only Telegram bot sits at a few hundred megabytes of RAM. A 1 GB server billed at around $2 a month keeps it answering updates around the clock, and for most projects that is the entire hardware story. The decisions that actually cost people money sit elsewhere: polling versus webhook, what happens to the process after a crash, and whether the billing model can quietly take the machine away.

SixTNet's VPS line starts at **$200.00 a year for 2 cores and 2 GB**, so it is not the budget answer to that first question. What it sells is a different IP type and a China-optimised network route. That matters enormously to a narrow set of bots and not at all to the rest. The table and the RAM math below should make it clear which side you are on.

> **Running one or two ordinary text bots, buying from SixTNet:** LAX-STARTER, 2 cores / 2 GB / 40 GB NVMe / 5 TB traffic at **$200.00 a year** (list $240.00). Two gigabytes is already more than a text bot needs, and 5 TB of traffic is far beyond what message polling consumes.
>
> **Bot that logs into other services, scrapes, or gets blocked from datacenter IPs:** Residential-STARTER, **$260.00 a year** for the same hardware with a residential IP. Ports 80 and 443 are closed on that line, so plan on polling or confirm 88 and 8443 with support first.
>
> 👉 [See SixTNet's full VPS plan list and current pricing](https://sixtynet.net/aff.php?aff=39)

## SixTNet VPS plans as listed on the site

| Plan | CPU / RAM | Storage / traffic | Price (billed annually) | Order |
| --- | --- | --- | --- | --- |
| LAX-STARTER | 2 cores / 2 GB | 40 GB NVMe / 5 TB | $200.00 USD (list $240.00, save 17%) | [ Order LAX-STARTER](https://sixtynet.net/aff.php?aff=39) |
| LAX-MICRO | 4 cores / 4 GB | 80 GB NVMe / 10 TB | $400.00 USD (list $480.00, save 17%) | [ Order LAX-MICRO](https://sixtynet.net/aff.php?aff=39) |
| LAX-MEDIUM | 8 cores / 8 GB | 160 GB NVMe / 20 TB | $800.00 USD (list $960.00, save 17%) | [ Order LAX-MEDIUM](https://sixtynet.net/aff.php?aff=39) |
| LAX-LARGE | 8 cores / 16 GB | 320 GB NVMe / 40 TB | $1,600.00 USD (list $1,920.00, save 17%) | [ Order LAX-LARGE](https://sixtynet.net/aff.php?aff=39) |
| Residential-STARTER | 2 cores / 2 GB | 40 GB NVMe / 5 TB | $260.00 USD (list $300.00, save 13%) | [ Order Residential-STARTER](https://sixtynet.net/aff.php?aff=39) |
| Los Angeles China Unicom-Premium | Not listed | 100M base, 3x burst, 95th percentile billing | $2,500.00 USD (list $3,000.00, save 17%) | [ Order China Unicom-Premium](https://sixtynet.net/aff.php?aff=39) |

What the table doesn't show, from the same page:

- The four LAX plans include DDoS protection and a 2–10 Gbps uplink depending on size.
- **Residential IP products do not open ports 80 and 443.** The stated reason is IP risk control, and it applies to every residential configuration, not just the starter.
- A second residential line exists with AS9929 routing for China. Those plans reject external probes and scans, and access is restricted unless you configure it, so whitelisted or port-forwarded devices only. The page also states that refunds on that line are issued as website balance.
- Four-, eight- and sixteen-core residential configurations are listed further down the page. Their price fields weren't confirmed, so check them on the live page before you order.
- Every price above is annual. A monthly option isn't shown for these plans.

## How much RAM does a Telegram bot actually use?

Hosting providers that sell bot servers publish their own figures, and they cluster tightly: 50–150 MB per bot at idle with spikes under load, 256–512 MB for most Python and Node.js bots, and "well under a few hundred megabytes" for a plain text bot. None of these are Telegram specifications. Treat them as vendor rules of thumb and plan with the higher end.

A workable conversion, again from vendor guidance rather than measurement:

- **1 GB** runs 3–4 small text bots with room for the OS.
- **2 GB** covers 10–15 of them, or one bot with PostgreSQL and Redis attached.
- **4 GB** handles 25 or more, or a bot with a web admin panel and background workers.

What breaks those numbers is not message volume. It is what the bot does between messages. A bot that wraps an LLM wants closer to 2 GB on its own because of the memory used while processing a response. Image resizing, voice transcription and OCR all spike the same way. If you plan to run Telegram's local Bot API server, one host reports it running comfortably on 4 GB, raising the upload ceiling to 2 GB files and removing some rate limits, which is the usual reason media-heavy bots bother with it. TDLib, if you need it, compiles from source after installing `build-essential cmake gperf libssl-dev zlib1g-dev` and running `./build.sh`.

Against all that, SixTNet's smallest option is already 2 GB. RAM will not be what limits you there.

## Polling or webhook on a telegram bot VPS?

Polling is the default in every library and it needs nothing from the server. The bot opens a connection and asks Telegram for updates in a loop. No domain, no certificate, no open inbound port. You can run it on a fresh VPS thirty seconds after it boots.

Webhook flips the direction. Telegram delivers each update over HTTPS to a URL you register, which means you need a domain pointed at the server, a certificate, and a reverse proxy such as nginx or Caddy in front of your app. Telegram only delivers to **ports 443, 80, 88 or 8443**, which is the number to remember when a webhook silently never arrives. In exchange you get lower latency, less CPU spent on the polling loop, and a public endpoint that payment providers can call back to.

The rule that covers most cases: no domain yet, still getting the bot to work → polling. Public bot, taking payments, or expecting steady traffic → webhook. Discord bots are a separate matter, since their libraries always hold a gateway connection.

Here the SixTNet residential line becomes a real constraint. Closed ports 80 and 443 rule out the standard setup of a domain on 443 with a Let's Encrypt certificate. Ports 88 and 8443 may still be usable, but that isn't stated on the page, so ask before you buy rather than after.

👉 [Check which SixTNet plans leave ports 80 and 443 open](https://sixtynet.net/aff.php?aff=39)

## Three ways to keep the bot running after a crash

Starting a bot from an SSH session is fine once. It dies the moment you disconnect. For anything real, the process has to be supervised, and there are three common answers.

**systemd** is built into every Linux image and needs one file. The parts that matter: `WorkingDirectory=/opt/bot`, `EnvironmentFile=/opt/bot/.env`, `ExecStart=/opt/bot/.venv/bin/python main.py`, `Restart=always`, `RestartSec=5`. Enable it with `systemctl enable --now tg-bot` and watch it with `journalctl -fu tg-bot`. It restarts on crash and on reboot with no extra tooling.

**pm2** is the Node.js habit. It keeps a process list, captures logs, and survives reboots after `pm2 startup`. Useful if you already run several Node services and want one dashboard.

**Docker** gives each bot its own container, its own dependencies and `--restart=unless-stopped`. Updates become `git pull` plus a rebuild, and rolling back means running the previous tag. The cost is a little overhead and one more thing to learn.

Whichever you choose, run one service per bot with separate directories and separate `.env` files, so a bad deploy of one doesn't take the other three down.

Two operational details that bite people later. Keep the bot token out of the code and out of Git; put it in an environment file readable only by the bot user. And when you set up the firewall, allow SSH plus HTTPS only. Outbound port 25 is closed by default on most providers, so password resets and notifications should go through an HTTPS email API rather than raw SMTP.

## Why free hosting isn't a home for a bot

Free application platforms sleep the process after a period of inactivity, which means the bot misses messages until something wakes it. Several wipe the local disk on every restart, taking SQLite state and conversation memory with it. Others hand out a new address on each deploy, so a webhook URL breaks each time you push. A VPS has none of those behaviours: no sleep, persistent disk, a stable IPv4 that a domain can point at.

## Billing terms to confirm before paying

SixTNet shows annual pricing on this page, with list prices struck through and a discount applied. That structure raises three questions worth answering before checkout, because none of them are settled in the plan list:

1. **What does renewal cost?** The discounted annual figure is the one displayed. Whether the same discount applies at renewal isn't stated.
2. **If a renewal is late, how long is the data kept?** Prepaid-balance providers in this market commonly suspend a server when the balance hits zero and delete it after a fixed window, sometimes 24 hours, with a single low-balance email as warning. That is a competitor's published policy, not SixTNet's, but it is the exact clause to ask about.
3. **How are refunds paid?** One SixTNet residential line explicitly refunds to website balance rather than to the original payment method. Ask which rule applies to the plan you're buying.

👉 [Open SixTNet's VPS page and check the terms on the plan you want](https://sixtynet.net/aff.php?aff=39)

## Where SixTNet sits on price

For pure bot hosting, it is not the cheap seat. Verified entry prices elsewhere in this market: a 1 GB KVM server at **$1.81/mo** and a 2 GB at **$3.54/mo** from DataPasa, both hourly-billed; OliveVPS at **$7.99/mo** for 1 GB and **$15.99/mo** for 2 GB; Infrawire at **€8.30/mo** for 4 cores and 4 GB. SixTNet's LAX-STARTER works out to about **$16.67 a month** for 2 GB, billed a year at a time.

That gap buys a residential IP option, an AS9929 route into China, and DDoS filtering on the LAX line. If your bot only exchanges text with Telegram, none of those change your result, and the budget tier does the same job.

## Which plan to pick

- **One text bot, nothing else:** SixTNet's LAX-STARTER, though any 1 GB server at a third of the price would also do. On SixTNet's lineup 2 GB is simply the floor.
- **Bot plus PostgreSQL or Redis, or a webhook behind Caddy:** LAX-STARTER still fits at 2 GB. Move to LAX-MICRO if you add background workers, a mini app, or more than a handful of bots.
- **Local Bot API server, media processing, or LLM calls:** 4 GB and up, which means LAX-MICRO. A 1 GB box will get killed at the first memory spike.
- **The bot needs a residential IP to reach another service:** Residential-STARTER at $260 a year, with polling rather than a webhook on 443.

Before you pay for anything, confirm the two things this page can't tell you: which ports your chosen plan actually leaves open, and how refunds are returned. Those two answers decide whether a cheap plan is cheap and whether a residential IP is usable at all.

👉 [Start with SixTNet's LAX-STARTER at $200.00 a year](https://sixtynet.net/aff.php?aff=39)

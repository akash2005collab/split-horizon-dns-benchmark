# BIND9 vs dnsmasq — Comparison Report

Same test conditions for both: same 8-line query file (mix of real records and intentionally non-existent names), same 10-second dnsperf run, same VM, run back to back.

## Split-horizon support

| | BIND9 | dnsmasq |
|---|---|---|
| Split-horizon (different answer by client source) | Yes — implemented via `internal`/`external` views | No — not supported at all |

Verified with dig:
```
dig @127.0.0.1 app.company.test   → 10.10.10.10   (internal)
dig @10.0.2.15 app.company.test   → 203.0.113.10  (external)
```
dnsmasq returns `203.0.113.10` to every client regardless of source — there's no view/ACL mechanism in dnsmasq to replicate this.

## Throughput and latency (dnsperf, 10s run)

| Metric | BIND9 (port 53) | dnsmasq (port 5354) |
|---|---|---|
| Queries completed | 387,230 (100%) | 118,245 (100%) |
| Queries lost | 0 | 0 |
| Queries per second | 38,701.60 | 11,811.72 |
| Average latency | 2.509 ms | 8.335 ms |
| Latency std. deviation | 1.012 ms | 8.731 ms |
| Response codes | 62.5% NOERROR / 37.5% NXDOMAIN | 25% NOERROR / 75% NXDOMAIN |

![Queries per second](graphs/qps.png)
![Average latency](graphs/latency.png)

BIND9 handled about 3.3x more queries per second than dnsmasq under the same load, with lower and more consistent latency. The response-code split differs because BIND9 has two real records configured (`ns`, `app`) against dnsmasq's one (`app`), so a larger share of the same query file returns NXDOMAIN on dnsmasq — that's a config difference, not an error on either side.

## Cache behavior (dig, cold vs warm query)

| Server | Query type | 1st query (cold) | 2nd query (warm) |
|---|---|---|---|
| BIND9 | Own zone record (`app.company.test`) | 0 ms | 0 ms |
| dnsmasq | Own static record (`app.company.test`) | 0 ms | 0 ms |
| BIND9 | External/recursive (`www.wikipedia.org`) | 880 ms | 0 ms |
| dnsmasq | External/forwarded (`www.github.com`) | 71 ms | 0 ms |

![Cache warming](graphs/cache.png)

Authoritative/static records are answered instantly on both servers regardless of repetition, since the data is already in memory — no caching effect is possible or needed there. Real cache warming only shows up on recursive/forwarded lookups: both servers took real time on the first external query and then answered instantly on the repeat, confirming both cache correctly once there's an actual external lookup involved.

## Bottom line

- **Split-horizon:** only BIND9 supports it.
- **Throughput/latency:** BIND9 outperforms dnsmasq under identical load.
- **Cache behavior:** both cache correctly for recursive lookups; neither shows a caching effect on purely authoritative/static answers, since there's nothing to warm up in that case.

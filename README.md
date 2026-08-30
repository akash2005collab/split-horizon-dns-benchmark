# Split-Horizon DNS with BIND9 vs dnsmasq — Performance Benchmarking Under Query Load

A Wipro 5G capstone project. Builds split-horizon DNS on BIND9 (same domain name resolves differently for internal vs external clients), replicates what's possible on dnsmasq, and benchmarks both under identical query load using dnsperf and dig.

## What's in this repo

```
configs/
  named.conf.local              BIND9 views/ACLs for split-horizon
  db.internal.company.test      Internal zone file
  db.external.company.test      External zone file
  dnsmasq.conf                  dnsmasq config (port 5354, static record)
graphs/
  qps.png                       Throughput comparison
  latency.png                   Latency comparison
  cache.png                     Cache warming comparison
screenshots/
  ...                           dig / dnsperf terminal output
comparison-report.md            BIND9 vs dnsmasq comparison (data + graphs)
PROJECT-REPORT.md               Full write-up of how the project was built
README.md                       This file
```

## Stack

BIND9, dnsmasq, dnsperf, dig — all run on a single Ubuntu Server 22.04.5 LTS VM (VirtualBox, NAT networking).

## Setup

1. Install packages:
   ```
   sudo apt install bind9 dnsmasq dnsutils
   ```
   (dnsperf isn't in default apt repos on all versions — install from source or the `dnsperf` package if available for your release.)

2. Copy the zone files from `configs/` into `/etc/bind/zones/` and `named.conf.local` into `/etc/bind/`. Check the zone files with:
   ```
   named-checkzone company.test /etc/bind/zones/db.internal.company.test
   named-checkzone company.test /etc/bind/zones/db.external.company.test
   ```
   Restart BIND9:
   ```
   sudo systemctl restart bind9
   ```

3. Copy `dnsmasq.conf` settings into `/etc/dnsmasq.conf` (or merge the `port=5354` and `address=` lines into your existing config). Restart dnsmasq:
   ```
   sudo systemctl restart dnsmasq
   ```

4. Confirm both are running:
   ```
   systemctl status bind9
   systemctl status dnsmasq
   ```

## Usage

**Verify split-horizon works:**
```
dig @127.0.0.1 app.company.test        # internal view → 10.10.10.10
dig @<your-external-facing-IP> app.company.test   # external view → 203.0.113.10
```

**Verify dnsmasq's static record:**
```
dig @127.0.0.1 -p 5354 app.company.test   # → 203.0.113.10 (same for every client)
```

**Run the load test** (create a query file first, one `<name> <type>` per line):
```
dnsperf -s 127.0.0.1 -p 53   -d queries.txt -l 10   # against BIND9
dnsperf -s 127.0.0.1 -p 5354 -d queries.txt -l 10   # against dnsmasq
```

**Check cache behavior** — query the same external domain twice in a row and compare the `Query time` in the dig output:
```
dig @127.0.0.1 www.wikipedia.org
dig @127.0.0.1 www.wikipedia.org
```

## Results

Full numbers and graphs are in [`comparison-report.md`](comparison-report.md). Short version:

- **Split-horizon:** BIND9 supports it natively via views. dnsmasq has no equivalent — it always returns the same answer regardless of client source.
- **Throughput:** BIND9 handled ~38,700 queries/sec vs dnsmasq's ~11,800 under the same 10-second load test — about 3.3x more.
- **Latency:** BIND9 averaged 2.5ms with tight consistency; dnsmasq averaged 8.3ms with much higher variance.
- **Cache behavior:** Both servers answer their own authoritative/static records instantly regardless of repetition (nothing to cache). For genuinely external/recursive lookups, both cache correctly — first query took real time (880ms on BIND9, 71ms on dnsmasq), repeat query came back in 0ms on both.

See [`PROJECT-REPORT.md`](PROJECT-REPORT.md) for the full account of how this was built, including the config decisions and issues worked through along the way.

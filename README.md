# vpn-client-config-inventory

> Extract proxy / VPN endpoints and subscription URLs from desktop client configuration on disk — then normalise, deduplicate and archive them into one auditable table.

[![release](https://img.shields.io/badge/release-v1.2.0-blue?style=flat-square)](https://github.com/ShiyuZeng/vpn-client-config-inventory-skill/releases)
[![python](https://img.shields.io/badge/python-3.8%2B-3776ab?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![platform](https://img.shields.io/badge/platform-Windows%2010%20%7C%2011-0078d4?style=flat-square&logo=windows&logoColor=white)]()
[![deps](https://img.shields.io/badge/dependencies-stdlib%20only-success?style=flat-square)]()
[![network](https://img.shields.io/badge/network-subscription%20fetch%20only-lightgrey?style=flat-square)]()
[![license](https://img.shields.io/badge/license-MIT-green?style=flat-square)](LICENSE)

**English** | [简体中文](README%EF%BC%88CH%EF%BC%89.md)

---

## Table of contents

- [TL;DR](#tldr)
- [Why this exists](#why-this-exists)
- [What changed in v1.2.0](#what-changed-in-v120)
- [Architecture](#architecture)
- [Installation](#installation)
- [Quick start](#quick-start)
- [CLI reference](#cli-reference)
- [Phase 0 — identify the stack](#phase-0--identify-the-stack)
- [Phase 1 — find the subscription URL](#phase-1--find-the-subscription-url)
- [Phase 2 — parse the returned YAML](#phase-2--parse-the-returned-yaml)
- [Phase 3 — normalise, merge, verify](#phase-3--normalise-merge-verify)
- [Output layout](#output-layout)
- [Validation record](#validation-record)
- [Failure modes](#failure-modes)
- [Design principles](#design-principles)
- [Repository layout](#repository-layout)
- [Version history](#version-history)
- [Scope](#scope)
- [License](#license)

---

## TL;DR

Three artefacts come out of one run, in this order of value:

| # | Artefact | What it answers |
|---|---|---|
| 1 | `subscriptions.csv` | Where each client's subscription lives, whether it still answers, plan quota and expiry |
| 2 | `node_list.csv` | One row per `host + port` — 节点IP或域名 \| 端口 \| 提取时间 \| 来源机场 |
| 3 | `yaml/<Airport>.yaml` | A loadable mihomo / sing-box profile per client, plus the untouched raw extract |

The data is almost always sitting in a local SQLite file, in plaintext, next to an encrypted YAML cache. Read the SQLite, request the URL, parse the response. Reverse-engineering the encryption or dumping process memory is the **last** resort and is usually unnecessary — that single insight is the difference between a 10-minute run and a 3-hour one.

```bash
python scripts/find_subscriptions.py --scan --probe --out subs   # Phase 1
python scripts/yaml_to_nodes.py --batch subs/yaml nodes/         # Phase 2
python scripts/build_node_list.py --root nodes --out node_list   # Phase 3
```

```text
布局: pair  校验: 571 行 / 571 个端口 / 唯一 host 86 / 有效端点 571
```

---

## Why this exists

v1.0.0 of this skill started at the middle of the pipe: it consumed `*_nodes.json` dumps that something else had already produced. In practice that "something else" was the hard part — every client stores its profile differently, and half of them store it encrypted.

v1.2.0 covers the whole pipe: **locate the client → locate its SQLite config → locate the subscription URL → fetch → parse → normalise → verify → archive**. It also encodes the fingerprints needed to tell which of the two dominant desktop stacks you are looking at before touching a single file, because the answer determines where the config lives and what the core binary is called.

Every rule in this repository was paid for by a failure in a real run. The sections below say which failure, so you can recognise it instead of repeating it.

---

## What changed in v1.2.0

| Area | v1.0.0 | v1.2.0 |
|---|---|---|
| Entry point | Consumes pre-made `*_nodes.json` | Starts at the installed client |
| Subscription discovery | — | `find_subscriptions.py`, 5-step decision order |
| Stack identification | — | Flutter + mihomo fork vs Flutter + sing-box/hiddify DLL |
| YAML parsing | Hand-rolled per run | `yaml_to_nodes.py`, three `proxies:` shapes, no third-party deps |
| Layout | One row per host, ports merged | `--layout pair` (default) or `merged` |
| Notice-row ports | Merged into the table | Dropped unless a real node also uses the pair |
| Ad-named real nodes | Manual rename before merging | Two-stage matcher inside the script |
| Encryption | In scope | Explicitly out of scope, documented as a dead end |
| Failure catalogue | 6 rows | 8 rows, each with the diagnosis that actually finds it |

Full changelog: [RELEASE_v1.2.0.md](RELEASE_v1.2.0.md).

---

## Architecture

```text
┌─ Phase 0 ──────────────┐   fingerprint the install directory
│  Stack A / Stack B     │   → tells you where config lives, what the core is called
└───────────┬────────────┘
            │
┌─ Phase 1 ─▼────────────┐   %APPDATA%\..\database.sqlite :: profiles.url
│  subscription URL      │   → 5-step decision order, first hit wins
└───────────┬────────────┘
            │  HTTPS GET, UA = clash-verge/v2.0.0
┌─ Phase 2 ─▼────────────┐   YAML / data: URI / expanded outbounds[]
│  *_nodes.json          │   → {name, type, server, port}
└───────────┬────────────┘
            │
┌─ Phase 3 ─▼────────────┐   normalise → dedup → subset suppression → verify
│  node_list.csv/.json   │   → 节点IP或域名 | 端口 | 提取时间 | 来源机场
│  REPORT.md  yaml/      │   → drop-reason audit log + archived profiles
└────────────────────────┘
```

Each phase is a separate script with a file contract between them. You can re-run any phase alone; you can hand-write the input of Phase 3; you can skip Phase 1 entirely if you already have the YAML.

---

## Installation

### As a skill

```text
~/.workbuddy/skills/vpn-client-config-inventory/     (WorkBuddy)
~/.claude/skills/vpn-client-config-inventory/        (Claude Code compatible layout)
```

Unzip the release asset so that `SKILL.md` sits at the directory root:

```text
vpn-client-config-inventory/
├── SKILL.md
├── references/api_reference.md
└── scripts/{find_subscriptions,yaml_to_nodes,build_node_list}.py
```

### As standalone scripts

Nothing to install. Python 3.8+, standard library only — no `pyyaml`, no `requests`, no `pycryptodome`. Copy `scripts/` anywhere and run it.

```bash
python --version   # 3.8+
```

---

## Quick start

```bash
# 1. what is installed, and does the subscription still answer?
python scripts/find_subscriptions.py --scan --probe --out subs

# 2. turn the fetched YAML into the Phase-3 input contract
python scripts/yaml_to_nodes.py --batch subs/yaml nodes/

# 3. merge, dedup, verify, archive
python scripts/build_node_list.py --root nodes --out out/node_list
```

`find_subscriptions.py` prints one line per profile:

```text
app           label      kind        status     nodes hosts  url
yinxingren    隐形人机场   remote      HTTP 200      63     2  https://aerilon.invisiblesud.sbs/s/32a50358...
Roxi          Roxi1      remote      HTTP 200      62    36  https://voygo.uk/sub/5ff1e79b-...
Roxi          Roxi       inline-data inline         9     -  data:application/octet-stream;base64,...
```

`build_node_list.py` prints its own verification line. Quote that line as evidence; do not assert success independently.

---

## CLI reference

### `scripts/find_subscriptions.py` — Phase 1

| Flag | Default | Meaning |
|---|---|---|
| `--root PATH` | `%APPDATA%` | Root to scan; expects the double-nested `<app>\<app>\` layout |
| `--scan` | on | Scan every app under `--root` (kept as an explicit switch for readability) |
| `--app NAME` | — | Restrict to one app directory; repeatable |
| `--probe` | off | Actually request each URL (network I/O) and record status / node count / quota |
| `--out DIR` | — | Write `subscriptions.csv` + `subscriptions.json` |
| `--show-token` | off | Keep the full token in CSV (masked to 72 chars by default) |

### `scripts/yaml_to_nodes.py` — Phase 2

| Invocation | Meaning |
|---|---|
| `yaml_to_nodes.py <in.yaml> <out.json>` | Single file |
| `yaml_to_nodes.py --batch <dir> <out-dir>` | Every `*.yaml` in a directory → `<name>.json` |

Output contract: `[{"name": ..., "type": ..., "server": ..., "port": ...}, ...]`. Only these four keys are extracted; nested `ws-opts` / `reality-opts` / `headers` are deliberately ignored so their `server:` / `port:` sub-keys cannot be mistaken for node fields.

### `scripts/build_node_list.py` — Phase 3

| Flag | Default | Meaning |
|---|---|---|
| `--root DIR` | — | Auto-discover `*nodes.json` under a root, one dump = one airport |
| `--manifest FILE` | — | Explicit list: `sources[]`, `extra_yaml[]`, `raw_dirs[]` |
| `--out DIR` | required | Output directory. Keep it **outside** `--root` |
| `--layout {pair,merged}` | `pair` | `pair` = one row per `host+port`; `merged` = one row per host, ports comma-joined |
| `--no-archive` | off | Do not copy source YAML into `yaml/` |

`--strict-info` from v1.0.0 is gone: dropping notice-only ports is now unconditional, because a `(host, port)` that no real node references is never a node.

---

## Phase 0 — identify the stack

Two stacks account for essentially every desktop client seen in practice. Identify first; the directory layout and core binary follow from the answer.

| | Stack A | Stack B |
|---|---|---|
| UI | Flutter (`flutter_windows.dll` + `data/app.so`) | Flutter (same) |
| Core | `<App>Core.exe`, 32–45 MB, Go | `roxi-core.dll`, ~57 MB, Go 1.25.6 |
| Core project | private mihomo fork, build path `UnrivaledSpeed/core/Clash.Meta/` | hiddify-core v2 + sing-box |
| Helper | `<App>HelperService.exe`, ~830 KB, Rust + hyper/warp + tokio | none — core runs in-process, driven by a Cobra CLI |
| Config | `%APPDATA%\<app>\<app>\` | same |
| Store | `database.sqlite :: profiles.url` | `db.sqlite :: profile_entries.url` |

One command confirms Stack A — the core panics without a socket argument and prints its build path:

```bash
./<App>Core.exe -v
# panic: dial unix -v: connect: ...
#   main.startServer  C:/actions-runner/_work/UnrivaledSpeed/UnrivaledSpeed/core/server.go:73
```

The numeric tail of `DisplayVersion` (`3.0.0+YYYYMMDDNN`) is a **build stamp**, not a version: two different airports sharing `3.0.0+2026092210` came off the same assembly line the same day.

---

## Phase 1 — find the subscription URL

Follow this order and stop at the first step that yields data:

| Step | Source | Cost | Hit rate |
|---|---|---|---|
| 1 | `database.sqlite :: profiles.url` | seconds | ~95 % of Flutter/Clash clients |
| 2 | `db.sqlite :: profile_entries.url` | seconds | sing-box/hiddify clients |
| 3 | plaintext `outbounds[]` in `configs/*.json` | seconds | clients that pre-expand the profile |
| 4 | decrypt `profiles/999999.yaml` / `config.yaml` | hours | only when 1–3 are empty |
| 5 | process memory dump | minutes, noisy | only when the client never persisted anything |

Steps 4–5 are the ones that get over-used. Reaching for them first is the single most common waste of a run.

### Rules that decide whether this phase succeeds

- **Two URL shapes, both valid.** `.../api/v1/client/subscribe?token=<32hex>` and `.../s/<32hex>`. A matcher that only looks for `token=` silently misses every airport on the short-link scheme.
- **`data:` URIs are subscriptions too.** Some profiles embed the base64 node list in the `url` column with no remote endpoint. Decode after the comma; it yields a `vless://` list.
- **UA decides the response.** Send `clash-verge/v2.0.0`. Anything else gets a 200 with a decoy page, or a 403. A 200 with zero `server:` occurrences and no `proxies:` block is a decoy, not an empty airport — the script flags it as `疑似诱饵/非订阅`.
- **Certificate errors are expected.** Several airports serve mismatched or self-signed certs; verification is disabled for this request only.
- **An empty `profiles` table is not proof of absence.** If `config.yaml` and `profiles/999999.yaml` are fresh and tens of KB in size, the client *has* a profile — the row is stale, or written only after a sync. Check `mtime`, ask for a sync, re-read.
- **Multiple profiles: take the union, not the newest.** A client can hold an active plan, an expired plan, and an inline `data:` profile. The active subscription can legitimately return fewer endpoints than the client's own expanded `outbounds[]`, because the local file pre-expands CDN variants per node. Merge subscription ∪ local outbounds, dedup on `(host, port)`; picking one source loses 30–40 endpoints.

---

## Phase 2 — parse the returned YAML

Three `proxies:` shapes appear in the wild, often mixed inside one provider's output:

| | Shape | Looks like |
|---|---|---|
| 1 | flow map | `- { name: 'SG07', type: vless, server: a.example, port: 54407 }` |
| 2 | block map | `- name: ...` / `  type: vless` / `  server: ...` / `  port: 443` |
| 3 | inline array | `proxies: [{...}, {...}]` |

A parser that handles only shape 1 returns **zero** records for roughly a third of airports, and does so silently. Always cross-check the parse count against the raw text:

```bash
grep -c 'server\s*:' sub.yaml     # must equal len(nodes.json)
```

A mismatch means a shape was dropped. Fix the parser; do not proceed with the short list.

---

## Phase 3 — normalise, merge, verify

The script encodes the rules this task gets wrong every time:

- **Label derivation** strips `_nodes`, platform suffixes (`_windows`, `-win64`, `-macos`, …) and weak instrumentation stems (`apex`, `core`, `app`, `wy`, `zy`). Without this one airport shows up twice in the airport column.
- **Subset suppression** skips a dump whose `(host, port)` set is a proper subset of another — those are intermediate scrapes, not airports.
- **Placeholder hosts** (`1.1.1.1`, `0.0.0.0`, `127.0.0.1`, `localhost`, `::1`, `::`) are dropped.
- **Notice rows** (`剩余流量` / `套餐到期` / `距离下次重置` / `请更新订阅` / `官网` / `到期`) and their English equivalents are not nodes.
- **Notice-only ports.** A `(host, port)` survives only if at least one *real* node references it. `cdn.example.lol:28931` exists solely on a row named `官网：www.example.lat` → not a node. `1.ltdflix-space.xyz:58061` also appears on a real `美国01` node → it stays.
- **Ad-named real nodes are not notices.** Several airports name genuine nodes as advertising (`香港1官网:xxx`, `香港2续费购买请前往官网 优惠码xxx`). They occupy consecutive ports, carry full UUIDs, and work; a naive "contains 官网 → notice" filter silently deletes 20 working endpoints. The script decides in two stages: an anchored whole-sentence notice (`^(剩余流量|套餐到期|…)`) is a notice without further argument; otherwise an ad word (`官网` / `续费` / `网址` / `购买` / `永久`) **plus** a region or line hint (`香港` / `日本` / `节点` / `专线` …) means real node. That is what separates `官网：www.rck.lat` (notice) from `香港1官网:xxx` (node).
- **Ports** `"443"`, `"080"`, `443`, `443.0` → `443` / `80`; out-of-range or non-numeric → dropped with reason `invalid-host-or-port`.
- **Hosts** are lowercased, IPv6 brackets stripped, scheme and path removed, so `Example.COM` and `example.com` fold into one row.

Every drop is written to `REPORT.md` and `node_list.json :: dropped_rows` with one of: `placeholder-host`, `invalid-host-or-port`, `info-row`, `notice-only-port`.

---

## Output layout

```text
<out-dir>/
├── node_list.csv         # main table, 4 fixed columns, one row per host+port
├── node_list.json        # endpoints + hosts aggregation + per-airport stats + dropped_rows
├── REPORT.md             # per-airport stats + full drop-reason audit log
├── subscriptions.csv     # subscription URLs, store location, probe status, quota, expiry
├── subscriptions.json    # same, machine-readable
└── yaml/
    ├── <Airport>.yaml    # loadable mihomo / sing-box profile per client
    └── raw/              # untouched originals, including undecrypted *.dat
```

### Output contract

| Column | Content | Rule |
|---|---|---|
| `节点IP或域名` | Normalised host | Lowercase, no scheme, no path, no brackets |
| `端口` | Port | `pair` layout: one integer. `merged`: comma-joined, no spaces |
| `提取时间` | `YYYY-MM-DD HH:MM:SS` | Source file `mtime`; identical for all rows of one airport |
| `来源机场` | Label | Derived from the dump filename / manifest, never from a weak stem |

CSV is written with `utf-8-sig` so Excel opens Chinese without garbling.

---

## Validation record

v1.2.0 was validated on **16 shipping desktop clients** in two batches, plus three controls (FlClash 0.8.95, Clash Verge Rev 2.5.10, a subscription-only sample):

| Metric | Value |
|---|---|
| Clients analysed | 16 (15 Stack A, 1 Stack B) |
| Endpoint rows produced | 1,129 across two batches (558 + 571) |
| Unique hosts, batch 1 / batch 2 | 91 / 86 |
| Subscription URLs recovered | 11, all HTTP 200 on re-probe |
| Largest single subscription | 41,249 B, 63 nodes, 2 hosts, ports 54101–54510 |
| Notice / placeholder rows dropped | 15 |

Static analysis only below Phase 1: nothing here resolves DNS, opens TCP to a node, logs into a panel, or modifies an installed client.

---

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| "No subscription found" | Client had not synced; `profiles` empty | Check `mtime` / size, re-read after the client syncs |
| 200 OK but 0 nodes | UA rejected, decoy page | Send `clash-verge/v2.0.0`; treat zero `server:` as decoy |
| A whole airport parses to 0 nodes | Block-style `proxies:` not handled | Use `yaml_to_nodes.py`; verify with `grep -c 'server\s*:'` |
| ~20 real ports vanish | Ad-named nodes hit the notice filter | Apply the `PURE`-anchored rename first |
| Endpoint count drops after a source switch | Took the subscription instead of the union with `outbounds[]` | Merge both, dedup on `(host, port)` |
| Airport column has duplicate labels | Weak stem used as a name | Subset suppression + `WEAK_STEMS`; read the skip log |
| One airport missing entirely | Dump filename does not end in `nodes.json` | Rename, or pin it in `manifest.json` |
| Row count inflated on re-run | `--out` nested inside `--root` | Move `--out` outside `--root` |
| Garbled Chinese in Excel | Missing BOM | Script writes `utf-8-sig` |

---

## Design principles

1. **Static first, dynamic last.** A read-only SQLite query beats a memory dump by two orders of magnitude in cost and a factor of ten in reproducibility.
2. **Each phase has a file contract.** Phases can be re-run, replaced, or skipped; no phase imports another.
3. **Verification is printed, not asserted.** The script measures its own output (header, row count, four non-empty columns, port range) and prints the result. Reports quote that line.
4. **Every drop is explained.** A row that disappears without a reason in `dropped_rows` is a bug, not a filter.
5. **No third-party dependencies.** Stdlib only, so the tool still runs inside an air-gapped analysis VM.

---

## Repository layout

```text
vpn-client-config-inventory-skill/
├── README.md                  # this file
├── README（CH）.md             # Chinese edition
├── RELEASE_v1.2.0.md          # release notes for the current tag
└── (skill payload ships as a release asset)
        vpn-client-config-inventory/
        ├── SKILL.md
        ├── references/api_reference.md
        └── scripts/
            ├── find_subscriptions.py
            ├── yaml_to_nodes.py
            └── build_node_list.py
```

---

## Version history

| Tag | Date | Summary |
|---|---|---|
| [v1.2.0](https://github.com/ShiyuZeng/vpn-client-config-inventory-skill/releases/tag/v1.2.0) | 2026-11-19 | Full pipeline: stack fingerprint → subscription discovery → YAML parse → pair layout → notice-only-port filtering |
| [v1.0.0](https://github.com/ShiyuZeng/vpn-client-config-inventory-skill/releases/tag/v1.0.0) | 2026-11-18 | First release: merge / dedup / verify pre-made node dumps |

---

## Scope

This tool inventories configuration that already exists on the machine it runs on, for that machine's owner. It reads local databases, requests the subscription URLs found there (the token *is* the authentication and it is already local), and archives the result.

It does not attack the providers' backends, does not modify installed clients, and does not test whether any endpoint is reachable. Recovered endpoint data is sensitive by nature — treat the output directory as a credential store.

---

## License

MIT. See [LICENSE](LICENSE).

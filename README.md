# SKILL.md（GitHub GFM 优化版）

> 
> 优化点：标准YAML front‑matter、代码块指定语言、标题层级规范化、表格对齐、目录、折叠块收纳长内容、告警块、目录锚点、文件树语法、修复原始文档换行与空格，完全兼容GitHub预览渲染。

```
---
name: vpn-client-config-inventory
description: >
  Extract & inventory VPN/proxy endpoints from local client configuration dumps.
  Supports node deduplication, port merging, airport provenance tracking,
  CSV export and mihomo/clash YAML archive. Static‑analysis only; no network probe.
agent_created: true
keywords:
  - node inventory
  - clash
  - mihomo
  - node deduplicate
  - subscription parse
  - host‑port table
trigger_phrases:
  - 把节点整理成一张表
  - 节点去重并合并端口
  - 把每个机场的域名和端口列出来
  - consolidate nodes json
  - build host port table
---
```

# VPN / Proxy Node Inventory

- [Overview](#overview)
- [Output Contract](#output-contract)
- [Workflow](#workflow)
  - [1. Locate the dumps](#1-locate-the-dumps)
  - [2. Run the builder script](#2-run-the-builder-script)
  - [3. Built‑in Deduplication Rules](#3-builtin-deduplication-rules)
  - [4. Validation & Verification](#4-validation--verification)
  - [5. Report Output](#5-report-output)
- [Deliverables](#deliverables)
- [Failure Modes & Troubleshooting](#failure-modes--troubleshooting)
- [Scope Boundary](#scope-boundary)

## Overview

Consume one‑or‑more node dump files (JSON records containing `name` / `type` / `server` / `port`), produce a single deduplicated table.

- Each row represents **one normalized host**, with all associated ports aggregated in one cell.
- Preserve extraction timestamp and source airport label.
- Archive original mihomo / clash YAML config alongside exported artifacts.

> 
> Heavy lifting implemented in script: `scripts/build_node_list.py` — do **not** re‑implement aggregation logic manually.

> 
> [!NOTE]
> This skill performs static file analysis **only**. It never launches client, dumps process memory, test connectivity or perform DNS resolution.

## Output Contract

Expected output table format (CSV column order fixed):

| 节点IP或域名 | 端口 | 提取时间 | 来源机场 |
| --- | --- | --- | --- |
| 1.ltdflix‑space.xyz | 58001,58002,58081 | 2026‑11‑18 11:52:53 | KooDog |
| cdn.rockzf.lol | 12933,13230,53679 | 2026‑11‑18 11:52:53 | RocketVPN |

Specification:

1. Ports separated by comma **without spaces**, keep single CSV cell.
2. **One row per normalized host**, do not expand to one row per raw node record.
3. Timestamp uses dump file mtime; same airport shares identical timestamp value.
4. Airport label parsed from dump metadata (not folder name).

## Workflow

### 1. Locate the dumps

Search for files matching patterns: `*nodes.json`, `*_nodes.json`, `*Nodes.json`.

Typical file layout examples:

- Per‑client consolidated dump: `output/<Airport>_nodes.json`
- Instrument intermediate scrape: `apex_nodes.json`, `wy_nodes.json`, `zy_core_nodes.json`
- Platform‑specific output: `wandacloud‑windows/_re/output/nodes.json`

Rules:

- If user provides target folder, scan within that root.
- **Do NOT re‑subscribe / re‑scrape / launch client / dump memory**. Operate purely on existing disk dumps.
- If required dump artifacts missing: list missing files and stop execution.

### 2. Run the builder script

#### Auto‑discovery (Recommended)

```
python scripts/build_node_list.py --root <scan-root> --out <out-dir>
```

> 
> ⚠️ Important: Place `--out` directory **outside** `--root`. Otherwise script output will be re‑scanned in next run. Script will auto‑exclude output folder if mis‑configured.

#### Explicit manifest mode (for precise labeling & extra yaml pinning)

```
python scripts/build_node_list.py --manifest manifest.json --out <out-dir>
```

`manifest.json` example:

```
{
  "sources": [
    {
      "label": "KooDog",
      "nodes": "output/KooDog_nodes.json",
      "yaml": "output/KooDog_config.yaml"
    }
  ],
  "extra_yaml": [
    {"label": "misc", "path": "extracted/FlClashMisc.yaml"}
  ],
  "raw_dirs": ["extracted"]
}
```

Available flags:

| Flag | Description |
| --- | --- |
| `--strict-info` | Drop ports originating purely from subscription notice rows |
| `--no‑archive` | Skip copying & archiving source YAML profiles |

🔍 Expand: Built‑in Deduplication Rules (click open)

### 3. Built‑in Deduplication Rules

Script implements all normalization logic below; avoid hand‑rolled dedup.

**Label derivation**

- File pattern `X_nodes.json` → source label `X`.
- Strip platform suffix: `_windows` / `-win64` / `-macos`.
- Drop weak instrumentation stems: `apex`, `core`, `app`, `wy`, `zy`.
- Fallback: nearest meaningful parent directory name; final fallback `unknown`.

**Subset suppression**

- If dump’s `(host,port)` set is proper subset of another dump → treat as intermediate scrape, skip and log.
- When two dumps have identical endpoint set: preserve source with cleaner human‑readable label.

**Placeholder hosts filtering**

> 
> These are subscription notice anchors, real proxy endpoints never match these:
> `1.1.1.1`, `0.0.0.0`, `127.0.0.1`, `localhost`, `::1`, `::`
> Entire rows containing above hosts get dropped.

**Notice‑row handling**
Node name contains keywords:

> 
> Chinese: `剩余流量` / `套餐到期` / `距离下次重置` / `请更新订阅` / `官网` / `到期`
> English: `remaining` / `expire` / `traffic left` / `reset in`

- Default behavior: merge their `server:port` into host table, but do **not** treat as valid node record.
- With `--strict‑info`: ports from notice rows also get discarded.

**Port parsing & sanitization**

- Normalize numeric port value: `"443"`, `"080"`, `443`, `443.0` → `443` / `80`.
- Drop invalid: port `0`, >65535, non‑numeric values; logged reason: `invalid‑host‑or‑port`.

**Host normalization**

- Lowercase hostname string.
- Strip IPv6 bracket wrapper `[]`.
- Remove URL scheme and trailing path segments.> 
> `Example.COM` and `example.com` merged as single host row.

### 4. Validation & Verification

Script self‑validates generated CSV before writing report:

- Exact header match
- Row count matches merged unique host count
- All four columns non‑empty per row
- Every port integer within `1‑65535`

Console print validation line example:

```
校验: N 行 / M 个端口 / 唯一 host K
```

> 
> Always quote this printed line as validation evidence, do not assert success independently.

> 
> [!WARNING]
> Validation does **not** mean nodes are reachable. No DNS resolution or TCP connectivity checks are performed.

### 5. Report Output

`REPORT.md` auto‑generated, includes:

- Per‑airport statistics: raw / valid / notice record counts
- Top hosts ranked by aggregated port count
- Full table of dropped items with rejection reason

Chat‑side human summary should include:

1. Unique host total count
2. Aggregated total port entries
3. How many rows contain multiple merged ports
4. Any airport dump parse failures

## Deliverables

Output directory tree:

```
<out-dir>/
├── node_list.csv          # main table, 4 columns
├── node_list.json         # structured data: ports array, timestamps, dropped_rows
├── REPORT.md              # stats + drop‑reason audit log
└── yaml/
    ├── <Airport>.yaml     # loadable mihomo/clash config per source airport
    └── raw/               # untouched original artifacts (including encrypted *.dat)
```

Archiving rules:

- Always archive YAML even if user only requested table output.
- Opaque binary artifacts (`*.dat`, encrypted profiles): copy under `yaml/raw/`, mark as non‑decrypted. **Never attempt decryption**.

## Failure Modes & Troubleshooting

| Symptom | Root Cause | Resolution |
| --- | --- | --- |
| Airport column duplicated e.g. `apex` + `龙猫云` | Weak stem interpreted as airport name | Controlled by `WEAK_STEMS` + subset suppression; inspect skip log printed by script |
| Entire airport missing from final table | Dump filename does not end with `nodes.json` | Rename file or add entry inside manifest.json |
| Ports missing after dedup execution | Ports from notice‑rows filtered | Default keeps notice‑row ports; `--strict‑info` explicitly removes them |
| Output row count unexpectedly inflated | Output folder nested under scan‑root, re‑discovering generated csv | Move `--out` outside scan‑root or clean old artifacts before rerun |
| Garbled Chinese CSV opening in Excel | Missing UTF‑8 BOM header | Script writes with `utf‑8‑sig` encoding natively |
| `解析失败` in airport column | Truncated / corrupted JSON source file | Report parse failure; **do not invent fake endpoints for broken dump** |

## Scope Boundary

✅ In‑scope:

- Inventory configuration data already present as local disk files provided by user.

❌ Out‑of‑scope (will never perform):

- Fetch remote subscriptions
- Authentication against service backend
- Decrypt encrypted profile blobs
- Dump running client process memory
- Active network probing / connectivity testing

> 
> [!IMPORTANT]
> If required data only lives inside running‑client runtime memory and no static dump available: explicitly state static dump insufficient and describe what runtime extraction would require. Do **not** silently switch to runtime extraction path.

---

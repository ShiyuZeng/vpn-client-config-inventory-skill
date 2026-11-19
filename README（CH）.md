# vpn-client-config-inventory

> 从本机桌面客户端的配置落盘中提取代理节点与订阅地址，归一化、去重、自校验，并归档为一份可审计的清单。

[![release](https://img.shields.io/badge/release-v1.2.0-blue?style=flat-square)](https://github.com/ShiyuZeng/vpn-client-config-inventory-skill/releases)
[![python](https://img.shields.io/badge/python-3.8%2B-3776ab?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![platform](https://img.shields.io/badge/platform-Windows%2010%20%7C%2011-0078d4?style=flat-square&logo=windows&logoColor=white)]()
[![deps](https://img.shields.io/badge/依赖-仅标准库-success?style=flat-square)]()
[![network](https://img.shields.io/badge/联网-仅拉取订阅-lightgrey?style=flat-square)]()
[![license](https://img.shields.io/badge/license-MIT-green?style=flat-square)](LICENSE)

[English](README.md) | **简体中文**

---

## 目录

- [一句话](#一句话)
- [为什么会有这个工具](#为什么会有这个工具)
- [v1.2.0 改了什么](#v120-改了什么)
- [流水线结构](#流水线结构)
- [安装](#安装)
- [快速开始](#快速开始)
- [命令行参数](#命令行参数)
- [阶段 0 —— 先认栈](#阶段-0--先认栈)
- [阶段 1 —— 找订阅地址](#阶段-1--找订阅地址)
- [阶段 2 —— 解析返回的 YAML](#阶段-2--解析返回的-yaml)
- [阶段 3 —— 归一化、合并、自校验](#阶段-3--归一化合并自校验)
- [输出结构](#输出结构)
- [实测记录](#实测记录)
- [故障对照表](#故障对照表)
- [设计原则](#设计原则)
- [仓库结构](#仓库结构)
- [版本历史](#版本历史)
- [适用范围](#适用范围)
- [许可](#许可)

---

## 一句话

一次运行产出三样东西，按价值排序：

| # | 产物 | 回答什么问题 |
|---|---|---|
| 1 | `subscriptions.csv` | 每个客户端的订阅挂在哪个地址、现在还通不通、还有多少流量、什么时候到期 |
| 2 | `node_list.csv` | 一行一个 `host + port` —— 节点IP或域名 \| 端口 \| 提取时间 \| 来源机场 |
| 3 | `yaml/<机场>.yaml` | 每个客户端一份可直接加载的 mihomo / sing-box 配置，外加未改动的原始提取件 |

数据基本都在本地 SQLite 里，明文，旁边躺着一份加密的 YAML 缓存。读 SQLite、请求订阅、解析返回，就够了。去逆向加密或者 dump 进程内存是**最后**一步，而且通常根本不需要走到那一步 —— 这一条判断决定了一次运行是十分钟还是三小时。

```bash
python scripts/find_subscriptions.py --scan --probe --out subs   # 阶段 1
python scripts/yaml_to_nodes.py --batch subs/yaml nodes/         # 阶段 2
python scripts/build_node_list.py --root nodes --out node_list   # 阶段 3
```

```text
布局: pair  校验: 571 行 / 571 个端口 / 唯一 host 86 / 有效端点 571
```

---

## 为什么会有这个工具

v1.0.0 只管管道中段：它消费别人已经导出的 `*_nodes.json`。实际用起来，"别人"那一步才是难点 —— 每个客户端存配置的地方都不一样，其中一半还是加密的。

v1.2.0 把整条管道接上了：**认客户端 → 认它的 SQLite → 认订阅地址 → 拉取 → 解析 → 归一化 → 自校验 → 归档**。同时把两种主流桌面技术栈的指纹写死在里面，因为栈的答案决定了配置存在哪、内核二进制叫什么。

这个仓库里的每一条规则，都是真人跑真样本时踩出来的。下面每节都写明了它是被哪个坑逼出来的，方便你在踩到之前认出来。

---

## v1.2.0 改了什么

| 维度 | v1.0.0 | v1.2.0 |
|---|---|---|
| 起点 | 消费现成的 `*_nodes.json` | 从已安装的客户端开始 |
| 订阅发现 | 无 | `find_subscriptions.py`，五级决策顺序 |
| 技术栈识别 | 无 | Flutter + mihomo 私有分支 / Flutter + sing-box(hiddify) DLL |
| YAML 解析 | 每次手写 | `yaml_to_nodes.py`，覆盖三种 `proxies:` 形态，不依赖三方库 |
| 表格布局 | 一行一 host，端口合并 | `--layout pair`（默认）/ `merged` |
| 提示行端口 | 混入表格 | 无真实节点引用即剔除 |
| 广告命名的真实节点 | 合并前手工改名 | 脚本内两段式判定 |
| 加密 | 算在范围内 | 明确划出范围，并说明为什么不必碰 |
| 故障表 | 6 条 | 8 条，每条给出真正能定位它的诊断动作 |

完整变更见 [RELEASE_v1.2.0.md](RELEASE_v1.2.0.md)。

---

## 流水线结构

```text
┌─ 阶段 0 ───────────────┐   给安装目录取指纹
│  Stack A / Stack B     │   → 决定配置在哪、内核叫什么
└───────────┬────────────┘
            │
┌─ 阶段 1 ─▼────────────┐   %APPDATA%\..\database.sqlite :: profiles.url
│  订阅地址               │   → 五级决策顺序，命中即停
└───────────┬────────────┘
            │  HTTPS GET，UA = clash-verge/v2.0.0
┌─ 阶段 2 ─▼────────────┐   YAML / data: URI / 已展开的 outbounds[]
│  *_nodes.json          │   → {name, type, server, port}
└───────────┬────────────┘
            │
┌─ 阶段 3 ─▼────────────┐   归一化 → 去重 → 子集抑制 → 自校验
│  node_list.csv/.json   │   → 节点IP或域名 | 端口 | 提取时间 | 来源机场
│  REPORT.md  yaml/      │   → 剔除原因审计 + 归档配置
└────────────────────────┘
```

每个阶段是独立脚本，阶段之间靠文件契约衔接。任意一段都能单独重跑，阶段 3 的输入可以手写，已经拿到 YAML 的时候阶段 1 可以完全跳过。

---

## 安装

### 作为技能

```text
~/.workbuddy/skills/vpn-client-config-inventory/     (WorkBuddy)
~/.claude/skills/vpn-client-config-inventory/        (兼容 Claude Code 的目录约定)
```

把 release 里的压缩包解开，让 `SKILL.md` 落在目录根部：

```text
vpn-client-config-inventory/
├── SKILL.md
├── references/api_reference.md
└── scripts/{find_subscriptions,yaml_to_nodes,build_node_list}.py
```

### 作为独立脚本

不用安装。Python 3.8+，只用标准库 —— 不需要 `pyyaml`、不需要 `requests`、不需要 `pycryptodome`。把 `scripts/` 拷到任何地方就能跑。

```bash
python --version   # 3.8+
```

---

## 快速开始

```bash
# 1. 看看本机装了什么，订阅还通不通
python scripts/find_subscriptions.py --scan --probe --out subs

# 2. 把拉到的 YAML 转成阶段 3 的输入契约
python scripts/yaml_to_nodes.py --batch subs/yaml nodes/

# 3. 合并、去重、自校验、归档
python scripts/build_node_list.py --root nodes --out out/node_list
```

`find_subscriptions.py` 每个 profile 打一行：

```text
app           label      kind        status     nodes hosts  url
yinxingren    隐形人机场   remote      HTTP 200      63     2  https://aerilon.invisiblesud.sbs/s/32a50358...
Roxi          Roxi1      remote      HTTP 200      62    36  https://voygo.uk/sub/5ff1e79b-...
Roxi          Roxi       inline-data inline         9     -  data:application/octet-stream;base64,...
```

`build_node_list.py` 会打印自己的校验行。报告里引用这一行作为证据，不要自己另行断言成功。

---

## 命令行参数

### `scripts/find_subscriptions.py` —— 阶段 1

| 参数 | 默认 | 含义 |
|---|---|---|
| `--root PATH` | `%APPDATA%` | 扫描根目录，按 `<app>\<app>\` 双层同名结构查找 |
| `--scan` | 开 | 扫描 `--root` 下所有应用（保留为显式开关，便于阅读） |
| `--app NAME` | — | 只处理指定目录名，可重复 |
| `--probe` | 关 | 真正发起请求（有网络 IO），记录状态 / 节点数 / 流量 / 到期 |
| `--out DIR` | — | 写出 `subscriptions.csv` 与 `subscriptions.json` |
| `--show-token` | 关 | CSV 里保留完整 token（默认截断到 72 字符） |

### `scripts/yaml_to_nodes.py` —— 阶段 2

| 调用方式 | 含义 |
|---|---|
| `yaml_to_nodes.py <in.yaml> <out.json>` | 单文件 |
| `yaml_to_nodes.py --batch <dir> <out-dir>` | 目录内每个 `*.yaml` → 同名 `.json` |

输出契约：`[{"name": ..., "type": ..., "server": ..., "port": ...}, ...]`。只提这四个键，嵌套的 `ws-opts` / `reality-opts` / `headers` 一律不参与，避免把它们的 `server:` / `port:` 子键误当成节点字段。

### `scripts/build_node_list.py` —— 阶段 3

| 参数 | 默认 | 含义 |
|---|---|---|
| `--root DIR` | — | 自动发现在根目录下的 `*nodes.json`，一个 dump 对应一个机场 |
| `--manifest FILE` | — | 显式清单：`sources[]`、`extra_yaml[]`、`raw_dirs[]` |
| `--out DIR` | 必填 | 输出目录，**必须放在 `--root` 之外** |
| `--layout {pair,merged}` | `pair` | `pair` = 一行一个 `host+port`；`merged` = 一行一个 host，端口逗号合并 |
| `--no-archive` | 关 | 不把源 YAML 复制进 `yaml/` |

v1.0.0 的 `--strict-info` 已移除：剔除"只被提示行引用"的端口现在是默认行为，因为没有任何真实节点引用的 `(host, port)` 本来就不是节点。

---

## 阶段 0 —— 先认栈

实际遇到的桌面客户端基本只有这两种栈。先认栈，配置在哪、内核叫什么，答案就跟着来了。

| | Stack A | Stack B |
|---|---|---|
| UI | Flutter（`flutter_windows.dll` + `data/app.so`） | Flutter（相同） |
| 内核 | `<App>Core.exe`，32–45 MB，Go | `roxi-core.dll`，约 57 MB，Go 1.25.6 |
| 内核来源 | mihomo 私有分支，构建路径 `UnrivaledSpeed/core/Clash.Meta/` | hiddify-core v2 + sing-box |
| 服务 | `<App>HelperService.exe`，约 830 KB，Rust + hyper/warp + tokio | 无 —— 内核在进程内加载，靠 Cobra CLI 驱动 |
| 配置 | `%APPDATA%\<app>\<app>\` | 相同 |
| 存储 | `database.sqlite :: profiles.url` | `db.sqlite :: profile_entries.url` |

一条命令确认 Stack A —— 内核少了 socket 参数会 panic，并把构建路径打出来：

```bash
./<App>Core.exe -v
# panic: dial unix -v: connect: ...
#   main.startServer  C:/actions-runner/_work/UnrivaledSpeed/UnrivaledSpeed/core/server.go:73
```

`DisplayVersion` 里 `3.0.0+YYYYMMDDNN` 的尾段是**构建时间戳**，不是版本号：两个不同机场都是 `3.0.0+2026092210`，说明它们同一天下的同一条产线。

---

## 阶段 1 —— 找订阅地址

按下面的顺序走，命中即停：

| 步 | 数据源 | 成本 | 命中率 |
|---|---|---|---|
| 1 | `database.sqlite :: profiles.url` | 秒级 | 约 95 % 的 Flutter/Clash 客户端 |
| 2 | `db.sqlite :: profile_entries.url` | 秒级 | sing-box / hiddify 系 |
| 3 | `configs/*.json` 里的明文 `outbounds[]` | 秒级 | 自己把订阅展开过的客户端 |
| 4 | 解 `profiles/999999.yaml` / `config.yaml` | 小时级 | 仅当 1–3 全空 |
| 5 | 进程内存 dump | 分钟级，且噪声大 | 仅当客户端从未落过盘 |

真正被滥用的是第 4、5 步。一上手就奔着它们去，是一次运行里最常见的浪费。

### 决定这一阶段成败的规则

- **两种 URL 形态都有效。** `.../api/v1/client/subscribe?token=<32位hex>` 与 `.../s/<32位hex>`。只匹配 `token=` 会静默漏掉所有走短链的机场。
- **`data:` URI 也是订阅。** 有些 profile 把 base64 节点表直接写在 `url` 列里，没有远程端点。解码逗号之后的部分，得到一串 `vless://`。
- **UA 决定返回什么。** 用 `clash-verge/v2.0.0`。换别的 UA，要么拿到 200 的诱饵页，要么 403。返回 200 但全文没有 `server:`、也没有 `proxies:` 段的，是诱饵不是空机场 —— 脚本会标成 `疑似诱饵/非订阅`。
- **证书报错是常态。** 不少机场用错配或自签证书，只对这一次请求关校验。
- **`profiles` 表为空不等于没有订阅。** 如果 `config.yaml` 和 `profiles/999999.yaml` 时间是新的、大小是几十 KB，说明客户端**有**配置，只是那一行是旧的，或者它同步完才写行。看 `mtime`，让客户端同步一次，再读一遍。
- **多个 profile 取并集，不取最新。** 一个客户端可能同时存着有效套餐、过期套餐、以及一条带 UUID 的 `data:` 内联配置。有效套餐拉回来的端点可能比客户端本地已展开的 `outbounds[]` 还少，因为本地文件对每个节点预先展开了 CDN 变体。订阅 ∪ 本地 outbounds 按 `(host, port)` 去重；只取一边会丢 30–40 个端点。

---

## 阶段 2 —— 解析返回的 YAML

`proxies:` 在真实样本里有三种写法，同一家的输出里都可能混着来：

| | 形态 | 长相 |
|---|---|---|
| 1 | flow map | `- { name: '新加坡07', type: vless, server: a.example, port: 54407 }` |
| 2 | block map | `- name: ...` / `  type: vless` / `  server: ...` / `  port: 443` |
| 3 | 内联数组 | `proxies: [{...}, {...}]` |

只认第 1 种的解析器，对大约三分之一的机场返回**零条**记录，而且不报错。解析完务必和原文对账：

```bash
grep -c 'server\s*:' sub.yaml     # 必须等于 nodes.json 的条数
```

对不上说明漏了一种形态。修解析器，不要拿着这份短清单往下走。

---

## 阶段 3 —— 归一化、合并、自校验

脚本把这件事每次都会做错的规则写死了：

- **标签推导**剥掉 `_nodes`、平台后缀（`_windows`、`-win64`、`-macos` …）和弱埋点词（`apex`、`core`、`app`、`wy`、`zy`）。不剥的话，同一个机场会在"来源机场"列里出现两次。
- **子集抑制**：某个 dump 的 `(host, port)` 集合是另一个的真子集时跳过 —— 那是中间态抓取，不是机场。
- **占位 host**（`1.1.1.1`、`0.0.0.0`、`127.0.0.1`、`localhost`、`::1`、`::`）剔除。
- **提示行**（`剩余流量` / `套餐到期` / `距离下次重置` / `请更新订阅` / `官网` / `到期`）及其英文对应词，不算节点。
- **仅提示行端口**：一对 `(host, port)` 只有在被至少一条**真实**节点引用时才保留。`cdn.example.lol:28931` 只出现在名为 `官网：www.example.lat` 的行上 → 不是节点；`1.ltdflix-space.xyz:58061` 同时出现在真实的 `美国01` 节点上 → 保留。
- **广告命名的真实节点不是提示行。** 有若干机场把真节点命名成广告（`香港1官网:xxx`、`香港2续费购买请前往官网 优惠码xxx`）。它们占连续端口、带完整 UUID、能用；一个"含官网即提示"的粗暴过滤会静默删掉 20 个可用端点。脚本分两段判定：以 `^` 锚定的整句提示（`^(剩余流量|套餐到期|…)`）直接判为提示；否则名称里同时出现广告词（`官网` / `续费` / `网址` / `购买` / `永久`）**和**地区 / 线路词（`香港` / `日本` / `节点` / `专线` …）的，判为真实节点。`官网：www.rck.lat`（提示）与 `香港1官网:xxx`（节点）就是靠这条分开的。
- **端口**：`"443"`、`"080"`、`443`、`443.0` → `443` / `80`；越界或非数字 → 以 `invalid-host-or-port` 剔除。
- **host**：小写、去 IPv6 方括号、去 scheme 与路径，于是 `Example.COM` 与 `example.com` 折叠成同一行。

每一条剔除都会写进 `REPORT.md` 和 `node_list.json :: dropped_rows`，原因取值：`placeholder-host`、`invalid-host-or-port`、`info-row`、`notice-only-port`。

---

## 输出结构

```text
<out-dir>/
├── node_list.csv         # 主表，固定 4 列，一行一个 host+port
├── node_list.json        # 端点 + host 聚合 + 各机场统计 + dropped_rows
├── REPORT.md             # 各机场统计 + 完整剔除原因审计
├── subscriptions.csv     # 订阅地址、存储位置、实测状态、流量、到期
├── subscriptions.json    # 同上，机器可读
└── yaml/
    ├── <机场>.yaml        # 每个客户端一份可直接加载的 mihomo / sing-box 配置
    └── raw/              # 未改动的原始件，含未解密的 *.dat
```

### 输出契约

| 列 | 内容 | 规则 |
|---|---|---|
| `节点IP或域名` | 归一化 host | 小写、无 scheme、无路径、无方括号 |
| `端口` | 端口 | `pair` 布局为单个整数；`merged` 为逗号连接、不带空格 |
| `提取时间` | `YYYY-MM-DD HH:MM:SS` | 取源文件 `mtime`；同一机场所有行相同 |
| `来源机场` | 标签 | 来自 dump 文件名或 manifest，绝不取弱埋点词 |

CSV 以 `utf-8-sig` 写出，Excel 打开中文不乱码。

---

## 实测记录

v1.2.0 在两批共 **16 款在役桌面客户端**上验证，另加三个对照（FlClash 0.8.95、Clash Verge Rev 2.5.10、纯订阅样本）：

| 指标 | 数值 |
|---|---|
| 分析客户端 | 16 款（15 款 Stack A，1 款 Stack B） |
| 产出端点行 | 两批合计 1,129 行（558 + 571） |
| 唯一 host（第一批 / 第二批） | 91 / 86 |
| 取回订阅地址 | 11 条，复测全部 HTTP 200 |
| 最大单份订阅 | 41,249 B，63 节点，2 个 host，端口 54101–54510 |
| 剔除的提示 / 占位行 | 15 条 |

阶段 1 之后全部为静态分析：不解析 DNS、不对节点开 TCP、不登录面板、不修改已安装的客户端。

---

## 故障对照表

| 现象 | 原因 | 处理 |
|---|---|---|
| 报"未发现订阅" | 客户端还没同步过，`profiles` 为空 | 看 `mtime` / 大小，等同步后重读 |
| 200 但 0 个节点 | UA 被拒，返回诱饵页 | 用 `clash-verge/v2.0.0`；把 0 个 `server:` 视为诱饵 |
| 整个机场解析出 0 条 | block 形态的 `proxies:` 没处理 | 用 `yaml_to_nodes.py`，再用 `grep -c 'server\s*:'` 对账 |
| 约 20 个真实端口消失 | 广告命名的节点被提示过滤误杀 | 先做 `PURE` 锚定的改名 |
| 换数据源后端点变少 | 只取了订阅，没和 `outbounds[]` 取并集 | 两边都取，按 `(host, port)` 去重 |
| 来源机场列重复 | 弱埋点词被当成机场名 | 子集抑制 + `WEAK_STEMS`，看跳过日志 |
| 某个机场整体缺失 | dump 文件名不是以 `nodes.json` 结尾 | 改名，或在 `manifest.json` 里钉住 |
| 重跑后行数虚高 | `--out` 放在 `--root` 里面 | 把 `--out` 挪到 `--root` 之外 |
| Excel 中文乱码 | 缺 BOM | 脚本本身用 `utf-8-sig` 写出 |

---

## 设计原则

1. **静态优先，动态兜底。** 一次只读 SQLite 查询，成本比内存 dump 低两个数量级，可复现性高一个数量级。
2. **每个阶段一个文件契约。** 阶段可重跑、可替换、可跳过；阶段之间不互相 import。
3. **校验是打印出来的，不是断言出来的。** 脚本自己检查表头、行数、四列非空、端口范围，然后打印结果。报告引用这一行。
4. **每一条剔除都要有原因。** 一行数据无声消失、`dropped_rows` 里却找不到，那是 bug，不是过滤。
5. **不引入三方依赖。** 只用标准库，这样在隔离的分析虚拟机里也能跑。

---

## 仓库结构

```text
vpn-client-config-inventory-skill/
├── README.md                  # 英文版
├── README（CH）.md             # 本文件
├── RELEASE_v1.2.0.md          # 当前 tag 的发布说明
└── (技能本体以 release 附件形式分发)
        vpn-client-config-inventory/
        ├── SKILL.md
        ├── references/api_reference.md
        └── scripts/
            ├── find_subscriptions.py
            ├── yaml_to_nodes.py
            └── build_node_list.py
```

---

## 版本历史

| Tag | 日期 | 摘要 |
|---|---|---|
| [v1.2.0](https://github.com/ShiyuZeng/vpn-client-config-inventory-skill/releases/tag/v1.2.0) | 2026-11-19 | 打完整条管道：认栈 → 找订阅 → 解析 YAML → pair 布局 → 仅提示行端口剔除 |
| [v1.0.0](https://github.com/ShiyuZeng/vpn-client-config-inventory-skill/releases/tag/v1.0.0) | 2026-11-18 | 首个版本：对现成节点 dump 做合并、去重、校验 |

---

## 适用范围

本工具清点的是它运行的那台机器上**已经存在**的配置，服务于该机器的所有者。它读本地数据库、请求其中记录的订阅地址（token 本身就是鉴权，且已在本地）、把结果归档。

它不攻击机场后端、不修改已安装的客户端、不测试任何端点是否可达。取回的端点数据本身具备凭据性质，输出目录请按凭据目录对待。

---

## 许可

MIT，见 [LICENSE](LICENSE)。

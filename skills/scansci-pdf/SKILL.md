---
name: scansci-pdf
description: 下载学术论文。支持 DOI、arXiv ID、关键词搜索、批量下载、Elsevier API、WebVPN/CARSI 机构访问、下载失败排障。当用户要求下载论文(单篇或批量)、搜索文献、获取引文、配置 Elsevier/ScienceDirect API、或下载遇到 Cloudflare/验证页/代理问题时使用。用户仅讨论 PDF 解析/转换工具的对比或选型(如 pymupdf4llm vs MinerU)而无需检索文献证据时不要使用;只有确实要检索或下载文献时才加载本 skill。
---

# ScanSci PDF — 学术论文极速下载与检索

20+ 数据源并行竞速，首个成功立即返回。数百篇以上的清单需要先分类摸底（OA/Sci-Hub/需机构）时，转用 `scansci-sort` skill。

---

## 核心设计哲学与铁律（Fast-Fail 标准规范）

在批量检索与真实文献下载中，**速度优先与确定性优先**。绝不为一个不可达渠道盲等数分钟阻塞全局流水线：

1. **硬核物理截断（Hard Physical Cutoffs）**:
   - **极短宽限期（`grace = 10s`）**：彻底废弃历史 180s/300s 人工等待机制。未在 10s 内拿到全文流即刻熔断切换。
   - **反爬与人工登录零盲等**：无头模式（`browser_headless = true`）下，遇到任何 SSO/CAS 人工登录跳转、二维码验证、或 ALTCHA/Cloudflare 僵尸验证墙，**0 秒硬拦截直接 fast-fail 跳过**，严禁进入 100 次循环（300s）人工等待。
   - **单篇判定上限**：全渠道综合判定耗时严格约束在 15~25 秒内闭环。
2. **齐发竞速模式（`race_mode = full`）**:
   - 默认开启 Flat Parallel Racing。轻量源与无头浏览器通道在同一毫秒并发齐发，单篇耗时由逐级累加降至“单源最大耗时”（通常 ≤ 15 秒）。
3. **黄金三梯队渠道架构（Golden 3-Tier Hierarchy）**:
   - **Tier 1 (极速 HTTP 并行流，1-5s，命中率最高且零开销)**:
     - 预印本原生源 (bioRxiv, arXiv, Research Square, SSRN)
     - 开放仓储与 OA 接口 (Unpaywall, EuropePMC, PMC, OpenAIRE, CORE)
     - 开放期刊直连 (Frontiers, eLife, eNeuro, MDPI CDN `lane_mdpi_cdn`)
   - **Tier 2 (校园网原生直连 Chrome 捕获，8-15s，高校主力)**:
     - 依托校园网原生 IP（如清华/中科院内网），TUN 模式必须配置对应出版商域名为 `DIRECT` 直连。
     - 针对 Elsevier/Cell Press，自动重写 DOI 为 PII (`sciencedirect.com/science/article/pii/<PII>`)，免去无权限 API Key 的无效 403 往返，直接由无头浏览器在校园内网完成 PDF 数据流抽取与 S3 签名重定向抓取。
     - 严格约束 `max_browser_workers = 1` 共享无头浏览器，避免 Windows Commit/分页内存耗尽错误（`0x800705AF`）。
   - **Tier 3 (Sci-Hub 闪电直链镜像，10-15s，经典文献底座)**:
     - 优先直出无验证码镜像（`sci-hub.bz`, `sci-hub.ru` 免验直链）。
     - 遇 ALTCHA 验证盾直接判定为灰源无权并熔断切换，不与防爬虫验证码死磕。
   - **黑名单与禁忌渠道（Strictly Banned Lanes）**:
     - 无机构绑定的普通 Elsevier API Key（100% 403 `NOT_ENTITLED`）。
     - 自动化批量模式下弹窗等待人工输入的 WebVPN / CARSI CAS 页面。
     - 境外 Tor SOCKS 代理直连。

---

## 环境检查

工具列表含 `scansci_pdf_*` → 用 MCP 工具;否则 CLI 兜底,先 `scansci-pdf check` 确认依赖。

## 策略选择规则(必读)

**用户指定了来源 = 只用那个来源:**

| 用户说的是 | 策略 | 说明 |
|-----------|------|------|
| "从 Sci-Hub 下载" | `scihub_only` | 只走 Sci-Hub |
| "优先 scihub" | `scihub_first` | OA 仍竞速——OA 更快时最终来源可能是 OA |
| "只要免费合法的" | `legal_only` | 排除 Sci-Hub / LibGen |
| 没指定 | `fastest`(默认) | 全源并行竞速 |

```bash
scansci-pdf config-cmd download_strategy scihub_only
```

## 单篇下载

```bash
scansci-pdf get <DOI>                    # 零配置竞速 (默认 race_mode=full, grace=10s)
scansci-pdf fetch <DOI> [--output DIR]   # 机构级联通道
```

竞速分层: Tier1 出版商/OA 直链(4s) → Tier2 OpenAlex/Unpaywall(5s) → Tier3 EuropePMC/PMC/arXiv(8s) → Tier4 Sci-Hub 闪电镜像(10s) → Tier5 校园网/机构浏览器直连(15s)。

**换源重下必须清缓存**:`rm -f <out>/.doi_index.json && rm -rf ~/.scansci-pdf/cache/*`。

## 批量下载标准流程

```bash
scansci-pdf batch dois.txt --output <dir> --lanes   # 标准车道调度极速模式
scansci-pdf batch dois.txt --no-lanes               # 逐篇平铺竞速模式
scansci-pdf batch dois.txt --scihub                 # 灰源纯享竞速引擎
```

**批量标准三步法**:
1. **预排查与本地断点 (Pre-triage)**:
   - 扫描目标目录现有已完成有效 PDF (大小 `> 10KB` 且魔数开头为 `%PDF-`)。
   - 自动生成去重队列，秒级跳过已成功论文。
2. **极速车道并发 (Lanes Batching)**:
   - S2 批量预嗅探 (500 DOI/请求直接获取 OA PDF 直链)
   - 并行 HTTP 快车道 (OA 直链 + MDPI CDN 构造)
   - 灰色源竞速 (Sci-Hub 免验镜像)
   - 校园网/机构直连快速探测 (未登录即刻跳过)
3. **审计与结算交付 (Reconciliation Manifest)**:
   - 生成 `final_download_manifest.csv` 与 `final_summary.json`。
   - 明确标注获取渠道 (`Sci-Hub`, `Nature`, `ScienceDirect`, `OA/Preprint`) 或不可达客观原因（如 2024+ 闭源顶刊未收录且无 OA）。

⚠️ **>300 篇必须分批**——校验阶段并发 validate 会 TimeoutError 崩溃。重跑同文件自动跳过已完成。

## 检索与引文

```bash
scansci-pdf search "关键词" --limit 10 --sort cited_by_count   # 13源引擎,失败降级三源
```

搜作者优先用 `--author "Dabo Guan"` 或 OpenAlex `--author-id`,别把人名放 query(全文匹配会混入同名/被引提及)。引文格式(citation: bibtex/ris/endnote)仅 MCP 支持。发现层:CLI `plan → estimate → find --out <dir>`,再 `build-queue <dir> --out queue.txt` 接 batch。OpenAlex 配额按 IP 每日计(429 就降级 Unpaywall 单点并发);Unpaywall 批量端点常 500,自写单点并发脚本更稳。

## 场景 → 工具链(常见意图直接对号入座)

| 用户说 | 动作 |
|---|---|
| "下载这篇 <DOI/arXiv/文章页URL>" | `get <标识符>`;URL 直接喂,自动抽 DOI/arXiv;要补充材料加 `--si`,要 AI 可读全文加 `--md`(PDF 仍是默认交付物) |
| "某人的全部/近年论文" | `search --author "Name" --out queue.txt` → `batch queue.txt --lanes` |
| "某主题/关键词 + 年份/被引过滤" | `search "kw" --year-from 2023 --sort cited_by_count [--out queue.txt]` |
| "给一份清单(xlsx/csv/txt/bib/APA)" | `batch 文件 --lanes`(表格/队列自动识别;渠道按 DOI 前缀自动预测) |
| "上次有失败的,补齐" | `batch --retry <output>/batch_results.json`(自动读失败清单重跑) |
| "模糊引用('Wang 2023 CRISPR 那篇')" | 先 `search "Wang 2023 CRISPR"` 拿候选让用户确认,别硬猜 DOI |
| 系统性文献发现(PRSIMA/引文追链/高召回多源检索) | 装了 `scansci-find` skill/CLI → 用它;没装 → 本地降级链 `search → verify → resolve-oa → build-queue → batch --lanes` |
| "只要合法来源" | `legal_only` 策略或 `--scihub` 反选 |

## 机构渠道(按 DOI 前缀路由,先快后慢)

1. **`10.1016`(Elsevier / Cell Press)**:
   - **校园网直通(推荐)**: 校园网环境(`is_campus_network: true`)直接走浏览器校园网直通模式,自动将 DOI 转换为 PII 直链 (`sciencedirect.com/science/article/pii/<PII>`),8 秒内秒级下载完整全文 PDF。
   - **无需 insttoken**: 即便 Elsevier API 报 403 NOT_ENTITLED,引擎也会自动回退至浏览器校园网直连通道;Session Cookie 自动持久化复用,无感绕过 Cloudflare。
2. **`10.1007`(Springer)→ Springer TDM API**:
   - 机构订阅+TDM 授权 key 交付 JATS XML 全文。
3. **其余出版商 → WebVPN/CARSI**:
   - 仅在用户明确交互模式下执行 CAS 登录；在自动化无头批处理中，未授权则秒级放行，绝不挂起任务。

## 排障速查

| 症状 | 原因与修复方案 |
|---|---|
| 卡在 180s 或 300s 不动 | 触发了历史的人工登录等待逻辑。现已在 `grace = 10` 与无头快速失败中杜绝。遇到时检查配置是否设置 `browser_headless: true` |
| Sci-Hub 遇到 ALTCHA / 机器人验证 | 验证盾直接触发 fast-fail 熔断，自动尝试下一个免验镜像（如 `sci-hub.bz`）或交由校园网/OA 渠道 |
| Windows 报 `0x800705AF` 页面文件太小 | 浏览器进程并发过多耗尽提交内存限制。确保 `max_browser_workers = 1` 并在复用单例 Chrome |
| Elsevier API 报 403 NOT_ENTITLED | 正常现象（无机构专属 Token）。已配置校园网直通模式，自动由 Chrome 访问 ScienceDirect 原生内网抓取 |
| 所有出版商均提示 Cloudflare 阻断 | 检查 Clash/TUN 代理分流：校园网环境必须将知网、ScienceDirect、Nature 等出版商域名配置为 `DIRECT` 直连 |
| 下载失败结果带 `source_failures` | 逐渠道失败明细已完成快速收集。对于 2024+ 闭源顶刊且无 OA 者，属客观不可达，直接输出未命中清单供用户线下补充 |

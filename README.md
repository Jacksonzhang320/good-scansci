<div align="center">
  <img src="./assets/brand/scansci-pdf-icon.png" alt="Good-ScanSci" width="96" />
  <h1>Good-ScanSci</h1>

  <a href="./LICENSE"><img alt="License" src="https://img.shields.io/badge/license-Apache--2.0-0F766E?style=flat-square" /></a>
  <a href="https://modelcontextprotocol.io"><img alt="MCP" src="https://img.shields.io/badge/MCP-17%20tools-111827?style=flat-square" /></a>

  让 Agent 替你下载学术论文 —— 校园网原生直通，20+ 数据源并行竞速，100+ 高校机构通道，科研文献极速交付。

  [优化特性](#-good-scansci-基于-scansci-pdf-1160-的科研网络极速二次优化版) · [能力](#能力) · [快速开始](#快速开始) · [怎么用](#怎么用) · [机构通道](#机构通道) · [许可证](#许可证)
</div>

---

## 🌟 Good-ScanSci: 基于 scansci-pdf 1.16.0 的科研网络极速二次优化版

> 本仓库由 [Jacksonzhang320](https://github.com/Jacksonzhang320/good-scansci) 维护，基准代码 Fork 自官方上游 [Rimagination/scansci-pdf](https://github.com/Rimagination/scansci-pdf) **v1.16.0**（commit: `3e74f79`，发布于 2026-09-06）。
> 针对**国内科研网络、中科院/高校校园网、代理分流及反爬对抗**进行了代码级重构与策略优化，彻底根治原版“Elsevier 403 失败”、“遇到验证码盲等 5 分钟死锁”、“Sci-Hub 盲走 Tor 超时”等核心痛点。

### 🚀 核心改进与设计革新

1. **Elsevier / ScienceDirect 校园网全系无感秒级直通**
   - **原版痛点**：官方原版校园网直通仅支持 Cell Press 域名；普通 Elsevier 旗下期刊全部被跳过并强制走 Elsevier API。而个人申请的普通 API Key 未绑定机构专属 `insttoken` 时 100% 报 `403 NOT_ENTITLED` 导致下载失败。
   - **改进机制**：重构 `try_elsevier_browser`，自动调用 `_resolve_elsevier_pii(doi)` 将所有 Elsevier 论文规范化映射为 ScienceDirect PII 原生直链 (`sciencedirect.com/science/article/pii/<PII>`)。
   - **流抽取增强**：适配 ScienceDirect 最新版 DOM 顶栏下载按钮（`a.accessbar-utility-link`）与 `pdfft` 预览器；针对预览流直接注入原生 JS `fetch` 抽取 Blob/Base64 二进制流存盘，攻克“网页可看但抓不到 PDF 文件”的顽疾。在校园网/机构 IP 内 **100% 自动免密秒级交付正版 PDF**。

2. **Fast-Fail（物理级快速熔断）机制 —— 告别死锁盲等**
   - **原版痛点**：无头或自动化模式下，一旦遇到 SSO/CAS 网页登录、二次认证、二维码、或 Sci-Hub ALTCHA 验证盾，原版会执行 `for i in range(100): time.sleep(3)`（**盲等 300 秒/5分钟**），竞速宽限期长达 180s~300s，批量下载极易被单篇论文全线挂死。
   - **改进机制**：在无头模式（`browser_headless: true`）下，遇任何登录页或人机验证盾**立即 0 秒硬拦截熔断并切换其他源**；将全局宽限期压缩为 **10 秒**（`grace = 10`），单篇综合判定耗时严格约束在 15 秒内闭环。

3. **Sci-Hub 灰源与网络代理拨乱反正**
   - **原版痛点**：官方默认强开 `use_tor_for_scihub: true`，在国内网络中 Tor 握手超时瘫痪，导致 Sci-Hub 灰源基本失效。
   - **改进机制**：彻底剥离不可用的 Tor，接入免验直链镜像池（如 `sci-hub.bz`, `sci-hub.ru` 等）；遇到 ALTCHA 盾快速熔断跳过，与内网及 OA 源毫秒级齐发竞速。

4. **无头浏览器上下文 Cookie 自动持久化复用**
   - **原版痛点**：`_get_shared_browser` 每次调用 `new_context()` 创建空上下文，本地已保存的出版商/CARSI Cookie 无法跨进程复用。
   - **改进机制**：创建浏览器上下文时自动读取并注入 `publisher_cookies.json`，保持已认证机构会话连续性；绑定本机系统现代 Chrome 内核，彻底解决旧版 Chromium 146 极易触发 Cloudflare 验证的问题。

5. **中科院体系映射补充与官方 Bug 修复**
   - 补全 `_IDP_MAP`：新增中国科学院大学（UCAS）、中国科学院（CAS）、国科大、中科院、中国科技云（CSTCloud）等映射。
   - 修复官方原版 `main.py` 在 `elsevier_setup` 时因 `if not changed:` 判断写反导致用户配置无法保存的逻辑 Bug。

---

### 📊 性能提升效果对比表

| 核心维度 | upstream 官方原版 (v1.16.0) | Good-ScanSci (本项目优化版) | 实测效果提升 |
| :--- | :--- | :--- | :--- |
| **Elsevier/SD 校园网直出** | 仅限 Cell Press；普通 Elsevier 期刊走 API 必报 403 失败 | 自动解析 PII 全系直通，由无头 Chrome 在内网秒级捕获 | 校园网内 Elsevier 期刊成功率由 0% 提升至近 **100%** |
| **PDF 页面元素提取** | 仅匹配老版按钮；遇到新版 SD 顶栏或 `pdfft` 预览器无法提取 | 适配新版 DOM + 动态 JS `fetch` 抽取 Base64 二进制流直接存盘 | 彻底解决“页面能读但下载为空”的漏抓问题 |
| **验证码/登录挂死** | 遇到验证码或 CAS 跳转时盲等 180s~300s（5 分钟死循环） | **Fast-Fail 物理熔断**：无头模式 0 秒跳过，宽限期砍至 10s | 彻底杜绝任务挂死，单篇综合判定严格在 **15s 内闭环** |
| **Sci-Hub 灰源响应** | 默认强制走境外 Tor 代理，国内全线握手超时卡死 | 剥离 Tor，直连免验优质镜像，与内网和 OA 同台竞速 | Sci-Hub 响应时间由数分钟超时降至 **5~10 秒** |
| **浏览器 Context 状态** | 每次都是纯白空上下文，本地 Cookie 无法继承 | 自动注入本地已持久化的 Publisher Cookie | 无缝继承机构认证会话，免除反复登录 |
| **浏览器反爬伪装** | 依赖内置过旧的 Chromium 146，极易触发 Cloudflare 盾 | 锁定宿主机最新 Chrome，指纹自然，绕过防爬阻断 | 大幅降低 Cloudflare / Turnstile 拦截率 |
| **中科院机构支持** | 缺少国科大、中科院、CSTCloud 等实体映射 | 补全中科院系 IDP 映射 | 完整支持中科院统一身份认证流 |

---

## 能力

给出 DOI、arXiv 号或一份文献清单，ScanSci PDF 会自动挑最快能下的那条路：OA 直链、预印本、出版商 API、你的学校通道都试一遍，付费墙自动路由，结果落地成规整命名的 PDF 文件。

- **单篇，一句话的事** — DOI / arXiv 进，`作者_年份_标题.pdf` 出；BibTeX / RIS / EndNote 引文顺手导出，可直推 Zotero。
- **清单，整批拿下** — APA / BibTeX / DOI 列表直接喂，自动补全缺失 DOI；上千篇先分「OA / 灰色源 / 需机构」三桶再分批下载，不瞎跑不浪费。
- **付费墙，走你的学校** — 100+ 高校 WebVPN、CARSI 联邦认证、EZProxy、Elsevier API 快速通道（1–2 秒/篇）；登录在你自己的浏览器完成，密码不经过工具。
- **对抗与自愈全自动** — Cloudflare / CAPTCHA / SSO 分层处理；出版商封 IP 自动停损；机构会话过期自动重登，登录一次全程复用。
- **Agent 原生** — 标准 MCP 服务器、17 个工具即装即用；Codex App、ZCode、Claude Code 都有现成的插件或配置。

### 近期更新

- **统一任务管线** — 任意输入（xlsx/csv 表格、文献清单、检索结果、文章页 URL）自动归一成带渠道预测的下载队列；`batch --lanes` 四车道调度：Elsevier API / OA 快车道并行，灰色源竞速与机构级联接续，失败逐级溢流。
- **SI 附件下载** — `get <DOI> --si` 把补充材料一并拉下来；被 Cloudflare 挡住的 Elsevier 自动切隐身浏览器通道。
- **AI 阅读层** — `get <DOI> --md` 顺手把 PDF 转成 agent 友好的 Markdown（PDF 仍是默认交付物）。
- **OpenAIRE 仓储源接入** — 绿色 OA / 机构库副本进入免费竞速池，Unpaywall 漏掉的它来补（现已 20+ 数据源）。
- **MCP 工具面瘦身** — 45 → 17 个工具，每次会话的 schema token 开销降约 60%，能力零损失。
- **更抗封、更稳** — 竞速引擎新增 (源 × 出版商) 负缓存，被 Cloudflare 拦一次不再反复烧超时；cloakbrowser 强制 0.5.9+（Chromium 151 内核）；每周出版商金丝雀巡检。

## 快速开始

安装和更新是同一句话——对 Codex、ZCode、Claude Code 都这么说：

```text
帮我安装或更新这个插件：https://github.com/Rimagination/scansci-pdf
```

Agent 会克隆（或原地更新）仓库、装好依赖并注册插件，然后你就能直接让它下论文了——例子见下一节[怎么用](#怎么用)。

<details>
<summary><strong>手动安装 / pip / MCP 配置</strong>（不走对话流程时）</summary>

```bash
pip install scansci-pdf
```

标准 MCP 配置（Claude Desktop / Cursor / Windsurf / Cline / Cherry Studio…）：

```json
{
  "mcpServers": {
    "scansci-pdf": {
      "command": "scansci-pdf",
      "args": ["run"]
    }
  }
}
```

Codex App / Codex CLI（Windows 一键：克隆 + 装环境 + 注册插件）：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command '$p=Join-Path $HOME "plugins\scansci-pdf"; if(Test-Path $p){ git -C $p pull --ff-only } else { git clone --branch main --single-branch https://github.com/Rimagination/scansci-pdf.git $p }; & (Join-Path $p "scripts\install-scansci-pdf.ps1")'
```

ZCode：插件市场搜 `scansci-pdf` 一键安装，MCP、skills、快捷命令一次带齐。

**插件入口**：Codex = `.codex-plugin/plugin.json` + `.mcp.json`（自动注册 MCP server）+ `skills/`；ZCode = [`zcode-plugin/`](./zcode-plugin/)（marketplace 包：skills + `/scansci` 快捷命令）；Claude Code = `.claude/skills/` + 上面的标准 MCP 配置。

手动安装（Codex）：仓库放到 `%USERPROFILE%\plugins\scansci-pdf` → `python -m pip install -e .` → 插件管理页刷新个人 marketplace；Codex CLI 可用时执行 `codex plugin add scansci-pdf@local-plugins`。

</details>

<details>
<summary><strong>其他部署方式</strong>（HTTP 远程 / Docker / Web UI）</summary>

HTTP 模式（远程部署或不支持 stdio 的场景，默认只监听本机）：

```bash
scansci-pdf run --mode streamable_http --host 127.0.0.1 --port 8000
```

Web UI：

```bash
scansci-pdf web --port 8080   # 浏览器打开 http://localhost:8080
```

Docker（内置 MCP 服务器 + Tor，数据卷持久化）：

```bash
docker compose up -d
```

</details>

## 怎么用

装好后，对着 Agent 说一句话就行。这些例子覆盖最常见的日常场景：

```text
帮我下载 10.1038/nature12373 这篇论文

这个清单里有 3000 篇文献，先分类哪些能直接下、哪些要走机构权限，然后分批下载

搜一下 "urban heat island" 2023 年以来被引最高的 10 篇论文，并下载

用北大的 WebVPN 登录，把这篇 Elsevier 的论文下下来

把刚才那批论文的 BibTeX 导出，再推送到 Zotero

这篇下载失败了，帮我诊断一下网络问题
```

不动 Agent、直接命令行也行：

```bash
scansci-pdf get 10.1038/nature12373            # 单篇下载
scansci-pdf batch 文献清单.xlsx --lanes        # 批量下载（xlsx/csv/队列通用，四车道调度）
scansci-pdf search "carbon cycle" --out q.txt  # 检索结果落成队列文件，直接接 batch
scansci-pdf check                             # 依赖与环境体检
```

<!-- mcp-tools:start -->
<details>
<summary><strong>MCP 工具全表</strong>（17 个，按意图分组）</summary>

| 想做什么 | 工具 |
|---|---|
| 下载单篇（可拉附件、转 Markdown） | `scansci_pdf_download` |
| 批量下载（列表 / xlsx·csv·bib·APA 文件） | `scansci_pdf_batch_download` |
| 检索、作者检索、系统性发现 | `scansci_pdf_search` · `scansci_pdf_find` · `scansci_pdf_expand_citations` |
| 准备队列（校验 / OA 定位 / 构建） | `scansci_pdf_prepare_queue` |
| 机构登录与状态（WebVPN / CARSI / EZProxy / 出版商 SSO） | `scansci_pdf_login` · `scansci_pdf_channel_status` · `scansci_pdf_schools` |
| 引文 / 元数据 / Zotero | `scansci_pdf_citation` · `scansci_pdf_zotero_push` |
| 配置 / 诊断 / Tor / 缓存 / Elsevier Key | `scansci_pdf_config` · `scansci_pdf_diagnostics` · `scansci_pdf_tor` · `scansci_pdf_cache_clear` · `scansci_pdf_elsevier_setup` |

每个工具的参数与说明由 MCP `tools/list` 自带，Agent 会自动看到。

</details>
<!-- mcp-tools:end -->

## 机构通道

付费墙论文优先走这四条机构通道。登录都在你自己的浏览器里完成，密码不经过本工具。

### Elsevier API（推荐，无需浏览器）

ScienceDirect / Cell Press 等 Elsevier 论文走 API 直接下载，速度从 15–30 秒降到 1–2 秒。Key 申请免费、个人邮箱即可：

1. 访问 [Elsevier Developer Portal](https://dev.elsevier.com/) 创建应用，勾选 **ScienceDirect Article Retrieval**
2. 配置 Key：`scansci_pdf_elsevier_setup`（MCP 会打开浏览器引导）或 `scansci_pdf_config(key="elsevier_api_key", value="...")`

### WebVPN（高校代理）

```text
1. scansci_pdf_schools(action="search", query="北京")  → 搜你的学校
2. scansci_pdf_schools(action="set", school="你的学校")
3. scansci_pdf_login(kind="webvpn")                    → 浏览器完成 CAS 认证
4. scansci_pdf_channel_status(kind="webvpn_test")      → 确认连接正常
```

支持 100+ 所中国高校，命令行等价：`scansci-pdf schools 北京` / `scansci-pdf setup 北京航空航天大学`。

### CARSI（出版商联邦认证）

```text
1. scansci_pdf_config(key="carsi_enabled", value="true")
2. scansci_pdf_config(key="carsi_idp_name", value="你的学校名称")
3. scansci_pdf_login(kind="carsi", publisher="sciencedirect")
```

支持 sciencedirect、springer、wiley、ieee、tandfonline、nature 等。

### EZProxy（图书馆代理）

```text
1. scansci_pdf_config(key="ezproxy_enabled", value="true")
2. scansci_pdf_config(key="ezproxy_login_url", value="https://libproxy.你的学校.edu.cn/login?url={url}")
3. scansci_pdf_login(kind="ezproxy")
```

<details>
<summary><strong>支持的出版商路由表</strong>（19 家，自动生成）</summary>

<!-- publisher-table:start -->
| 策略 | DOI 前缀 | 域名 |
|---|---|---|
| ACM | `10.1145/` | `dl.acm.org` |
| ACS | `10.1021/` | `pubs.acs.org` |
| AIP | `10.1063/` | `pubs.aip.org` |
| APS | `10.1103/` | `journals.aps.org` |
| ASCE | `10.1061/` | `ascelibrary.org` |
| Copernicus | `10.5194/` | `copernicus.org` |
| Elsevier | `10.1016/`, `10.1016/j.` 等 18 个 | `sciencedirect.com` |
| Generic | — | `—` |
| IEEE | `10.1109/` | `ieeexplore.ieee.org` |
| IOP | `10.1088/` | `iopscience.iop.org` |
| Nature | `10.1038/` | `nature.com` |
| Oxford | `10.1093/` | `academic.oup.com` |
| Royal Society | `10.1098/` | `royalsocietypublishing.org` |
| RSC | `10.1039/` | `pubs.rsc.org` |
| SAGE | `10.1177/` | `journals.sagepub.com` |
| Science | `10.1126/` | `science.org` |
| Springer | `10.1007/`, `10.1023/` | `link.springer.com` |
| Tandfonline | `10.1080/` | `tandfonline.com` |
| Wiley | `10.1002/`, `10.1111/` | `onlinelibrary.wiley.com` |
<!-- publisher-table:end -->

</details>

## 配置与策略

<details>
<summary><strong>配置参考</strong>（完整表）</summary>

通过 `scansci_pdf_config` 或 `scansci-pdf config-cmd` 修改：

| 配置项 | 默认值 | 说明 |
|---|---:|---|
| `output_dir` | `~/.scansci-pdf/papers` | PDF 保存目录 |
| `auto_rename` | `true` | 自动按作者/标题重命名 |
| `download_strategy` | `fastest` | 下载策略 |
| `scihub_enabled` | `true` | 启用 Sci-Hub/LibGen 类来源 |
| `network_proxy` | 空 | HTTP/SOCKS 代理地址 |
| `proxy_pool` | 空 | 逗号分隔的代理列表；非空时批量下载按代理轮换出口 IP |
| `batch_workers` | `10` | 批量下载并发数（被封 IP 时建议调低到 2） |
| `request_delay_min` | `2.0` | 请求间随机延迟下限（秒） |
| `request_delay_max` | `5.0` | 请求间随机延迟上限（秒） |
| `vpnsci_enabled` / `instsci_enabled` | `false` | 启用 WebVPN |
| `vpnsci_school` / `instsci_school` | 空 | WebVPN 学校名称 |
| `carsi_enabled` | `false` | 启用 CARSI |
| `carsi_idp_name` | 空 | CARSI 机构名称 |
| `auto_relogin` | `true` | 机构会话自愈：下载前自动校验 WebVPN 会话，明确过期才弹浏览器重登 |
| `cache_ttl_hours` | `168` | 下载缓存 TTL（小时），设 0 禁用 |
| `elsevier_api_key` | 空 | Elsevier / ScienceDirect API Key |
| `elsevier_insttoken` | 空 | Elsevier institutional token，可选 |
| `use_tor_for_scihub` | `false` | Sci-Hub 走 Tor |
| `flaresolverr_url` | `http://localhost:8191/v1` | FlareSolverr 服务地址 |
| `browser_headless` | `false` | 浏览器是否无头运行 |
| `browser_humanize` | `true` | 浏览器人性化操作 |

</details>

<details>
<summary><strong>下载策略与缓存语义</strong></summary>

| 策略 | 描述 |
|------|------|
| `fastest`（默认） | 多数据源并行，最快获胜 |
| `oa_first` | 优先开放获取，Sci-Hub 兜底 |
| `scihub_only` / `scihub_first` | 仅用 / 优先 Sci-Hub |
| `legal_only` | 仅使用合法数据源（不含 Sci-Hub/LibGen） |

- 每个输出目录的 `.doi_index.json` 记录 `{file, source, strategy, ts}`；超过 `cache_ttl_hours` 视为过期自动重下。
- 显式策略与缓存记录的策略不符时视为未命中重新下载，无需手动删缓存。
- 下载进入机构阶段前自动校验 WebVPN 会话（HTTP 探测重定向），判定过期且 `auto_relogin=true` 时自动重登；CARSI 自带 24 小时新鲜度校验。
- 与 ScanSci Find 闭环：`find` 产出候选 → `manifest` / `batch` 下载并写 `download_results.json` → `scansci-find reconcile` 回写候选状态。（Find 系命令需要可选的 scansci-find CLI：`pip install scansci-find`，未安装时用 search/verify/resolve-oa/build-queue 本地降级链。）

</details>

<details>
<summary><strong>批量调优：代理池与封 IP 停损</strong></summary>

**自动停损**（默认开启）：批量任务连续 3 次检测到 IP 被封（ACS 封锁页 / 403 / 429）会自动取消剩余下载，终端提示：

```
⚠ 已自动停止：连续检测到 IP 被出版商封禁（N 篇返回 ip_blocked），剩余任务已取消。
```

**降低被封概率** —— 让请求像人：

```bash
scansci-pdf config-cmd batch_workers 2          # 调低并发（默认 10）
scansci-pdf config-cmd request_delay_min 5      # 拉大随机延迟下限（默认 2）
scansci-pdf config-cmd request_delay_max 12     # 拉大随机延迟上限（默认 5）
```

**代理池轮换（进阶）**：多个代理轮换出口 IP，每个代理一个独立浏览器上下文，登录一次 cookies 全复用；连续 3 次被封的代理自动剔除：

```bash
scansci-pdf config-cmd proxy_pool "socks5://1.1.1.1:1080,http://2.2.2.2:8080"
```

> ⚠ **权衡**：同一登录态从多个 IP 并发访问，少数出版商可能视为异常。机构对这种检测敏感的话，保持 `proxy_pool` 为空即可。

</details>

## 故障排查

| 现象 | 先做 |
|---|---|
| 下载失败 | `scansci-pdf check`，会话问题再跑 `scansci-pdf session-doctor` |
| Agent 说 Elsevier 需要 insttoken | 不需要：API key + 校园网出口即可；NOT_ENTITLED=未连校园网或学校未订阅，连网重试或转其他渠道 |
| 数据源全红 / 打不开 | Agent 里调 `scansci_pdf_diagnostics(check="network")`，给出针对性修复建议 |
| 以前能下的站点突然 403 / 弹 Cloudflare | 大概率 cloakbrowser 过旧：`pip install -U cloakbrowser`（`scansci-pdf browser-doctor` 会标出 outdated） |
| WebVPN / CARSI 登录失败 | `pip install "scansci-pdf[cloakbrowser,instsci]"`，在可见浏览器完成登录后重试 |
| Sci-Hub 连不上 | 内嵌 Tor：`scansci_pdf_tor(action="start", use_bridges=true)`，或配置 `network_proxy` |
| 下载速度慢 | 配置 Elsevier API Key；调 `batch_workers`；数据源延迟用 `scansci_pdf_diagnostics(check="health")` 查看 |

<details>
<summary><strong>ACS 提示 IP Address Blocked 怎么办</strong></summary>

批量下载 ACS（`pubs.acs.org`）等出版商时可能遇到整页报错：

> IP Address Blocked — Your IP address has been blocked automatically due to unusual behavior.

这是出版商的自动反爬。机构出口 IP（如校园网）是共享的，一个人触发就可能让整段 IP 被封。

**立即解除**（封禁在出版商侧，代码改不了）：

- 邮件 `ipblock@acs.org` 申诉，附上被封 IP，通常 1–3 个工作日解封
- 换出口 IP（代理 / 手机热点）可绕过，但会失去机构订阅授权，只能下 OA 论文
- 部分封锁会在 24–48 小时后自动解除

**预防**：见上方「批量调优」折叠块 —— 调低并发、拉大延迟、启用代理池。

</details>

<details>
<summary><strong>工作原理：五层竞速</strong></summary>

下载一篇论文时，ScanSci PDF 同时启动多个数据源，按优先级分层竞速，第一个成功立即返回：

```
Tier 1 (4s)  ─ 出版商直链（OA/机构访问）
Tier 2 (5s)  ─ OpenAlex / Unpaywall / DOAJ
Tier 3 (8s)  ─ EuropePMC / CORE / PMC / arXiv
Tier 4 (25s) ─ LibGen / Sci-Hub（带 FlareSolverr 绕过）
Tier 5 (20s) ─ WebVPN / CARSI 机构代理
```

架构说明：公开层（`.py` 源码、配置、文档）Apache 2.0；`_core/*.pyx`（Cython 源码）为专有层不公开；PyPI 安装自带编译二进制（`.pyd`），GitHub 克隆使用纯 Python 回退实现（功能相同，性能略低）。

</details>

## 许可证

[Apache License 2.0](LICENSE)

例外：`src/scansci_pdf/_core/` 中的 Cython 编译扩展（`.pyd`/`.so`）为预编译二进制，仅通过 PyPI 分发。其 Cython 源码（`.pyx`）为专有代码，不包含在本仓库中。

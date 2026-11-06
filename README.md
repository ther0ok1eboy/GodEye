# AssetWatch

输入企业名称，自动发现并汇总其**域名 / 子域名 / 端口 / Web 资产 / 小程序与公众号**，在终端渲染出资产清单 dashboard，并与上一次运行结果对比，标出**新增 / 消失的资产**。

## 特性

- **企业 → 域名（双路并行）**：`enscan`（爱企查 aqc + 天眼查 tyc）与**企查查无头浏览器**同时采集，互不阻塞；合并去重后进入后续扫描。
- **多公司批量**：`scan --file company_name.txt` 从文本读取多家公司（支持 `#` 注释、去重）。
- **enscan 批量模式**：使用 `-type tyc,aqc -delay 2 -json -f company_name.txt -no-merge` 一次跑完所有公司（`-no-merge` 修复 ENScan_GO `-f -json` 导出到错误嵌套路径的 bug），并按 `enterprise_info[].extra` 回填公司归属。
- **本地扫描链（可并行）**：子域名（`subfinder` + `anubis` 并行）→ `naabu`（端口）→ `httpx`（站点发现）→ 指纹 / 爬取 / 目录扫描并行（`xingfinger` / `katana` / `ffuf` / `dirsearch`）→ `nuclei`（漏洞，可选）。所有工具路径与参数都在配置 `scanners.*.path` / `scanners.*.args` 中。
- **dashboard 底部整合**：端口扫描、站点发现、指纹识别结果整合展示在布局底部三个块；URL 收集（`katana`）与目录扫描（`ffuf`/`dirsearch`）结果不显示在终端，写入 `<data_dir>/scans/urls_collected.txt` 与 `<data_dir>/scans/dir_scan.txt`。
- **外部 API**：鹰图（Hunter）与 FOFA 查询完整 URL 列表。
- **资产清单**：按域 / 子域 / URL / 端口 / 应用 / 漏洞 / **商标 / 专利 / 著作权**分组展示（后三者来自企查查知识产权页）。
- **历史对比**：每次扫描保存 JSON 快照，自动与上次对比，标注 `NEW` / `REMOVED`。
- **按类型落库**：扫描结果还会按资产类型写入 `<data_dir>/assets/`，每个文件每行一条，便于复制粘贴：`domains.txt`、`subdomains.txt`、`ports.txt`、`urls.txt`、`apps.txt`、`vulns.txt`、`trademarks.txt`、`patents.txt`、`copyrights.txt`；带字段的结构化数据另有同名 `.jsonl`。
- **终端 dashboard**：基于 `rich` 的最终页面布局 —— 第一排四个统计方块（资产总数 / 今日变化 / 采集成功 / 采集异常），第二排资产总览长条（🌐域名 / 🔗子域名 / 📢公众号 / 📱APP / 🧩小程序），下面左侧「去重后的子域名」方块（显示全部）+ 右侧「域名 / 小程序 / 公众号 / APP」四个小方块，底部整合「端口扫描 / 站点发现 / 指纹识别」三个块，并占满终端宽度。加 `--detailed` 可展示端口、URL、漏洞、商标、专利、著作权等完整资产。

## 安装

```bash
pip install -r requirements.txt
# 或可执行入口
python assetwatch.py --help
```

可选安装为命令：

```bash
pip install -e .
assetwatch --help
```

## 配置

```bash
python -m assetwatch init            # 生成 ~/.config/assetwatch/config.yaml
# 或拷贝示例
cp config.example.yaml ~/.config/assetwatch/config.yaml
```

编辑 `config.yaml`：

- `scanners.<工具>.enabled`：是否运行该工具。
- `scanners.<工具>.path`：工具二进制路径（留空则用 PATH 自动探测）。
- `scanners.<工具>.args`：传给工具的命令行参数（可覆盖默认）。
- `scanners.<工具>.wordlist`：ffuf/dirsearch 字典路径。
- `apis.hunter.cookie` / `apis.fofa.email+key`：填入 API 凭证并设 `enabled: true`。Hunter 走第三方站 `safe-uat.enhancer-sec.site` 搜索接口，cookie 为浏览器登录后复制（含 `SESSION` 与 `XSRF-TOKEN`），另可配 `base_url`/`engine`/`page_size`。
- `enscan.path`：enscan 二进制路径（默认自动探测 `~/T00ls/...` 或 PATH）；`enscan.fields` 控制采集 `icp,wechat,app,weibo,wx_app` 等。
- `company_domain.qcc_cookie`：企查查 API 采集凭证（从已登录页面复制 Cookie），用于获取网站备案、APP、小程序和公众号。
- `company_domain.use_qcc_browser`：开启企查查 API 采集（保留历史配置名，默认 true）。
- `company_domain.qcc_request_delay` / `qcc_company_delay`：企查查接口间/公司间间隔，默认 `5s` / `10s`；不建议降低，避免触发访问控制。

## 使用

```bash
# 扫描一家/多家企业（也可直接传域名）
python -m assetwatch scan "上海电信" "腾讯" --label telecom

# 从文本批量读取公司（每行一个，支持 # 注释）
python -m assetwatch scan --file company_name.txt

# 不保存快照 / 不对比
python -m assetwatch scan example.com --no-save --no-compare

# 查看历史与展示
python -m assetwatch history
python -m assetwatch show --snapshot telecom_20261102_144814.json

# 查看完整资产（含端口/URL/漏洞/商标/专利等）
python -m assetwatch show --detailed
python -m assetwatch scan "小米" --detailed

# 查看终端 dashboard 详情视图（切换第一排方块对应的详情）
python -m assetwatch show --detail changes   # 今日变化
python -m assetwatch show --detail success   # 采集成功
python -m assetwatch show --detail errors    # 采集异常

# 对比两个快照
python -m assetwatch diff --from a.json --to b.json
```

## 依赖的本地工具

| 工具       | 用途            | 默认开启 | 配置 |
|------------|-----------------|----------|------|
| enscan     | 企业域名/APP/公众号/微博 | 是 | `enscan.path` |
| subfinder  | 子域名枚举       | 是       | `scanners.subfinder.*` |
| anubis     | 子域名枚举（备选）| 是       | `scanners.anubis.*` |
| naabu      | 端口扫描         | 是       | `scanners.naabu.*` |
| httpx      | 存活站点发现     | 是       | `scanners.httpx.*` |
| xingfinger | 指纹识别         | 是       | `scanners.xingfinger.*` |
| katana     | URL 收集/爬取    | 是       | `scanners.katana.*` |
| ffuf       | 目录/文件爆破    | 是       | `scanners.ffuf.*` |
| dirsearch  | 目录扫描         | 否       | `scanners.dirsearch.*` |
| nuclei     | 漏洞检测         | 否       | `scanners.nuclei.*` |

未安装的工具会静默跳过，并在 dashboard 汇总中体现。

## 数据存储

快照保存在 `<data_dir>/snapshots/*.json`，默认为项目内 `data/`，可在配置 `storage.data_dir` 修改。
按类型落库的明细保存在 `<data_dir>/assets/*.txt` / `*.jsonl`。

## 目录结构

```
assetwatch/
  cli.py            # Typer CLI
  orchestrator.py   # 扫描编排
  models.py         # 数据模型
  config/config.py  # 配置加载
  scanners/local.py # 本地工具封装
  sources/company.py# 企业名→域名
  sources/apis.py   # 鹰图/FOFA
  storage/store.py  # 快照+diff
  dashboard/render.py# rich dashboard
```

## 说明

- enscan 采用批量模式 `-type tyc,aqc -delay 2 -json -f <list> -no-merge`（`-no-merge` 修复其 `-f -json` 导出 bug）。爱企查（aqc）渠道开箱即用地解析域名、APP、公众号、微博；天眼查等渠道需在 `config.yaml` 配置对应 cookie（`enscan.cookies.*`）。
- **鹰图（Hunter）**：改为第三方站 `safe-uat.enhancer-sec.site` 的 `POST /api/content/search/getList`，需在 `apis.hunter.cookie` 填浏览器登录后的 `SESSION`/`XSRF-TOKEN`，`enabled: true` 即生效；不再依赖奇安信付费 API key。
- **FOFA**：需配 `email+key` 并 `enabled: true`。
- 企查查采集复用 `qcc/qcc_assets.js` 的签名 JSON API 方式，不再启动无头浏览器；需要已登录 `qcc_cookie`，并会严格串行限速。
- 工具面向授权测试 / 资产梳理场景，请遵守目标站点合规要求。
- 本工具面向授权测试 / 资产梳理场景，请遵守目标站点合规要求。

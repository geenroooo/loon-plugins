# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This repo is for developing **Loon plugins**. Loon is an iOS/iPadOS/tvOS/macOS network proxy tool (like Surge / Quantumult X). A "plugin" is a single `.plugin` text file that bundles rules, URL rewrites, MITM hostnames, and JavaScript scripts so a user can enable a feature (ad-blocking, app unlock, sign-in automation, etc.) with one toggle.

There is no build/lint/test toolchain — a plugin is a plain text file plus the `.js` scripts it references. "Testing" means installing the plugin in the Loon app and watching its behaviour. Validate syntax by eye and by checking regex/JSON.

## Plugin file structure

A `.plugin` file is INI-like: metadata header lines (start with `#!`), then `[Section]` blocks. Order of sections doesn't matter; a section can be omitted if unused.

### Metadata header (`#!key = value`)

```
#!name        = 插件显示名
#!desc        = 一句话说明这个插件做什么
#!author      = 作者
#!homepage    = https://...
#!icon        = https://...        ; 图标 URL
#!system      = iOS,iPadOS,tvOS,macOS   ; 支持的系统(可选)
#!loon_version= 3.2.1(733)          ; 最低 Loon 版本(用到新特性时声明)
#!tag         = 分类标签
#!type        = normal              ; normal=普通功能; parser=出现在订阅节点/规则/配置解析页
```

### Sections

`[Argument]` — 用户可在 App 里调的参数(build 733+)。脚本/重写用 `{参数名}` 引用,脚本内用 `$argument.参数名` 读取。
```
参数名 = input,"默认值",tag=显示名,desc=说明        ; 文本输入
参数名 = select,"选项A","选项B",tag=显示名          ; 下拉选择,第一个是默认
参数名 = switch,tag=显示名,desc=说明                 ; 开关,默认 false
```

`[Rule]` — 分流规则。策略只能是 `DIRECT` / `REJECT`(及 `REJECT-IMG`/`REJECT-DICT`/`REJECT-DROP` 等) / `PROXY`(用户选的策略组)。
```
DOMAIN,google.com,PROXY
DOMAIN-SUFFIX,example.com,REJECT
```

`[Rewrite]`(URL Rewrite)— 正则匹配 URL 做改写/重定向。
```
^https?:\/\/(www\.)?g\.cn https://www.google.com 302    ; 302 重定向
^https?:\/\/example\.com/ad reject                       ; 直接拒绝
```

`[Script]` — 核心。脚本类型: `http-request` / `http-response` / `cron` / `generic` / `dns`。
```
http-request  ^https?:\/\/example\.com  script-path=local.js,tag=请求脚本,enable=true
http-response ^https?:\/\/example\.com  script-path=https://.../x.js,requires-body=true,timeout=10,tag=响应脚本,enable=true
cron "0 8 * * *" script-path=daily.js,tag=每日任务,enable=true
```
常用键: `script-path`(本地文件名或 https URL)、`requires-body=true`(脚本要读 body 时必加)、`timeout`(秒)、`tag`、`enable`、`argument=[{参数名}]`(把 Argument 传给脚本)。用 `enable={某switch参数}` 让开关控制脚本启停。

`[MITM]` — 要解密的域名(脚本/重写碰 HTTPS body 时必须列出,否则抓不到)。
```
hostname = *.example.com,*.sample.com
```

`[Host]` — 自定义 DNS 映射。 `[General]` — `bypass-tun`/`skip-proxy`/`dns-server` 等覆盖项。

## Writing the `.js` scripts

脚本跑在 Loon 的 JS 引擎里(兼容 Surge/QuantumultX 风格 API),不是 Node。可用全局对象:

- `$request` / `$response` — 当前请求/响应(`.url`/`.headers`/`.body`)。
- `$httpClient.get/post(opts, cb)` — 发 HTTP 请求,`cb(error, response, data)`。
- `$persistentStore.read(key)` / `.write(value, key)` — 持久化读写(跨脚本共享数据)。
- `$notification.post(title, subtitle, body)` — 发系统通知。
- `$argument.参数名` — 读 `[Argument]` 传进来的值(switch 为布尔,其余为字符串)。
- `$done(obj)` — **每条路径都必须调用一次**结束脚本。http-request/response 里 `$done({})` 表示放行不改;返回 `{body, headers}` 表示改写;`$done({})` 之外不返回会卡住请求。

每个脚本最后必须命中 `$done(...)`,否则该请求会一直挂起 —— 这是最常见的 bug。

## Conventions

- http-response 脚本要读/改 body 时,`[Script]` 行加 `requires-body=true`,且对应域名必须出现在 `[MITM] hostname` 里。
- 正则要转义 `.` 为 `\.`,匹配 http 和 https 用 `https?`。
- 改 JSON 响应用 `JSON.parse($response.body)` → 改 → `$done({body: JSON.stringify(obj)})`。

## 本仓库现状

公开仓库 `geenroooo/loon-plugins`,推到 GitHub 后 Loon 通过 raw 链接远程引用。改动推送后用户在 Loon 点「更新插件」即可拉到最新(raw 有几分钟 CDN 缓存)。

**目录约定:每个插件单独一个文件夹**(插件文件 + 它引用的脚本都放一起),避免混乱。新插件照此新建文件夹。

**节点查询工具箱** (`node-query-toolbox/`) — 把三个脚本打包成一个插件。Loon 远程链接:
```
https://raw.githubusercontent.com/geenroooo/loon-plugins/main/node-query-toolbox/node-query-toolbox.plugin
```

| 脚本 | 作用 | 出处 |
|------|------|------|
| `net-lsp-x.js` | 入口落地查询(节点入口 IP / 落地 IP) | xream network-info,跨 Surge/QX/Loon/Stash 通用,1500+ 行 |
| `streaming-ui-check.js` | 流媒体解锁查询(NF/Disney/YTB/ChatGPT 等) | 改自 KOP-XIAO |
| `geo_location.js` | 地理位置查询 | 改自 KOP-XIAO,最简,可当模板 |

### 关键:这三个是「节点操作脚本」,在 Loon 插件里用 `generic` 类型声明
Loon 没有专门的 node-action 关键字 —— 这类「选中节点→运行→`$done({title, htmlMessage})` 弹窗」的脚本,在插件里就是写成 `generic`。用户选中某节点运行时,Loon 把该节点通过 `$environment.params.node`(及 `.nodeInfo.address`)传进来;脚本里靠 `isInteraction()`(即 `$environment.params.node` 是否存在)判断当前是不是节点操作模式。请求时带 `node: $environment.params.node` 就走那条线路。
- 直接在插件列表点(没选节点)= 非交互模式,`net-lsp-x.js` 只显示当前网络信息、没有「入口」段,这是正常的。
- `img-url` 可用 SF Symbol(如 `globe.asia.australia.system`,`.system` 后缀表示系统符号),不依赖外部图床。

### net-lsp-x.js 的入口解析改动(已上线)
原脚本解析节点域名拿入口 IP 时,阿里 DNS 解析器(`net-lsp-x.js` 约 1324 行)写死了 `edns_client_subnet: '223.6.6.6/24'`,等于伪装成「阿里云机房」去解析,GeoDNS/CDN 节点会因此返回与用户手机实际连接不同的入口 IP。已删除该行,改回按真实查询来源解析 —— 用户在国内时这样最贴近手机实际连接。脚本仍支持 `DNS` 参数手动切换解析器(`ali`/`cf`/`google`/`tencent`)。本地有 `net-lsp-x.js.bak` 备份(已 gitignore)。

## References

- 官方插件文档: https://nsloon.app/docs/Plugin/
- 插件格式示例: https://github.com/chiupam/tutorial/blob/master/Loon/Plus/Plugin_Format.md

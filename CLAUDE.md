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

## References

- 官方插件文档: https://nsloon.app/docs/Plugin/
- 插件格式示例: https://github.com/chiupam/tutorial/blob/master/Loon/Plus/Plugin_Format.md

# sb-rules

sing-box 远程规则集，供多台 VPS 共用的 WARP 分流域名列表。

| 文件 | 出口 | 用途 |
|---|---|---|
| `warp-domains.json` | WARP IPv4 | Cloudflare Turnstile 站（OpenAI / Claude / linux.do …） |
| `warp6-domains.json` | WARP IPv6（`resolve prefer_ipv6`） | Google 全家 / reCAPTCHA / YouTube / Netflix |
| `direct-domains.json` | 直连 | hCaptcha 等对 WARP 不友好的站 |

`warp6` 的解析策略用 **`prefer_ipv6`** 而不是 `ipv6_only`：双栈站（Google / YouTube / Netflix）一定选 AAAA，
而 Google 系里只有 A 记录的域名（`beacons4.gvt2.com`、`android.clients.google.com`、`www.googleadservices.com` 等）
在 `ipv6_only` 下会 `lookup … empty result` 直接解析失败，`prefer_ipv6` 会回退到 A 并从 WARP v4 出去。

## 维护

直接编辑对应文件的 `domain_suffix` 数组并 push。各 VPS 上的 sing-box 每小时自动拉取；
想立即生效在服务器执行 `systemctl restart sing-box`。

加域名前先想清楚放哪一组：
- 站点弹 **Cloudflare Turnstile**（转圈方块）→ `warp-domains`，并确认 `challenges.cloudflare.com` 仍在列表里；
- 站点弹 **Google reCAPTCHA** / "unusual traffic" → `warp6-domains`；
- 站点走 WARP 反而变差（hCaptcha、部分银行/支付站）→ `direct-domains`。

## 服务器端接入（sb.json 的 route.rule_set）

```json
{ "tag": "warp-domains",   "type": "remote", "format": "source", "url": "https://raw.githubusercontent.com/ppvia/sb-rules/main/warp-domains.json",   "download_detour": "direct", "update_interval": "1h" },
{ "tag": "warp6-domains",  "type": "remote", "format": "source", "url": "https://raw.githubusercontent.com/ppvia/sb-rules/main/warp6-domains.json",  "download_detour": "direct", "update_interval": "1h" },
{ "tag": "direct-domains", "type": "remote", "format": "source", "url": "https://raw.githubusercontent.com/ppvia/sb-rules/main/direct-domains.json", "download_detour": "direct", "update_interval": "1h" }
```

对应的 route.rules 顺序（`resolve` 是非终结动作，必须排在 `warp6-domains → warp-out` 之前）：

```json
{ "action": "sniff" },
{ "rule_set": ["direct-domains"], "outbound": "direct" },
{ "rule_set": ["warp6-domains"], "action": "resolve", "strategy": "prefer_ipv6" },
{ "rule_set": ["warp6-domains"], "outbound": "warp-out" },
{ "rule_set": ["warp-domains"],  "outbound": "warp-out" }
```

并开启 `experimental.cache_file.enabled = true`，这样规则集会缓存到本地，GitHub 暂时不可达时仍可启动。
注意 `cache.db` 被线上实例独占，任何用同一份 sb.json 起的临时测试实例都要把 `cache_file.path` 改到 /tmp。

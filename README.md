# sb-rules

sing-box 远程规则集，供多台 VPS 共用的 WARP 分流域名列表。

| 文件 | 出口 | 用途 |
|---|---|---|
| `warp-domains.json` | WARP IPv4 | Cloudflare Turnstile 站（OpenAI / Claude / linux.do …） |
| `warp6-domains.json` | WARP IPv6（`resolve ipv6_only`） | Google 全家 / reCAPTCHA / YouTube / Netflix |
| `direct-domains.json` | 直连 | hCaptcha 等对 WARP 不友好的站 |

## 维护

直接编辑对应文件的 `domain_suffix` 数组并 push。各 VPS 上的 sing-box 每小时自动拉取；
想立即生效在服务器执行 `systemctl restart sing-box`。

## 服务器端接入（sb.json 的 route.rule_set）

```json
{ "tag": "warp-domains",   "type": "remote", "format": "source", "url": "https://raw.githubusercontent.com/ppvia/sb-rules/main/warp-domains.json",   "download_detour": "direct", "update_interval": "1h" },
{ "tag": "warp6-domains",  "type": "remote", "format": "source", "url": "https://raw.githubusercontent.com/ppvia/sb-rules/main/warp6-domains.json",  "download_detour": "direct", "update_interval": "1h" },
{ "tag": "direct-domains", "type": "remote", "format": "source", "url": "https://raw.githubusercontent.com/ppvia/sb-rules/main/direct-domains.json", "download_detour": "direct", "update_interval": "1h" }
```

并开启 `experimental.cache_file.enabled = true`，这样规则集会缓存到本地，GitHub 暂时不可达时仍可启动。

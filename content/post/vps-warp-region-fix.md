---
title: "VPS 机房 IP 被标错地域：用 WARP 修好 Gemini / Google / Claude / Grok"
date: 2026-09-23T15:00:00+08:00
draft: false
categories: ["运维"]
tags: ["warp", "vps", "google", "cliproxy"]
---

机房出口 IP 被 Geo 库标成中国区之后，Google 会直接拦 API，浏览器也会把 `google.com` 302 到 `google.com.hk`。本文记录一次在 Ubuntu 18.04 VPS 上的排查：官方 `warp-cli` 装不上，改用 Docker 跑 WARP，再分别接到 CLIProxyAPI 和 v2ray。

<!--more-->

背景对应 [router-for-me/CLIProxyAPI#3999](https://github.com/router-for-me/CLIProxyAPI/issues/3999)：Antigravity Gemini 返回 `400 User location is not supported for the API use`。实测和请求体无关，是出口 IP 的 ASN/Geo 被 Google 的 regional gate 拒绝。同一台机器上，`google.com` 被跳到香港站，`aistudio.google.com` 被判中国区不可用。

实战环境：HostPapa / ColoCrossing，出口形如 `107.174.210.xxx`，Ubuntu 18.04。

## 1. 先诊断，再动手

```bash
# 出口画像
curl -s https://ipinfo.io/json
curl -s https://www.cloudflare.com/cdn-cgi/trace

# Google 怎么看这台机
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" https://www.google.com/
```

直连若 `302` 到 `google.com.hk?hl=zh-CN`，就是 Geo 标错；经 WARP 若变成 `200`，说明 WARP 能解。

CLIProxyAPI 侧确认是不是同一条 400：

```bash
grep -l "User location is not supported" /root/cliproxy/logs/* | while read f; do
  echo "$f: $(stat -c %y "$f" | cut -d. -f1)"
done
```

## 2. Ubuntu 18.04 上官方 warp-cli 为什么没戏

现象：`warp-cli registration new` 报 `Error: Failed to contact the WARP API`，daemon 日志里是：

```text
JsonError("no variant of enum ExcludeOrInclude found in flattened data")
```

原因：`warp-cli 2024.4` 解析不了新版 WARP API；Cloudflare 的 `bionic` 仓库已冻结，`apt-cache policy` 最高就是 `2024.4.133-1`，官方升级路线断了。

解法：不要在 18.04 宿主机上死磕客户端，改用 Docker 跑新版（下一节）。

## 3. 部署 WARP 代理容器（只做一次）

```bash
docker pull zenexas/warp-cli:latest
docker run -d --name warp --network cliproxy_default \
  -p 127.0.0.1:40000:40000 --restart unless-stopped \
  zenexas/warp-cli:latest
```

验证（这几条都要过）：

```bash
docker logs warp 2>&1 | grep -i "WARP status: Connected"

curl -s -x socks5h://127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace | grep -E "warp|ip="
# 期望：warp=on，ip 为 104.x Cloudflare 段

curl -s -x socks5h://127.0.0.1:40000 -o /dev/null -w "%{http_code}\n" https://www.google.com/
# 期望：200（直连是 302 跳 hk）

# 从 CLIProxyAPI 所在网络访问
docker run --rm --network cliproxy_default curlimages/curl:latest \
  -s -x socks5h://warp:40000 https://www.cloudflare.com/cdn-cgi/trace | grep warp
```

写进 compose，避免重启丢容器（`docker compose config` 校验）：

```yaml
services:
  warp:
    image: zenexas/warp-cli:latest
    container_name: warp
    ports:
      - "127.0.0.1:40000:40000"
    restart: unless-stopped
```

## 4. CLIProxyAPI：按凭证分流

注意字段名：凭证 JSON 里是 **`proxy_url`**（下划线），不是 `proxy-url`。只给受影响的 Antigravity 凭证加代理，Claude / Codex 继续直连。

```bash
cp /root/cliproxy/auths/<name>.json /root/cliproxy/auths/<name>.json.bak-$(date +%Y%m%d)
python3 -c "import json; p='/root/cliproxy/auths/<name>.json'; d=json.load(open(p)); d['proxy_url']='socks5h://warp:40000'; json.dump(d,open(p,'w'),indent=2)"
docker restart cli-proxy-api
```

几点：

- 用 `socks5h`，DNS 也走代理，避免本地解析泄漏。
- 容器互访用主机名 `warp:40000`（同一 Docker 网络）；宿主机用 `127.0.0.1:40000`。
- 验证：`POST /v1beta/models/gemini-3-flash:generateContent` 应从 400 变成 200。

`antigravityProxyURL()` 的优先级是：请求级覆盖 > `auth.ProxyURL` > 全局。官方明确拒绝在上游错误里追加 hint 的 PR（#3999 closed as not_planned），所以只做接入和文档，不去改 `newAntigravityStatusErr`。

## 5. v2ray：宿主机进程走 127.0.0.1

```bash
cp /etc/v2ray/config.json /etc/v2ray/config.json.bak-$(date +%Y%m%d)
```

`outbounds` 追加：

```json
{
  "tag": "warp",
  "protocol": "socks",
  "settings": {
    "servers": [{"address": "127.0.0.1", "port": 40000}]
  }
}
```

`routing.rules` 插在 `api` 规则之后。顺序就是优先级，放太靠后容易被别的规则截走：

```json
{
  "type": "field",
  "domain": [
    "geosite:google",
    "geosite:anthropic",
    "domain:x.ai",
    "domain:grok.com"
  ],
  "outboundTag": "warp"
}
```

注意：

- `geosite.dat`（`/etc/v2ray/bin/`）没有 grok 独立分类，`x-ai` 也不存在，必须写裸域名 `domain:x.ai` 和 `domain:grok.com`（`domain:` 含子域名）。
- 已有 `fix_openai`（openai → direct）时，不要把 openai 写进 warp 规则。集合无交集时，新规则放在它前面也不影响。
- `geoip:cn → block` 等规则保持不动。

```bash
python3 -c "import json,glob; json.load(open('/etc/v2ray/config.json')); [json.load(open(f)) for f in glob.glob('/etc/v2ray/conf/*.json')]; print('JSON-ALL-OK')"
systemctl restart v2ray && sleep 3 && systemctl is-active v2ray
tail -n 5 /var/log/v2ray/error.log
```

日志里应只有常规 default route warning，没有配置报错。客户端上 `aistudio` / `claude.ai` / `grok.com` 应恢复正常，`google.com` 不再跳 `hk`。

## 6. 回滚

```bash
# v2ray
cp /etc/v2ray/config.json.bak-YYYYMMDD /etc/v2ray/config.json && systemctl restart v2ray

# CLIProxyAPI 凭证
cp /root/cliproxy/auths/<name>.json.bak-YYYYMMDD /root/cliproxy/auths/<name>.json && docker restart cli-proxy-api
```

## 7. 下次同类问题先对这张表

| 现象 | 指向 |
|---|---|
| `400 User location is not supported` 只打 Gemini，Claude/Codex 正常 | 出口 ASN 被 Google regional gate 拒，见 #3999 |
| `google.com` 302 到 `google.com.hk?hl=zh-CN` | Geo 库把机房 IP 标成 CN |
| `warp-cli registration new` 失败，日志 `ExcludeOrInclude` | 客户端太旧，18.04 直接换 Docker |
| 配完代理仍 400 | 依次查：规则顺序、`proxy_url` 拼写、`socks5h`、容器名解析、warp 是否 Connected |
| 全站切 WARP | 单点故障、无 fallback、共享 IP 声誉差、免费 WARP 无 SLA。默认直连，只把问题域放进 allowlist |

## 8. 这次机器上落地的东西

- WARP：容器 `warp`，网络 `cliproxy_default`，监听 `127.0.0.1:40000`，`restart: unless-stopped`，已写入 `/root/cliproxy/docker-compose.yml`（原文件 `.bak-20260923`）
- CLIProxyAPI：`/root/cliproxy/auths/antigravity-*.json` 增加 `proxy_url`（`.bak-20260923`）
- v2ray：`/etc/v2ray/config.json` 增加 `warp` 出口和路由（`.bak-20260923`）
- 宿主机旧 `warp-svc`（2024.4）还在跑但已无用，可选 `systemctl disable --now warp-svc`（这次没执行）

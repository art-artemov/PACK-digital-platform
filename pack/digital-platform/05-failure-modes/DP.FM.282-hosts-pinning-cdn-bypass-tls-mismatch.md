---
id: DP.FM.282
name: "Hosts-pinning домена мимо CDN ломает TLS: origin должен отдавать сертификат на прямом маршруте"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-11
source: "session-close 2026-07-11; WP-6 (closed-loop прогон, S9→S2 через /etc/hosts)"
related:
  see_also:
    - "DP.FM.024: git-pull in production (смежно: межсерверные операции)"
tags: [infrastructure, tls, nginx, hosts-pinning, cdn, certificate, ssl, inter-server]
---

# DP.FM.282 — Hosts-pinning домена мимо CDN ломает TLS: origin требует валидный сертификат

## Паттерн

Для ускорения межсерверного трафика домен запинен через `/etc/hosts` напрямую на origin-сервер (мимо CDN/Cloudflare). Origin-сервер отдаёт на прямом соединении дефолтный сертификат хоста (`*.server.internal`), а не сертификат целевого домена.

TLS-клиент с соседнего сервера получает `certificate_verify_failed`, хотя из публичного интернета через CDN домен работает нормально.

## Симптом

```
SSL: CERTIFICATE_VERIFY_FAILED
# или
curl: (60) SSL certificate problem: certificate subject name '*.s2.server.internal'
      does not match target host name 'domain.example.com'
```

**Обманчивость:** домен открывается в браузере (через CDN) → команда думает, что TLS настроен. С соседнего сервера — certificate mismatch.

## Диагностика

```bash
# Проверить наличие пиннинга на целевом сервере
grep domain.example.com /etc/hosts

# Проверить, какой сертификат отдаёт origin напрямую
openssl s_client -connect <origin-ip>:443 -servername domain.example.com 2>&1 | grep subject
```

## Инвариант (fix)

**Для каждого домена, запиненного через `/etc/hosts` к origin:** origin-сервер должен иметь валидный SSL-сертификат для этого домена в своей nginx-конфигурации (`server_name domain.example.com` + сертификат).

```nginx
server {
    listen 443 ssl;
    server_name domain.example.com;  # не только *.server.internal
    ssl_certificate /etc/letsencrypt/live/domain.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.example.com/privkey.pem;
    ...
}
```

## Применимость

Любая межсерверная интеграция через `/etc/hosts` пиннинг (мимо CDN, load balancer, прокси). Требование документировать в `infrastructure.md` при каждом пиннинге.
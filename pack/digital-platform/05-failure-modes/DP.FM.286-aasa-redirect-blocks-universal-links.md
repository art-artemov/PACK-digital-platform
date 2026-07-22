---
id: DP.FM.286
name: "AASA/assetlinks.json отдаёт redirect — Universal Links / App Links могут не верифицироваться"
type: failure-mode
domain: DP
status: active
valid_from: 2026-07-16
source: "session-close 2026-07-14; WP-13 диагностика доменов"
tags: [ios, android, universal-links, app-links, aasa, assetlinks, redirect, mobile, deeplinks]
---

# DP.FM.286 — AASA/assetlinks.json отдаёт HTTP redirect: Universal Links / App Links не верифицируются

## Паттерн

`/.well-known/apple-app-site-association` или `/.well-known/assetlinks.json` возвращает **HTTP redirect** (301/302) вместо прямого ответа.

Apple CDN и Google при первичной верификации **не гарантируют следование редиректу** → Universal Links / App Links для этого домена могут не верифицироваться на части устройств.

## Симптом

- Устройство с кэшированной AASA **работает** (кэш установлен до появления redirect)
- «Новое» устройство (первый вход, переустановка приложения) **не верифицирует домен** → тап по ссылке открывается в Safari/браузере, а не в приложении
- Из браузера домен открывается нормально (redirect прозрачен)

Симптом скрытый: выглядит как «Universal Links работают у одних пользователей и не работают у других».

## Диагностика

```bash
# Проверка прямого ответа
curl -I https://example.com/.well-known/apple-app-site-association
# Ожидаем: HTTP 200
# Ошибка: HTTP 301/302

curl -I https://example.com/.well-known/assetlinks.json
# То же для Android App Links
```

## Инцидент

WP-13 (диагностика deeplinks `mobile` vs `mobile-new`): `https://humdes.com/.well-known/apple-app-site-association` отдавал `301 → www.humdes.com` — оба домена должны отдавать `.well-known/*` напрямую.

## Fix

nginx: оба хоста (`example.com` и `www.example.com`) отдают `.well-known/` напрямую, без редиректа между ними.

```nginx
server {
    server_name example.com www.example.com;

    # Well-known без редиректа — обязательно
    location /.well-known/ {
        root /var/www/html;
        try_files $uri =404;
    }

    # Остальной трафик — можно редирект
    ...
}
```

## Правило

**`.well-known/apple-app-site-association` и `.well-known/assetlinks.json` — только прямой 200, без redirect.**

Любой 3xx = потенциальный сбой верификации Universal Links / App Links.

## Применимость

iOS Universal Links (`apple-app-site-association`), Android App Links (`assetlinks.json`).
Актуально при переходе `http → https`, `example.com → www.example.com`, смене CDN.
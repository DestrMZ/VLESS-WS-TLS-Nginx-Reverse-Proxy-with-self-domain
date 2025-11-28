# Настройка VLESS + WebSocket + TLS  
## Полностью рабочая связка через Nginx Reverse Proxy и собственный домен  
(Без 3x-ui, X-ui и других панелек — чистый Xray + Nginx + Let’s Encrypt)

---

### Подготовка

1. Арендуйте VPS в любой стране (кроме РФ)
2. Купите или получите бесплатно домен любого уровня (.com, .net, .org, .xyz и т.д.)
3. Привяжите домен к IP вашего сервера:

| Тип | Хост        | Значение       |
|-----|-------------|------------------|
| A   | @           | ваш IP VPS       |
| A   | resurse1    | ваш IP VPS       |

> Пример:  
> `yourdomain.com` → IP  
> `resurse1.yourdomain.com` → IP

Подождите 10–15 минут и проверьте:

```bash
ping yourdomain.com
ping resurse1.yourdomain.com
```

Или на сайте: https://dnschecker.org/

---

### Подключаемся к серверу

Рекомендую Termius, PuTTY, MobaXterm или обычный SSH.

Сервер должен быть чистым (без панелей типа 3x-ui).

---

### Шаг 1. Установка Nginx и Certbot

```bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

Проверяем статус Nginx:

```bash
systemctl status nginx
```

Должно быть `active (running)`.

---

### Шаг 2. Создаём временный HTTP-конфиг для поддомена

```bash
sudo nano /etc/nginx/sites-available/resurse1.conf
```

Вставляем:

```nginx
server {
    listen 80;
    server_name resurse1.yourdomain.com;

    location / {
        return 200 'ok';
        add_header Content-Type text/plain;
    }
}
```

Активируем и перезапускаем:

```bash
sudo ln -s /etc/nginx/sites-available/resurse1.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default 2>/dev/null || true
sudo nginx -t && sudo systemctl reload nginx
```

Проверка:

```bash
curl http://resurse1.yourdomain.com
# Должно вернуть: ok
```

---

### Шаг 3. Получаем бесплатный SSL-сертификат Let’s Encrypt

```bash
sudo certbot --nginx -d resurse1.yourdomain.com
```

- Введите email  
- Согласитесь с условиями (Y)  
- Рассылка — на ваш выбор (N)  
- Выберите вариант 2 (Redirect HTTP → HTTPS)

Certbot сам добавит SSL в конфиг SSL и сделает редирект.

Проверка:

```bash
curl -v https://resurse1.yourdomain.com
# Должно быть HTTP/2 200 и тело "ok"
```

---

### Шаг 4. Установка Xray-core (последняя версия)

```bash
bash <(curl -Ls https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh)
```

Проверка:

```bash
which xray
systemctl status xray
```

---

### Шаг 5. Генерируем UUID и настраиваем VLESS+WS

```bash
uuidgen
# Скопируйте полученный UUID
```

Редактируем конфиг Xray:

```bash
sudo nano /usr/local/etc/xray/config.json
```

Заменяем содержимое на:

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 10000,
      "listen": "127.0.0.1",
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "ВАШ-UUID-ЗДЕСЬ",
            "flow": "",
            "email": "main"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "ws",
        "security": "none",
        "wsSettings": {
          "path": "/ws",
          "headers": {
            "Host": "resurse1.yourdomain.com"
          }
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "blocked"
    }
  ]
}
```

Сохраняем и перезапускаем:

```bash
sudo systemctl restart xray
sudo systemctl status xray
```

---

### Шаг 6. Финальный конфиг Nginx (TLS + прокси на Xray)

```bash
sudo nano /etc/nginx/sites-available/resurse1.conf
```

Полный рабочий конфиг:

```nginx
server {
    listen 80;
    server_name resurse1.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name resurse1.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/resurse1.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/resurse1.yourdomain.com/privkey.pem;

    # Заглушка для обычных запросов
    location / {
        return 404;
    }

    # Проксируем WebSocket на Xray
    location /ws {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Проверка и перезапуск:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

### Готово! Настройка клиента (Nekobox, v2rayNG, Streisand, Hiddify и др.)

```
Тип: VLESS
Адрес: resurse1.yourdomain.com
Порт: 443
UUID: ваш-uuid-из-конфига
Шифрование: none
Тип передачи: WebSocket (ws)
Путь (Path): /ws
Host / SNI / Header Host: resurse1.yourdomain.com
TLS: включён
Allow Insecure: выключен (если сертификат валидный)
```

Сохраняете — подключаетесь — наслаждаетесь чистым, быстрым и полностью своим VLESS+WS+TLS сервером.

Удачи и стабильного пинга! 🚀

# VLESS + WebSocket + TLS + Nginx Reverse Proxy

## 0. Что нужно заранее
Перед началом убедитесь, что у вас есть:

	•	VPS с Ubuntu 22.04 или 24.04 (не РФ)
	•	Собственный домен (любой: .online, .site, .com и т.д.)
	•	SSH-доступ к серверу
	•	Поддомен для прокси
	
## 1. Настройка DNS

Зайдите в панель управления вашим доменом и создайте A-записи:

| Host	| Тип  Значение |

| @		| IP вашего VPS |

| cdn 	| IP вашего VPS |

Проверка:
```
nslookup cdn.your-domain.com 8.8.8.8
```
Должен вернуться ваш IP.

## 2. Установка nginx и certbot

Подключитесь к серверу через ssh, либо через программы по типу Termius.
```
ssh root@IP_вашего_сервера
```
Установите nginx и certbot:
```
apt update && apt upgrade -y
apt install -y nginx certbot python3-certbot-nginx
```
Проверьте, что nginx работает:
```
systemctl status nginx
```
⸻

## 3. Создание HTTP-конфига nginx

Создаём файл:
```
nano /etc/nginx/sites-available/cdn.conf
```
Вставьте:
```
server {
    listen 80;
    server_name cdn.your-domain.com;

    location / {
        return 200 'ok';
        add_header Content-Type text/plain;
    }
}
```
Подключаем сайт:
```
ln -s /etc/nginx/sites-available/cdn.conf /etc/nginx/sites-enabled/cdn.conf
rm /etc/nginx/sites-enabled/default 2>/dev/null || true
```
Проверяем:
```
nginx -t
systemctl reload nginx
```
Тест:
```
curl http://cdn.your-domain.com
```
Должно вернуть ok

⸻

## 4. Выдача HTTPS-сертификата Let’s Encrypt

Запускаем certbot:
```
certbot --nginx -d cdn.your-domain.com
```
Выбираем:

	•	Email — любой рабочий
	•	Agree — Y
	•	Share email — N
	•	Redirect — YES

Проверяем:
```
curl -v https://cdn.your-domain.com
```
Перейдите на сайт - если высветилось ok, идем дальше

⸻

## 5. Установка Xray-core
```
bash <(curl -Ls https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh)
```
Проверка:
```
which xray
systemctl status xray
```

## 6. Генерация UUID для пользователей

Для каждого устройства:
```
uuidgen
```
Пример:
```
85226060-dcdf-4a90-8f4a-8cf30f0974d2
```

⸻

## 7. Конфиг Xray — VLESS + WS (без TLS)

Открываем:
```
nano /usr/local/etc/xray/config.json
```
Пример с несколькими пользователями:

```
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
            "id": "UUID-MAIN",
            "flow": "",
            "email": "main"
          },
          {
            "id": "UUID-MACBOOK",
            "flow": "",
            "email": "macbook"
          },
          {
            "id": "UUID-IPHONE14",
            "flow": "",
            "email": "iphone14"
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
            "Host": "cdn.your-domain.com"
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

Проверка:
```
xray run -test -config /usr/local/etc/xray/config.json
```

Если ошибок нет:
```
systemctl restart xray
```

⸻

## 8. Настройка nginx как reverse-proxy для Xray

Правим nginx-конфиг:
```
nano /etc/nginx/sites-available/cdn.conf
```

Полный рабочий вариант:
```
server {
    listen 80;
    server_name cdn.your-domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name cdn.your-domain.com;

    ssl_certificate /etc/letsencrypt/live/cdn.your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cdn.your-domain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        return 404;
    }

    location /ws {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:10000;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $server_name;
    }
}
```

Перезагружаем:
```
nginx -t
systemctl reload nginx
```

⸻

## 9. Добавление новых пользователей

Каждый раз нужно вручную прописывать нового пользователя. Также можно сделать автоскрипт, собственно это фундамент который можете модернизировать как вам удобно, мб позже добавлю скрипт

1.	Делаешь новый UUID командой:
```
uuidgen
```
2.	Добавляешь в "clients": [ ]
3.	Перезапускаешь Xray:
```
systemctl restart xray
```
⸻

## 10. Пример ключа, необходимо заменить UUID на пользовательский
Формат:
```
vless://UUID@cdn.your-domain.com:443?encryption=none&flow=&type=ws&host=cdn.your-domain.com&path=%2Fws&security=tls&sni=cdn.your-domain.com#Name
```
⸻

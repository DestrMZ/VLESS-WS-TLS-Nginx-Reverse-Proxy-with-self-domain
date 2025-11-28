# VLESS-WS-TLS-Nginx-Reverse-Proxy-with-self-domain
Настройка связки VLESS over WebSocket over TLS (Reverse Proxy via Nginx)

Первым шагом нужно арендовать VPS, в любой для Вас удобной точки(кроме России). После, для связки с self-domain нам необходим сам домен, при желании есть сайты где можно их взять бесплатно, но в моем случае я купил его на довольно изместном Российском сайте по продаже доменов. 

Далее, как у нас будет на руках домен и ip нашел VPS, необходимо привязать наш домен к собственно ip сервера, через панель на сайте где вы преобрели домент сделать нужно следующем образом. 

Нажать "Добавить ресурсные записи" указать тип "А" и перое поле проставить как @ -> "your ip VPS", а также добавить еще одну запись для поддомена, можете назвать допустим "resurse(как вам нравится)" -> "your ip VPS", по итогу у вас должно быть две записи. 
После всех этих действий необходимо подождать 10-15 минут пока домены добавятся в общий доменны реестр. По итогу проверяем наши домены, открываем CMD или PowerShell и вводим команду

- ping yourdomain.com
- ping resurse1.yourdomain.com 

Обязательно проверяем оба доменных адреса. Есть сайт где вы можете указать свой домент и убедиться в его доступности https://dnschecker.org/

Отлично, поздравляю вы зарегистрировали свой собственный DNS.

Переходим к следующему шагу, и идем на наш сервер, подключается к серверу используя любые удобные для вас методы, я лично использую Termius, а вы смотрите сами.
У нас должен быть желательно полностью чистый сервер, без 3x-ui панелей и тому подобные.

Первым шагом проверяем обновление устанавливаем NGINX и Certbot(для получения легитимного сертификата).

apt update
apt install -y nginx certbot python3-certbot-nginx

Проверяем что Nginx запустился командой 
systemctl status nginx

Должно быть active (running).

Шаг 3.
Делаем простой конфиг для HTTP на 80 порту. Создаем конфиг для нашего поддомена
nano /etc/nginx/sites-available/node1.conf

Туда нужно будет списать следующее

server {
    listen 80;
    server_name resurse1.yourdomain.com; # здесь указываем именно поддомен
 
    простая заглушка, чтобы certbot смог проверить домен
    location / {
        return 200 'ok';
        add_header Content-Type text/plain;
    }
}
Сохраняем (Ctrl+O, Enter, Ctrl+X).

Далее, подключаем сам сайт и выключаем дефолтный, вписываем следующие команды

ln -s /etc/nginx/sites-available/resurse1.conf /etc/nginx/sites-enabled/resurse1.conf
rm /etc/nginx/sites-enabled/default 2>/dev/null || true

nginx -t
systemctl reload nginx

Если nginx -t выдал syntax is ok и test is successful — отлично.
По итогу проверяем сервер 
curl http://resurse1.yourdomain.online
Должно вывести ok.

Шаг 4. 
Получаем валидный сертификат Let's Enctypto, выполняем слудюющую команду в консоли
certbot --nginx -d resurse1.yourdomain.online

Он попросит:
email — вводишь любой рабочий
согласие с условиями — Y
hare email — как хочешь (N можно)
про redirect — выбираешь вариант с перенаправлением на https

После успешного выполнения certbot сам пропишет SSL в конфиг и перезапустит nginx.
curl -v https://resurse1.yourdomain.online
В ответе должно быть что-то вроде HTTP/2 200 и тело ok.

Шаг 5. 
Установка Xray-core
bash <(curl -Ls https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh)
(если спросит — всё по умолчанию)

После этого проверяем что все окей
which xray
systemctl status xray
Сервис может быть пока с дефолтным конфигом — мы его перепишем.

Шаг 6.
Настроить VLESS+WS инбаунд в Xray

Необходимо сгенерировать uuid и куда-нибудь записать, в консоли выполняем команду 
uuidgen

Далее, открываем сам конфиг Xray
nano /usr/local/etc/xray/config.json

Там будет приблизительно такой json

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
            "id": "YOUR-UUID-HERE", # Сюда вставляете свой uuid, который вы сгенерировали
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
            "Host": "resurse1.yourdomain.com" #Здесь должен быть ваш поддомен, проверяйте
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

В целом можете скопировать этот вариант, подставить свои данные, после удалить тот дефолтный и вставить свой, если все ок.
Сохраняем (Ctrl+O, Enter, Ctrl+X) и перезапускаем ядро

systemctl restart xray
systemctl status xray

Статус должен быть active (running) без ошибок.

Шаг 7.
Настроить Nginx как TLS-прокси к Xray
Теперь правим тот же файл /etc/nginx/sites-available/resurse1.yourdomain.conf, в котором уже есть блок с SSL от certbot.

Откройте 
nano /etc/nginx/sites-available/resurse1.conf

Приведу пример, как он должен выглядеть в итоге
(важен второй server с listen 443 ssl):

server {
    listen 80;
    server_name resurse1.yourdomain.online;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name resurse1.yourdomain.online;

    ssl_certificate /etc/letsencrypt/live/resurse1.yourdomain.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/resurse1.yourdomain.online/privkey.pem;

    # обычный сайт-заглушка
    location / {
        return 404;
    }

    # VLESS через WebSocket
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
Пути к сертификатам certbot иногда подставляет сам, главное — не трогать их, просто добавить location /ws и редирект с 80 на 443.

Проверяем и перезапускаем nginx
nginx -t
systemctl reload nginx

Самая сложная часть закончена, по сути осталось настроить клиентствую сторону и можно тестировать

Открываем конфиг Xray

Пример:
cat /usr/local/etc/xray/config.json

И здесь будут поля client
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
            "id": "e859586e-22d1-4d80-a8e9-748a266125d2",
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

Это то, как у вас будет выглядить этот конфиг, вы можете в нем добавлять новых пользователей, задавать им параметры, информацию можете найти в интернете. Для того чтобы убедиться что вам сервер настроен и работает корректно, вам нужно открыть клинсткое приложение, будто на телефоне, компьютере где угодно.

Параметры, если будете забивать руками:
Тип: VLESS
Адрес: resurse1.yourdomain.com
Порт: 443
UUID: e859586e-22d1-4d80-a8e9-748a266125d2
Encryption: none
Transport: WebSocket (ws)
Path: /ws
Host (или Host / Header): resurse1.yourdomain.com
TLS: включить
SNI: resurse1.yourdomain.com








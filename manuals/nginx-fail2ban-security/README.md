# Безопасность без бюджета: пошаговое руководство по защите веб-сервера силами Nginx и Fail2ban


У моего приятеля случилась беда: его сайт взломали (написан на WordPress). Для его маленького бизнеса это очень чувствительно. Хостинговая компания предложила установить и настроить WAF, но для него это оказалось больно по деньгам.

В общем, покажу, как мы настроили Nginx, Fail2ban и полностью закрыли вопрос **абсолютно бесплатно**. Размещу это здесь, так как в сети нормальной выжимки не найти, а эти индусы с их вечным «хелоу евреван» на YouTube уже достали. Собрал надежную **четырехуровневую крепость** силами стандартного и бесплатного стека, о которой почему-то редко пишут без лишней «воды».

---

## Линия обороны 1. Nginx как «Умный забор»

Веб-сервер умеет не только раздавать картинки, но и эффективно отбивать атаки на подлете.

### Настройки в глобальном блоке `http { ... }` (Файл `nginx.conf`)

```nginx
http {
    # 1. МАСКИРОВКА: Скрываем точную версию Nginx, лишая хакеров первой подсказки
    server_tokens off;

    # 2. ЗАЩИТА ПАМЯТИ: Жестко ограничиваем размер загрузок до 10 МБ
    client_body_buffer_size      128k;
    client_header_buffer_size    1k;
    client_max_body_size         10M;

    # 3. ТАЙМАУТЫ: Если клиент подключился и молчит 10 секунд — отключаем его (Защита от Slowloris)
    client_body_timeout          10;
    client_header_timeout        10;

    # 4. АНТИ-DDOS: Считаем скорость кликов с одного IP-адреса
    limit_req_zone $binary_remote_addr zone=site_limit:10m rate=10r/s;  # Для всего сайта: не более 10 запр/сек
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=2r/m;  # Для входа: не более 2 попыток в минуту
}
```

### Настройки внутри блока сайта `server { ... }` (Файл конфигурации домена)

```nginx
server {
    listen 443 ssl;
    server_name test.by; # Укажите здесь ваш домен

    # 5. HTTP-ЗАГОЛОВКИ БЕЗОПАСНОСТИ
    add_header X-Frame-Options "SAMEORIGIN" always; # Защита от кликджекинга
    add_header X-Content-Type-Options "nosniff" always; # Запрет браузеру угадывать тип файла
    add_header Referrer-Policy "strict-origin-when-cross-origin" always; # Скрытие путей перехода
    add_header Permissions-Policy "camera=(), microphone=()" always; # Блокировка доступа к девайсам
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always; # Принудительный HTTPS

    # Контроль CORS (заменяем trust.by на конкретного доверенного партнера)
    add_header Access-Control-Allow-Origin "https://trust.by";

    # Политика CSP (Разрешаем скрипты только от себя, Яндекса и Google)
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' *.test.by *.yandex.ru *.google.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; frame-ancestors 'self';" always;

    # Защита сессионных кук от кражи скриптами
    proxy_cookie_path / "/; HTTPOnly; Secure; SameSite=Strict";

    # Применяем общий лимит кликов на весь сайт
    limit_req zone=site_limit burst=20 nodelay;

    location / {
        proxy_pass http://localhost:3000; # Путь к вашему приложению

        # Блокировка известных хакерских утилит-сканеров по имени (User-Agent)
        if ($http_user_agent ~* (sqlmap|nikto|dirbuster|nmap)) {
            return 403;
        }
    }

    # Защита админки WordPress (включаем жесткий лимит — не более 2 попыток в минуту)
    location = /wp-login.php {
        limit_req zone=login_limit burst=5 nodelay;
        proxy_pass http://localhost:3000/login;
    }

    # Запрет на выполнение любых программных скриптов в папке загрузок пользователя
    location /uploads/ {
        location ~ \.(php|pl|py|jsp|sh|cgi|exe)$ {
            deny all;
            return 403;
        }
    }
}
```

---

## Линия обороны 2. Фильтрация трафика (WAF)

Что делать, если хакер действует аккуратно, не шумит, лимиты не превышает, но отправляет всего один, но смертельный запрос с SQL-инъекцией прямо в форму поиска? Стандартный Nginx такое пропустит. Для этого мы встраиваем в него бесплатный современный модуль **Coraza WAF**.

### Установка модуля Coraza WAF для Nginx (Ubuntu/Debian)

```bash
sudo add-apt-repository ppa:coraza/nginx -y
sudo apt update
sudo apt install libnginx-mod-http-coraza -y
```

> **Прим.:** Для активации модуля добавьте строку `load_module modules/ngx_http_coraza_module.so;` в самый верх вашего файла `nginx.conf`.

Чтобы не писать правила защиты с нуля, мы скачиваем и подключаем официальный бесплатный пак правил **OWASP Core Rule Set (CRS)** — мировой стандарт кибербезопасности.

### Как быстро скачать пак правил OWASP CRS через терминал

```bash
cd /etc/nginx/coraza/
sudo git clone https://github.com/coreruleset/coreruleset.git owasp-crs
sudo cp owasp-crs/crs-setup.conf.example owasp-crs/crs-setup.conf
```

### Настройки в файле конфигурации файрвола `/etc/nginx/coraza/coraza.conf`

```nginx
# Включаем WAF в боевой режим (On — блокировать атаки, DetectionOnly — только записывать в лог)
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess On
SecAuditLog /var/log/nginx/coraza_audit.log

# Подключаем скачанный пак правил OWASP CRS одной строчкой
Include /etc/nginx/coraza/owasp-crs/crs-setup.conf
Include /etc/nginx/coraza/owasp-crs/rules/*.conf
```

После установки Coraza проверьте:

```bash
sudo nginx -t                # Проверка конфигурации
sudo systemctl reload nginx  # Перезагрузка
```

---

## Линия обороны 3. Fail2ban как «Автоматический охранник»

Программа, которая круглосуточно читает системные логи сайта. Если кто-то настойчиво перебирает пароли — она автоматически банит его IP на уровне всей системы.

### Установка Fail2ban через терминал

```bash
sudo apt update
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Шаг 1. Создаем фильтр уязвимостей в `/etc/fail2ban/filter.d/nginx-bf.conf`

```ini
[Definition]
# Замечаем IP, которые получили ошибку 401 (неверный пароль) на странице входа
failregex = ^<HOST> -.*"POST .*/login.*" 401
# Замечаем ботов, сканирующих сайт в поисках стандартных лазеек WordPress
            ^<HOST> -.*"GET .*wpad.dat.*" 404
            ^<HOST> -.*"GET .*wp-admin.*" 404
```

### Шаг 2. Задаем логику наказания в `/etc/fail2ban/jail.local`

```ini
[nginx-bf]
enabled  = true
port     = http,https
filter   = nginx-bf
logpath  = /var/log/nginx/access.log
maxretry = 5                           # Если один IP сделал 5 плохих попыток...
findtime = 60                          # ...в течение всего 60 секунд...
bantime  = 3600                        # ...полностью заблокировать ему доступ к серверу на 1 час.
```

> **Прим.:** После сохранения файла примените изменения командой `sudo systemctl restart fail2ban`.

---

## Линия обороны 4. Дополнительные вещи (Системная гигиена)

Затыкаем скрытые лазейки в Linux:

- **Отключаем IPv6:** Большинство настраивает лимиты и правила Fail2ban только для адресов IPv4. Про IPv6 часто забывают, оставляя его без защиты. Если ваш сайт отлично работает на IPv4 — полностью отключите IPv6 в настройках ОС.
- **Изолируем базу данных:** Закройте порт базы данных фаерволом намертво для внешнего мира. База должна общаться только внутри сервера. Для работы админа используйте защищенные SSH-туннели.

Эти инструменты я не настраивал в первом подходе, но они очень могут пригодиться для усиления контура безопасности.

- **Секретный стук (Port Knocking):** Сделайте порт управления сервером (SSH) абсолютно невидимым. Сервер будет казаться полностью пустым, пока администратор не отправит на него серию пустых сетевых пакетов на определенные порты в строгом порядке. Только после этого «секретного стука» сервер откроет дверь лично вам на несколько секунд.
- **Автоматические черные списки IP:** Используем утилиту **IPset** (она проверяет миллионы IP мгновенно, не нагружая процессор). Пишем простой скрипт, который раз в сутки по Cron скачивает базу адресов известных ботнетов (например, Firehol Level 1).

### Реализация скрипта автоматического бана (`/usr/local/bin/update_blacklist.sh`)

```bash
#!/bin/bash

# Скачиваем свежий список плохих IP
curl -s "https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/firehol_level1.netset" -o /tmp/blacklist.txt

# Создаем новый список (если существует - удаляем)
ipset destroy blacklist 2>/dev/null
ipset create blacklist hash:ip

# Добавляем IP в список
while read ip; do
    # Пропускаем комментарии и пустые строки
    [[ -z "$ip" || "$ip" == \#* ]] && continue
    ipset add blacklist "$ip" 2>/dev/null
done < /tmp/blacklist.txt

# Удаляем временный файл
rm -f /tmp/blacklist.txt

echo "Готово! Заблокировано IP: $(ipset list blacklist | grep -c '^[0-9]')"
```

Чтобы фаервол Linux молча и без лишних ответов сбрасывал входящие пакеты от серверов из этого черного списка, добавляем в систему одну железную команду iptables:

```bash
sudo iptables -I INPUT -m set --match-set blacklist src -j DROP
```

Добавьте в crontab (`crontab -e`):

```cron
# Обновлять список каждый день в 3:00
0 3 * * * /путь/к/скрипту/update_blacklist.sh > /dev/null 2>&1
```

---

## Заключение

Безопасность — это не всегда про огромные чеки и enterprise-лицензии. Правильная, осознанная настройка базового бесплатного софта закрывает большинство автоматических угроз, экономя ресурсы и нервы.
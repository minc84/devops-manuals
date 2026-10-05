# Безопасность без бюджета: пошаговое руководство по защите веб-сервера силами Nginx и Fail2ban

Практическое руководство по настройке защиты веб-сервера от распространенных сетевых угроз, перебора паролей и DDoS-атак с использованием бесплатного стандартного стека инструментов.

## Линия обороны 1. Nginx как умный забор

Веб-сервер Nginx позволяет эффективно отбивать атаки на подлете, ограничивать скорость запросов и маскировать системные данные от сканеров.

### Настройки в глобальном блоке http (Файл nginx.conf)

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

### Настройки внутри блока сайта server (Файл конфигурации домена)

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
        
        # Блокировка известных хакерских утилиты-сканеров по имени (User-Agent)
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

Если злоумышленник действует аккуратно и не превышает лимиты запросов, но отправляет SQL-инъекцию или XSS-нагрузку, стандартный Nginx это пропустит. Для глубокого анализа трафика встраивается бесплатный модуль Coraza WAF с набором правил OWASP Core Rule Set (CRS).

### Установка модуля Coraza WAF для Nginx (Ubuntu/Debian)

```bash
sudo add-apt-repository ppa:coraza/nginx -y
sudo apt update
sudo apt install libnginx-mod-http-coraza -y
```

*Примечание:* Для активации модуля необходимо добавить строку `load_module modules/ngx_http_coraza_module.so;` в самый верх глобального конфигурационного файла `nginx.conf`.

### Загрузка пака правил OWASP CRS через терминал

```bash
cd /etc/nginx/coraza/
sudo git clone https://github.com owasp-crs
sudo cp owasp-crs/crs-setup.conf.example owasp-crs/crs-setup.conf 
```

### Настройки в файле конфигурации файрвола /etc/nginx/coraza/coraza.conf

```ini
# Включаем WAF в боевой режим (On — блокировать атаки, DetectionOnly — только записывать в лог)
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess On
SecAuditLog /var/log/nginx/coraza_audit.log

# Подключаем скачанный пак правил OWASP CRS одной строчкой
Include /etc/nginx/coraza/owasp-crs/crs-setup.conf
Include /etc/nginx/coraza/owasp-crs/rules/*.conf
```

После сохранения настроек проверьте конфигурацию и перезапустите веб-сервер:

```bash
sudo nginx -t  # Проверка конфигурации
sudo systemctl reload nginx  # Перезагрузка
```

---

## Линия обороны 3. Fail2ban как автоматический охранник

Fail2ban анализирует системные логи веб-сервера в реальном времени. Если один и тот же IP-адрес совершает серию подозрительных действий или перебирает пароли, утилита автоматически блокирует его на уровне сетевого экрана.

### Установка Fail2ban через терминал

```bash
sudo apt update
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban 
```

### Шаг 1. Создание фильтра уязвимостей

Создайте файл `/etc/fail2ban/filter.d/nginx-bf.conf` со следующим содержимым:

```ini
[Definition]
# Замечаем IP, которые получили ошибку 401 (неверный пароль) на странице входа
failregex = ^<HOST> -.*"POST .*/login.*" 401
# Замечаем ботов, сканирующих сайт в поисках стандартных лазеек WordPress
            ^<HOST> -.*"GET .*wpad.dat.*" 404
            ^<HOST> -.*"GET .*wp-admin.*" 404 
```

### Шаг 2. Настройка логики блокировки

Добавьте конфигурацию тюрьмы (jail) в файл `/etc/fail2ban/jail.local`:

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

После применения изменений перезапустите службу:

```bash
sudo systemctl restart fail2ban
```

---

## Линия обороны 4. Системная гигиена и расширенная защита

Дополнительные шаги для изоляции критических узлов операционной системы:

1. **Отключение IPv6:** Если инфраструктура полностью работает на IPv4, рекомендуется отключить сетевой протокол IPv6 в настройках ОС, чтобы избежать ситуации, когда правила Fail2ban или лимиты Nginx обрабатывают только IPv4-трафик, оставляя IPv6 без контроля.
2. **Изоляция базы данных:** Порты СУБД должны быть закрыты для внешних подключений. Доступ к базе данных извне должен осуществляться исключительно через защищенные SSH-туннели.

### Продвинутые инструменты автоматизации

* **Секретный стук (Port Knocking):** Позволяет сделать порт управления SSH полностью закрытым для внешних сканеров, пока администратор не отправит строго определенную последовательность пакетов на скрытые порты.
* **Автоматические черные списки IP:** Использование утилиты `ipset` для мгновенной обработки списков вредоносных IP-адресов без избыточной нагрузки на CPU.

#### Скрипт автоматического обновления черных списков (/usr/local/bin/update_blacklist.sh)

```bash
#!/bin/bash

# Скачиваем свежий список плохих IP
curl -s "https://githubusercontent.com" -o /tmp/blacklist.txt

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

Для блокировки пакетов из этого списка на уровне ядра выполните команду:

```bash
sudo iptables -I INPUT -m set --match-set blacklist src -j DROP 
```

Для регулярного обновления списка добавьте задачу в планировщик `crontab -e`:

```cron
# Обновлять список каждый день в 3:00
0 3 * * * /usr/local/bin/update_blacklist.sh > /dev/null 2>&1
```

Правильная и последовательная настройка базовых механизмов операционной системы и веб-сервера позволяет нейтрализовать большинство автоматизированных угроз и защитить веб-ресурс без затрат на дорогостоящие коммерческие лицензии.

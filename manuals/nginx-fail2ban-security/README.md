# Безопасность без бюджета: пошаговое руководство по защите веб-сервера (Nginx + Fail2ban)

Практическое руководство по базовой защите веб-инфраструктуры от автоматического сканирования, брутфорса и базовых DDoS-атак с использованием встроенных инструментов Linux.

---

## Архитектура и стек защиты
* Reverse Proxy: Nginx (веб-сервер / балансировщик).
* IDS/IPS элемент: Fail2ban (анализ логов и автоматическая блокировка атакующих IP через iptables/nftables).
* Платформа: Ubuntu Server / openSUSE Linux.

---

## 1. Тюнинг безопасности Nginx

### Ограничение зоны лимитов (Rate Limiting)
Чтобы защитить формы авторизации и API от перебора паролей (брутфорса) и DDoS-атак, настраиваем ограничение количества запросов. 

Открываем глобальный конфиг `/etc/nginx/nginx.conf` и в секцию `http` добавляем зону для лимитов:
```nginx
http {
    # Ограничение: 1 запрос в секунду на один IP-адрес
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=1r/s;
    
    # Скрываем версию Nginx в заголовках ответов (Security by Obscurity)
    server_tokens off;
}
```

Применяем зону лимита к конкретному блоку (`location`) в конфиге сайта (например, на страницу логина):
```nginx
location /login/ {
    # Разрешаем всплеск до 5 запросов, задержка остальных
    limit_req zone=login_limit burst=5 nodelay;
    proxy_pass http://backend_server;
}
```

---

## 2. Развертывание и настройка Fail2ban

Устанавливаем Fail2ban в систему:
```bash
# Для Ubuntu/Debian:
sudo apt install fail2ban -y

# Для openSUSE/RHEL:
sudo zypper install fail2ban
```

### Конфигурация локальной тюрьмы (jail.local)
Никогда не правим файл `jail.conf` напрямую. Создаем локальную копию конфигурации для наших правил:
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Добавляем или активируем фильтр для защиты портов Nginx от агрессивных ботов:
```ini
[nginx-http-auth]
enabled  = true
port     = http,https
filter   = nginx-http-auth
logpath  = /var/log/nginx/error.log
maxretry = 3
findtime = 600
bantime  = 3600
```
* maxretry = 3 — разрешено всего 3 ошибки авторизации.
* findtime = 600 — окно анализа логов составляет 10 минут.
* bantime = 3600 — нарушитель блокируется в брандмауэре на 1 час (3600 секунд).

Запускаем и включаем службу в автозагрузку:
```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
```

---

## 3. Мониторинг и управление банами

Проверка текущего статуса защиты и списка заблокированных IP-адресов:
```bash
sudo fail2ban-client status nginx-http-auth
```

Если вы случайно заблокировали сами себя или доверенного пользователя, разбанить IP можно одной командой:
```bash
sudo fail2ban-client set nginx-http-auth unbanip X.X.X.X
```

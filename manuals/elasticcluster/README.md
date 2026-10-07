# Кластер Elasticsearch 7.17.27

Инструкция по развёртыванию отказоустойчивого кластера Elasticsearch из трёх нод с Nginx-балансировщиком, настройкой снапшотов и миграцией данных из трёх независимых одиночных БД через `_reindex`.

---

## Архитектура

| IP | Hostname | Роль | Примечание |
|----|----------|------|------------|
| 10.0.0.11 | node1 | master, data, ingest | Основная нода |
| 10.0.0.12 | node2 | master, data, ingest | Основная нода |
| 10.0.0.13 | node3 | master, data, ingest | Основная нода |
| 10.0.0.41 | es-proxy-cluster | Nginx-балансировщик | Домен: `entrypoint.es-proxy.example.com` |

**Источники для reindex:**

- 10.0.0.153:9200
- 10.0.0.154:9200
- 10.0.0.155:9200

**Параметры кластера:**

- Имя кластера: `example-prod`
- Версия Elasticsearch: `7.17.27` (одинаковая на источнике и приёмнике)
- Java: `17`

---

## Требования

| Параметр | Значение |
|----------|----------|
| ОС | AlmaLinux 9 / RHEL 9 / CentOS 9 |
| RAM на ноду | ≥ 64 ГБ |
| Диск | SSD, ≥ 500 ГБ |
| Java | 17 |
| Elasticsearch | 7.17.27 |

На всех нодах был отключён фаервол — порты 9200 (HTTP) и 9300 (transport) открыты без ограничений.

---

## Подготовка ОС

Выполнить на **всех трёх нодах**.

### Настройка ядра и лимитов

```bash
# Увеличить лимит mmap
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
sysctl -p

# Лимиты для пользователя elasticsearch
cat >> /etc/security/limits.conf <<EOF
elasticsearch soft nofile 65536
elasticsearch hard nofile 65536
elasticsearch soft nproc  4096
elasticsearch hard nproc  4096
elasticsearch soft memlock unlimited
elasticsearch hard memlock unlimited
EOF
```

### Отключение swap

```bash
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
```

### Базовые пакеты

```bash
dnf update -y
dnf install -y less nano wget curl openssh-server openssh-clients
```

---

## Установка Elasticsearch

Elasticsearch 7.17.x требует Java 17. Если система по умолчанию использует более новую версию (например, Java 21), Elasticsearch может падать при старте — явно указываем Java 17 через `ES_JAVA_HOME`.

### Java 17

```bash
dnf install -y java-17-openjdk-devel

# Проверить путь к Java 17
ls /usr/lib/jvm/ | grep java-17

# При необходимости переключить альтернативу
alternatives --config java
```

### Установка пакета

**Официальный источник Elastic:**

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.27-x86_64.rpm
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.27-x86_64.rpm.sha512
shasum -a 512 -c elasticsearch-7.17.27-x86_64.rpm.sha512
yum install -y elasticsearch-7.17.27-x86_64.rpm
```

**Зеркало Aliyun (если официальный недоступен):**

```bash
wget https://mirrors.aliyun.com/elasticstack/yum/elastic-7.x/7.17.27/elasticsearch-7.17.27-x86_64.rpm
yum install -y elasticsearch-7.17.27-x86_64.rpm
```

### Указать Java для Elasticsearch

```bash
nano /etc/sysconfig/elasticsearch
```

Прописать актуальный путь:

```bash
ES_JAVA_HOME=/usr/lib/jvm/java-17-openjdk
# либо полный путь
ES_JAVA_HOME=/usr/lib/jvm/java-17-openjdk-17.0.15.0.6-3.el9.alma.1.x86_64
```

### Разрешить блокировку памяти в systemd

```bash
mkdir -p /etc/systemd/system/elasticsearch.service.d
cat > /etc/systemd/system/elasticsearch.service.d/override.conf <<EOF
[Service]
LimitMEMLOCK=infinity
EOF

systemctl daemon-reload
```

### Настройка JVM heap

```bash
nano /etc/elasticsearch/jvm.options.d/heap.options
```

Содержимое:

```
-Xms25g
-Xmx25g
```

**Правила:**

- `-Xms` и `-Xmx` должны быть равны.
- Heap не должен превышать 31 ГБ.
- Ориентир: heap = ~50% RAM, но не более 31 ГБ.

---

## Конфигурация Elasticsearch

Открыть основной конфиг:

```bash
nano /etc/elasticsearch/elasticsearch.yml
```

### Для node1 (10.0.0.11)

```yaml
cluster.name: example-prod
node.name: node1
node.roles: [ master, data, ingest ]

path.data: /var/lib/elasticsearch/data
path.logs: /var/lib/elasticsearch/logs
path.repo: ["/var/elastic_backups"]

network.host: ["10.0.0.11", "_local_"]
http.port: 9200
transport.port: 9300

discovery.seed_hosts:
  - "10.0.0.11:9300"
  - "10.0.0.12:9300"
  - "10.0.0.13:9300"

cluster.initial_master_nodes:
  - "node1"
  - "node2"
  - "node3"

reindex.remote.whitelist:
  - "10.0.0.153:9200"
  - "10.0.0.154:9200"
  - "10.0.0.155:9200"
```

### Для node2 (10.0.0.12)

Отличаются только две строки:

```yaml
node.name: node2
network.host: ["10.0.0.12", "_local_"]
```

### Для node3 (10.0.0.13)

```yaml
node.name: node3
network.host: ["10.0.0.13", "_local_"]
```

### Директория снапшотов

```bash
# Смонтировать NFS-шару во всех трёх нодах
mount -t nfs nfs-server:/export/elastic_backups /var/elastic_backups
# Убедиться, что путь смонтирован
df -h /var/elastic_backups
chown -R elasticsearch:elasticsearch /var/elastic_backups
chmod 755 /var/elastic_backups
```

Для автоматического монтирования при загрузке добавить в `/etc/fstab`:

```
nfs-server:/export/elastic_backups  /var/elastic_backups  nfs  defaults,_netdev  0 0
```

---

## Запуск и проверка кластера

```bash
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
systemctl status elasticsearch

tail -f /var/log/elasticsearch/example-prod.log
```

### Проверка

```bash
curl -X GET "http://10.0.0.11:9200/_cat/nodes?v"
curl -X GET "http://10.0.0.11:9200/_cluster/health?pretty"
curl -X GET "http://10.0.0.11:9200/_cat/shards?v&h=index,shard,prirep,state,node"
```

Ожидаемый результат: 3 ноды, статус `green` (или `yellow` при отсутствии реплик на первом этапе).

После успешной сборки кластера закомментировать `cluster.initial_master_nodes` во всех трёх конфигах и перезапустить ноды по одной, дожидаясь статуса `green` после каждой. Параметр нужен только для первичной инициализации — при последующих рестартах его наличие может вызвать конфликты bootstrap и split-brain.

---

## Nginx-балансировщик

Выполняется на ноде `es-proxy-cluster` (10.0.0.41).

### Установка

```bash
dnf update -y
dnf install -y less nano wget curl openssh-server openssh-clients yum-utils

cat > /etc/yum.repos.d/nginx.repo <<EOF
[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/9/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF

yum-config-manager --enable nginx-mainline
yum install -y nginx

systemctl enable nginx
systemctl start nginx
```

### Конфигурация `/etc/nginx/conf.d/elasticcluster.conf`

Архитектурная особенность: основная нагрузка по записи и поиску направляется на `node2` и `node3`, так как они развёрнуты на более быстрых дисках. Нода `node1` намеренно помечена флагом `backup` — она выступает в роли резервной и принимает клиентский трафик только в случае недоступности основных нод.

```nginx
upstream elasticsearch {
    least_conn;

    server 10.0.0.12:9200 max_fails=3 fail_timeout=30s;
    server 10.0.0.13:9200 max_fails=3 fail_timeout=30s;

    # Резервная нода — включается только при недоступности основных
    server 10.0.0.11:9200 backup max_fails=1 fail_timeout=10s;
}

server {
    listen 9200;
    server_name 10.0.0.41;

    location / {
        proxy_pass http://elasticsearch;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout 60s;
        proxy_send_timeout    600s;
        proxy_read_timeout    600s;

        client_max_body_size 1G;
    }

    location ~ ^/(_cluster|_nodes|_shutdown|_snapshot) {
        allow 10.0.0.0/24;
        deny  all;
    }

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny  all;
    }

    error_log  /var/log/nginx/elastic_error.log;
    access_log /var/log/nginx/elastic_access.log;
}
```

### Применение

```bash
nginx -t
nginx -s reload

curl -X GET "http://10.0.0.41:9200/_cluster/health?pretty"
curl -X GET "http://10.0.0.41:9200/_cat/nodes?v"
```

---

## Снапшоты

Важно: для работы снапшотов в кластере из 3-х нод директория `/var/elastic_backups` обязательно должна быть сетевой файловой системой (NFS, CephFS и т.д.), примонтированной на всех трёх серверах. Локальные папки приведут к ошибке верификации репозитория.

### Регистрация репозитория

Выполняется один раз после старта кластера:

```bash
curl -X PUT "http://10.0.0.41:9200/_snapshot/elastic_backup" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "fs",
    "settings": {
      "location": "/var/elastic_backups",
      "compress": true
    }
  }'

curl -X GET "http://10.0.0.41:9200/_snapshot/elastic_backup?pretty"
```

### Создание снапшота

```bash
curl -X PUT "http://10.0.0.41:9200/_snapshot/elastic_backup/snapshot_$(date +%Y%m%d_%H%M)?wait_for_completion=true&pretty"
```

### Автоматизация (cron)

```
0 2 * * * curl -X PUT "http://10.0.0.41:9200/_snapshot/elastic_backup/snap_$(date +\%Y\%m\%d)?wait_for_completion=false"
```

---

## Создание индексов

Создать каждый индекс **до** reindex. Параметры шардов и реплик подбираются под объём данных.

### Крупные индексы (10+ ГБ)

```bash
curl -X PUT "http://10.0.0.41:9200/general" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 2
    }
  }'
```

### Мелкие индексы (менее 5 ГБ)

```bash
curl -X PUT "http://10.0.0.41:9200/reference" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 2
    }
  }'
```

### Проверка

```bash
curl -X GET "http://10.0.0.41:9200/_cat/indices?v"
```

### Правила выбора шардов и реплик

| Объём индекса | Шардов | Реплик |
|---------------|--------|--------|
| менее 5 ГБ | 1 | 2 |
| 5–10 ГБ | 1–2 | 2 |
| 10–50 ГБ | 3 | 2 |
| 50–100 ГБ | 3–6 | 2 |
| более 100 ГБ | 6+ | 2 |

**Правила:**

- `number_of_shards` должно быть кратно количеству data-нод.
- `number_of_replicas` не больше `number_of_nodes - 1`.
- Целевой размер шарда: 10–50 ГБ.
- Для мелких индексов (менее 5 ГБ) — всегда 1 шард.

---

## Миграция данных через reindex

Данные переносятся из трёх изолированных баз. На каждом исходном Elasticsearch жили свои уникальные индексы, поэтому они последовательно мигрируют в новый общий кластер без риска пересечения документов или перезаписи данных.

### Шаблон команды

```bash
curl -X POST "http://10.0.0.41:9200/_reindex?wait_for_completion=true&refresh=true&pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "remote": { "host": "http://<IP_ИСТОЧНИКА>:9200" },
      "index": "<ИМЯ_ИНДЕКСА>"
    },
    "dest": { "index": "<ИМЯ_ИНДЕКСА>" }
  }'
```

### Пример: general

```bash
curl -X POST "http://10.0.0.41:9200/_reindex?wait_for_completion=true&refresh=true&pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "remote": { "host": "http://10.0.0.153:9200" },
      "index": "general"
    },
    "dest": { "index": "general" }
  }'
```

### Проверка результата

Для каждого индекса сравнить количество документов на источнике и приёмнике:

```bash
curl -X GET "http://10.0.0.153:9200/general/_count?pretty"
curl -X GET "http://10.0.0.41:9200/general/_count?pretty"
```

Числа должны совпадать.

### Общая проверка

```bash
curl -X GET "http://10.0.0.41:9200/_cat/indices?v&s=index"
curl -X GET "http://10.0.0.41:9200/_cluster/health?pretty"
```

---

## Переключение DNS

Выполняется **только после** успешной проверки всех индексов.

1. Убедиться, что все `_count` совпадают, кластер `green`.
2. Уменьшить TTL DNS-записи до 60 секунд (за час до переключения).
3. Переключить DNS-запись `entrypoint.es-proxy.example.com` на `10.0.0.41`.
4. Проверить с клиента:

   ```bash
   dig entrypoint.es-proxy.example.com
   curl -X GET "http://entrypoint.es-proxy.example.com:9200/_cluster/health?pretty"
   ```

5. Не выключать старые ноды 3–7 дней.
6. Вернуть TTL к исходному значению через сутки.

### План отката

Вернуть DNS на старые IP. Благодаря низкому TTL изменения применятся в течение минуты.

---

## Финальная проверка

```bash
curl -X GET "http://10.0.0.41:9200/_cluster/health?pretty"
curl -X GET "http://10.0.0.41:9200/_cat/nodes?v"
curl -X GET "http://10.0.0.41:9200/_cat/shards?v&h=index,shard,prirep,state,node"
curl -X GET "http://10.0.0.41:9200/_cat/indices?v&s=index"
```

**Норма:**

- `status: green`
- `number_of_nodes: 3`
- `unassigned_shards: 0`
- Все шарды в состоянии `STARTED`

---

## Типовые проблемы

### Кластер не собирается

**Симптом:** `_cat/nodes` показывает одну ноду.

**Проверить:**

- Уникальность `node.name` на каждой ноде.
- Открыт ли порт 9300 между нодами.
- Совпадает ли `cluster.name`.
- Совпадает ли `discovery.seed_hosts`.

### max virtual memory areas too low

**Решение:**

```bash
sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
```

### Reindex: no such remote cluster

**Решение:** проверить, что хост-источник указан в `reindex.remote.whitelist` **на всех нодах** кластера-приёмника. Формат: `IP:9200`.

### Кластер в статусе yellow

**Причина:** не все реплики распределены. Обычно при `number_of_replicas` больше или равно `number_of_nodes`.

**Решение:** добавить ноды или уменьшить количество реплик.

### Reindex идёт медленно

- Увеличить `scroll` и `batch_size`.
- Проверить пропускную способность сети.
- Запускать в часы низкой активности.

---

## Чек-лист

- [ ] На всех нодах: `vm.max_map_count=262144`, лимиты, swap отключён
- [ ] Java 17 установлена, `ES_JAVA_HOME` указывает на неё
- [ ] `node.name` уникален на каждой ноде
- [ ] `path.repo` создан, права `elasticsearch:elasticsearch`
- [ ] NFS-шара примонтирована на всех нодах и прописана в `/etc/fstab`
- [ ] Репозиторий снапшотов зарегистрирован
- [ ] Nginx: `least_conn` до `server`, управляющий API закрыт
- [ ] Индексы созданы до reindex с правильными шардами
- [ ] Reindex выполнен с `wait_for_completion=true`
- [ ] `_count` на источнике и приёмнике совпадают
- [ ] `cluster.initial_master_nodes` закомментирован после bootstrap
- [ ] Кластер `green`, `unassigned_shards: 0`
- [ ] DNS переключён, старые ноды не выключены 3–7 дней
- [ ] Настроены бэкапы по cron

---

## Ссылки

- [Elasticsearch 7.17 Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/index.html)
- [Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docs-reindex.html)
- [Snapshot and restore](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/snapshot-restore.html)
- [Официальный пакет Elasticsearch 7.17.27](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.27-x86_64.rpm)
- [Зеркало Aliyun](https://mirrors.aliyun.com/elasticstack/yum/elastic-7.x/7.17.27/elasticsearch-7.17.27-x86_64.rpm)
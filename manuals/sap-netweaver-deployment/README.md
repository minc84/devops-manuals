# Развертывание SAP NetWeaver 7.52 SP04 ABAP Developer Edition на openSUSE Leap 15.3 (KVM)

> «Ну-ка, давай-ка, поглядим, как тут избы делают...» 

Этот проект — подробная хроника ручного деплоя и глубокого DevOps-дебага при установке тяжелой Enterprise-системы на чистую ОС без использования готовых Docker или Vagrant контейнеров. Гайд собран на основе реального опыта решения инфраструктурных конфликтов.

---

##  Архитектура и ресурсы хоста
* **Хост-машина:** Ubuntu 24.04 LTS (20 ядер CPU, 64 GB RAM).
* **Гипервизор:** KVM (Virt-Manager).
* **Ресурсы гостевой ОС (виртуалки):** 4 ядра CPU, 16 GB RAM, жесткий диск VirtIO (`/dev/vda`) на 350 GB.

### Разметка диска в инсталляторе openSUSE:
Чтобы SAP не упирался в искусственные лимиты отдельных каталогов, была выбрана простая и гибкая схема:
* `/dev/vda1` (500 MB) — EFI-раздел для загрузки системы (FAT32).
* `/dev/vda2` (30 GB) — Выделенный раздел подкачки **Swap** (критично для прохождения Prerequisites инсталлятора).
* `/dev/vda3` (остаток пространства) — Основной раздел, отформатированный в **XFS** (стандарт для SUSE Linux Enterprise) и смонтированный целиком под корень **`/`**. Все каталоги SAP (`/sybase`, `/usr/sap`, `/sapmnt`) динамически делят это пространство.

---

## ⚠Критически важный выбор ОС: только openSUSE Leap 15.3!
Не пытайтесь ставить систему на современные дистрибутивы (openSUSE 15.6, Ubuntu 24.04 и т.д.).
**Причина:** В новых дистрибутивах установлена библиотека `glibc 2.39+`. Из-за изменений в формате вывода системных утилит ядра инсталлятор SAP некорректно парсит ответы, путается в цифрах, ошибочно принимает версию СУБД за `15.7.0.000` и аварийно завершает работу.

---

## 🛠️ Предварительная настройка ОС (под пользователем root)

### 1. Сетевое имя и статический IP
Прописываем имя хоста `vhcalnplci` и статический IP в файл `/etc/hosts`:
```text
192.168.122.232    vhcalnplci    vhcalnplci.local
```

### 2. Тюнинг лимитов дескрипторов и оперативной памяти
Добавляем в самый конец файла `/etc/security/limits.conf`:
```ini
npladm          hard    memlock         unlimited
npladm          soft    memlock         unlimited
sybnpl          hard    memlock         unlimited
sybnpl          soft    memlock         unlimited
npladm          hard    nofile          65536
npladm          soft    nofile          65536
sybnpl          hard    nofile          65536
sybnpl          soft    nofile          65536
```

Открываем файл `/etc/sysctl.conf` и выставляем лимит областей памяти согласно **SAP Note 900929**:
```ini
vm.max_map_count=2000000
```
Применяем настройки ядра «на лету» командой: `sysctl -p`.

### 3. Доустановка системных пакетов и UUID-демона
```bash
zypper mr -d 1
zypper ref && zypper update -y
zypper install -y uuidd unrar
systemctl enable --now uuidd
```

---

## Распаковка и запуск установки

Копируем 11 томов архива в `/root/sap` и распаковываем первый том:
```bash
cd /root/sap/
unrar x TD752SP04part01.rar
```
Делаем скрипт исполняемым и запускаем установку:
```bash
chmod +x install.sh
./install.sh
```

### Главный маневр: Подмена протухшей лицензии СУБД Sybase ASE
Встроенный шаблон лицензии в дистрибутиве истек еще в 2021 году, из-за чего инсталлятор падает на шаге смены пароля `sa` с ошибкой `Unable to generate a new password for database login 'sa'`.

**Решение:**
Официальный триал на этот продукт закрыт SAP. Как только инсталлятор в первый раз упадет на шаге паролей `sa` (папка к этому моменту уже будет создана на диске), открываем параллельную вкладку терминала под `root` и перезаписываем файл лицензии актуальным «вечным» ключом до **марта 2027 года**:
```bash
nano /sybase/NPL/SYSAM-2_0/licenses/SYBASE_ASE_TestDrive.lic
```
*(Вставляем текст рабочего ключа до 2027 года)*.

Чтобы инсталлятор при повторном проходе шагов не стёр наши труды, **запираем файл на замок ядра Linux (атрибут immutable)**:
```bash
chmod 755 /sybase/NPL/SYSAM-2_0/licenses/SYBASE_ASE_TestDrive.lic
chown sybnpl:sapsys /sybase/NPL/SYSAM-2_0/licenses/SYBASE_ASE_TestDrive.lic
chattr +i /sybase/NPL/SYSAM-2_0/licenses/SYBASE_ASE_TestDrive.lic
```
Возвращаемся в первое окно и перезапускаем `./install.sh`. Установка успешно завершится.

---

## Управление ландшафтом (под пользователем npladm)
Все операции выполняются строго под пользователем `npladm` (`su - npladm`) с помощью утилиты `sapcontrol`:

###  Корректное выключение (Полный стоп):
```csh
sapcontrol -prot NI_HTTP -nr 00 -function StopSystem ALL
sapcontrol -prot NI_HTTP -nr 01 -function StopSystem ALL
cleanipc 00 remove
cleanipc 01 remove
```
*(После этого пишем `exit` под рута и гасим саму виртуалку командой `poweroff`)*.

###  Корректное включение (Полный старт):
```csh
sapcontrol -prot NI_HTTP -nr 01 -function StartSystem ALL
sapcontrol -prot NI_HTTP -nr 00 -function StartSystem ALL
```

### Проверка статуса процессов (ищем везде GREEN):
```csh
sapcontrol -prot NI_HTTP -host 127.0.0.1 -nr 00 -function GetProcessList
```

---

## Настройка графического интерфейса (SAP GUI) на Ubuntu
Мы используем кроссплатформенный **SAP GUI for Java (PlatinGUI)**, инсталлятор которого уже лежит в дистрибутиве сервера в папке `/root/sap/client/JavaGUI/`.

1. Скачиваем папку инсталлятора с сервера на ноутбук по SCP:
   ```bash
   scp -r root@192.168.122.232:/root/sap/client/JavaGUI ~/Загрузки/
   ```
2. Устанавливаем среду Java на Ubuntu:
   ```bash
   sudo apt update && sudo apt install default-jre default-jdk -y
   ```
3. Запускаем установку клиента на ноутбуке:
   ```bash
   cd ~/Загрузки/JavaGUI
   java -jar PlatinGUI-Linux-X.jar   # подставьте имя вашего .jar файла
   ```
4. Запускаем **SAP Logon**, создаем новое подключение, во вкладке **Advanced** ставим галочку *«Use expert configuration»* и прописываем строку подключения:
   ```text
   conn=/H/192.168.122.232/S/3200
   ```
5. Входим под **Client `000`**, **User `SAP*`**, **Password `Ваш_Мастер_Пароль`**, заходим в транзакцию **`/nSLICENSE`** и накатываем постоянный цифровой ABAP-ключ, сгенерированный на портале [SAP Trial and Developer License Keys](https://sap.com).

---
*Искренне надеюсь, что этот собранный по крупицам мануал сэкономит кому-то кучу часов и окажется полезным для комьюнити!*

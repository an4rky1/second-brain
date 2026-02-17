---
created: 2026-02-16
tags:
  - cheat-sheet
  - linux
  - bash
  - sysadmin
aliases:
  - Linux Cheatsheet
  - Linux Commands Reference
related:
  - Bash-Cheatsheet
  - Docker-Cheatsheet
  - Security-Hardening
---

# Linux — Полная шпаргалка

> [!SUMMARY] Обзор
> Linux — операционная система для серверов, разработки и embedded. Командная строка, файловая система, процессы, сеть, безопасность. 15 лет опыта в одном месте.

---

## 📚 Теория

### Файловая система

```
/
├── bin/          # Бинарники пользователей (ls, cp, mkdir)
├── boot/         # Загрузчик (kernel, initrd)
├── dev/          # Устройства (sda, tty, null)
├── etc/          # Конфигурация системы
├── home/         # Домашние директории пользователей
├── lib/          # Системные библиотеки
├── lib64/        # 64-битные библиотеки
├── media/        # Сменные носители
├── mnt/          # Точки монтирования
├── opt/          # Дополнительное ПО
├── proc/         # Виртуальная ФС (информация о процессах)
├── root/         # Дом root пользователя
├── run/          # Runtime данные
├── sbin/         # Системные бинарники (для root)
├── srv/          # Данные сервисов
├── sys/          # Виртуальная ФС (информация о ядре)
├── tmp/          # Временные файлы
├── usr/          # Пользовательские программы
│   ├── bin/
│   ├── lib/
│   ├── local/
│   └── share/
└── var/          # Переменные данные (логи, кэш)
    ├── log/
    ├── cache/
    └── spool/
```

### Права доступа

```
┌─────────────────────────────────────────────────────┐
│  -rwxrwxr-x  2  user  group  4096  Jan 1  12:00  file  │
│  │││││││││                                          │
│  │││││││└└─── Other (world)                          │
│  │││││└└└───── Group                                 │
│  ││││└──────── Owner                                  │
│  │││└───────── Directory (-) or File (d)             │
│  ││└────────── Special (setuid, setgid, sticky)      │
│  └└─────────── Type                                  │
│                                                      │
│  r = 4, w = 2, x = 1                                 │
│  rwx = 7, rw- = 6, r-x = 5, r-- = 4                  │
└─────────────────────────────────────────────────────┘
```

---

## 🔧 Практические примеры

### Файлы и директории

```bash
# Навигация
pwd                     # Текущая директория
cd /path/to/dir         # Сменить директорию
cd ..                   # Наверх
cd ~                    # Домой
cd -                    # Предыдущая

# Список файлов
ls                      # Список
ls -la                  # Все файлы + детали
ls -lh                  # Человекочитаемые размеры
ls -lt                  # По времени
ls -R                   # Рекурсивно
tree -L 2               # Дерево (2 уровня)

# Создание
mkdir dir               # Директория
mkdir -p a/b/c          # С родительскими
touch file.txt          # Пустой файл
cp file.txt copy.txt    # Копировать
cp -r dir/ dir_copy/    # Копировать рекурсивно

# Перемещение
mv file.txt new.txt     # Переименовать
mv file.txt /path/      # Переместить

# Удаление
rm file.txt             # Файл
rm -r dir/              # Директория
rm -rf dir/             # Принудительно (ОСТОРОЖНО!)

# Просмотр
cat file.txt            # Весь файл
less file.txt           # Постранично
head -n 20 file.txt     # Первые 20 строк
tail -n 20 file.txt     # Последние 20 строк
tail -f file.txt        # Follow (логи)
tail -F file.txt        # Follow с reopen

# Поиск файлов
find /path -name "file.txt"           # По имени
find /path -name "*.log" -mtime +7    # Старше 7 дней
find /path -type f -size +100M        # Больше 100MB
find /path -type f -exec rm {} \;     # С действием
find /path -type d -empty             # Пустые директории

# Размер
du -sh dir/               # Размер директории
du -sh *                  # Все в текущей
du -h --max-depth=1       # Первый уровень
df -h                     # Свободное место
df -i                     # Inodes
```

### Текст и поиск

```bash
# Поиск в файлах
grep "pattern" file.txt           # Поиск
grep -r "pattern" .               # Рекурсивно
grep -i "pattern"                 # Без учёта регистра
grep -v "pattern"                 # Исключить
grep -l "pattern" *.txt           # Только имена
grep -c "pattern"                 # Количество
grep -E "pat1|pat2"               # Regex
grep -A 3 "pattern"               # +3 строки после
grep -B 3 "pattern"               # +3 строки до

# Замена
sed 's/old/new/' file.txt         # Первое в строке
sed 's/old/new/g' file.txt        # Все в строке
sed -i 's/old/new/g' file.txt     # In-place
sed -i.bak 's/old/new/g' file.txt # С бэкапом

# Извлечение
cut -d: -f1 /etc/passwd           # Первое поле
cut -c1-5 file.txt                # Символы 1-5

# Сортировка и уникальность
sort file.txt                     # Сортировать
sort -n file.txt                  # Числовая
sort -r file.txt                  # Обратная
uniq file.txt                     # Уникальные
uniq -c file.txt                  # С количеством
sort | uniq -c | sort -rn         # Топ

# Сравнение
diff file1.txt file2.txt          # Различия
diff -r dir1/ dir2/               # Рекурсивно
comm file1.txt file2.txt          # Общие/разные
```

### Процессы

```bash
# Список процессов
ps aux                      # Все процессы
ps aux | grep nginx         # Поиск процесса
ps -ef --forest             # Дерево процессов
top                         # Интерактивно
htop                        # Улучшенный top
pgrep nginx                 # PID по имени

# Информация о процессе
pidof nginx                 # PID сервиса
pstree -p                   # Дерево с PID
lsof -p PID                 # Открытые файлы
ls -l /proc/PID/fd          # То же через proc

# Управление
kill PID                    # SIGTERM (15)
kill -9 PID                 # SIGKILL (убить)
kill -HUP PID               # Перезагрузить конфиг
killall nginx               # По имени
pkill -f pattern            # По паттерну

# Приоритет
nice -n 10 command          # Запустить с приоритетом
renice 10 -p PID            # Изменить приоритет

# Фоновые задачи
command &                   # В фон
jobs                        # Список задач
fg %1                       # На передний план
bg %1                       # Продолжить в фоне
Ctrl+Z                      # Пауза
disown %1                   # Отвязать от терминала

# Systemd
systemctl status service    # Статус
systemctl start service     # Запустить
systemctl stop service      # Остановить
systemctl restart service   # Перезапустить
systemctl reload service    # Перезагрузить конфиг
systemctl enable service    # Автозапуск
systemctl disable service   # Отключить автозапуск
systemctl list-units        # Все юниты
systemctl list-unit-files   # Все файлы
journalctl -u service       # Логи сервиса
journalctl -f               # Follow логи
journalctl --since "1 hour ago"
```

### Сеть

```bash
# Информация
ip addr                     # IP адреса
ip link                     # Интерфейсы
ip route                    # Таблица маршрутизации
ip neigh                    # ARP таблица
hostname -I                 # Все IP
hostname -f                 # FQDN

# Устаревшие (но полезные)
ifconfig                    # Интерфейсы
netstat -tulpn              # Порты и процессы
route -n                    # Маршруты

# Современные аналоги
ss -tulpn                   # Сокеты и процессы
ss -tuln                    # Listening порты
ss -an | grep :80           # Поиск по порту

# Диагностика
ping google.com             # Проверка связи
ping -c 4 google.com        # 4 пакета
traceroute google.com       # Трассировка
tracepath google.com        # Без root
mtr google.com              # ping + traceroute

# DNS
dig google.com              # DNS запрос
dig google.com ANY          # Все записи
dig +short google.com       # Кратко
nslookup google.com         # Альтернатива
host google.com             # Проще

# HTTP
curl http://example.com     # GET запрос
curl -I http://example.com  # Только заголовки
curl -X POST -d "key=val" http://example.com
curl -H "Authorization: Bearer token" http://example.com
curl -o file.zip http://example.com/file.zip
wget http://example.com/file.zip  # Скачать

# Сетевые утилиты
nc -zv host 22              # Проверка порта
nc -l 8080                  # Слушать порт
telnet host 22              # Telnet (устарело)
ssh user@host               # SSH подключение
ssh -i key.pem user@host    # С ключом
scp file.txt user@host:/path # Копирование
rsync -avz src/ dest/       # Синхронизация

# Firewall (iptables)
iptables -L -n              # Список правил
iptables -L -n -v           # С деталями
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -j DROP
iptables-save > /etc/iptables/rules.v4

# Firewall (ufw - проще)
ufw status                  # Статус
ufw enable                  # Включить
ufw allow 22/tcp            # Разрешить порт
ufw deny 80/tcp             # Запретить
ufw delete allow 22/tcp     # Удалить правило
ufw status numbered         # С номерами
ufw delete 1                # Удалить правило 1
```

### Система

```bash
# Информация о системе
uname -a                    # Ядро и система
cat /etc/os-release         # Дистрибутив
cat /proc/cpuinfo           # CPU
cat /proc/meminfo           # RAM
lscpu                       # CPU детали
free -h                     # Память
lsblk                       # Диски
df -h                       # Место на дисках
uptime                      # Время работы и нагрузка

# Загрузка
top                         # Процессы
htop                        # Улучшенный top
vmstat 1                    # Статистика каждую секунду
iostat -x 1                 # Disk I/O
sar -u 1 3                  # CPU история

# Логи
tail -f /var/log/syslog     # Системный лог
tail -f /var/log/auth.log   # Аутентификация
tail -f /var/log/nginx/error.log
journalctl -xe              # Systemd логи
dmesg | tail                # Ядро сообщения
dmesg -w                    # Follow

# Пользователи
who                         # Кто залогинен
whoami                      # Текущий пользователь
id                          # ID пользователя
last                        # История входов
w                           # Кто и что делает

# Управление пользователями
useradd username            # Создать
usermod -aG sudo username   # Добавить в группу
passwd username             # Сменить пароль
userdel -r username         # Удалить с домашней

# Группы
groups                      # Группы пользователя
groupadd groupname          # Создать группу
usermod -aG groupname user  # Добавить в группу

# Права
chmod 755 file              # Изменить права
chmod +x script.sh          # Исполняемый
chown user:group file       # Владелец
chgrp group file            # Группа

# Sudo
sudo command                # Выполнить как root
sudo -i                     # Root shell
sudo -u user command        # Как другой пользователь
visudo                      # Редактировать sudoers
```

### Архивы и сжатие

```bash
# tar
tar -czvf archive.tar.gz dir/   # Создать gzip
tar -xzvf archive.tar.gz        # Распаковать gzip
tar -cjvf archive.tar.bz2 dir/  # Создать bzip2
tar -xjvf archive.tar.bz2       # Распаковать bzip2
tar -tf archive.tar.gz          # Список файлов
tar -tvf archive.tar.gz         # Список с деталями

# zip/unzip
zip -r archive.zip dir/         # Создать
unzip archive.zip               # Распаковать
unzip -l archive.zip            # Список

# gzip/gunzip
gzip file.txt                   # Сжать
gunzip file.txt.gz              # Распаковать
zcat file.txt.gz                # Просмотр

# Сравнение
diff -u original.txt modified.txt > patch.diff
patch original.txt < patch.diff
```

### Производительность

```bash
# Мониторинг
top                             # Процессы
htop                            # Улучшенный top
iotop                           # Disk I/O по процессам
iftop                           # Network по процессам
nethogs                         # Network по процессам

# Статистика
vmstat 1                        # Виртуальная память
iostat -x 1                     # Disk статистика
sar -u -r -n DEV 1              # Система статистика
dstat                           # Комбайн метрик

# Поиск проблем
# Высокая CPU
top                             # Найти процесс
perf top                        # Профилирование
strace -p PID                   # Системные вызовы

# Высокая память
free -h                         # Общая информация
ps aux --sort=-%mem | head      # Топ по памяти

# Высокая Disk I/O
iotop                           # Найти процесс
lsof +D /path                   # Открытые файлы

# Высокая Network
iftop                           # Трафик
ss -tulnp                       # Сокеты
```

### Безопасность

```bash
# Файлы и права
chmod 600 ~/.ssh/id_rsa         # Приватный ключ
chmod 644 ~/.ssh/id_rsa.pub     # Публичный ключ
chmod 700 ~/.ssh                # SSH директория
find / -perm -4000              # SUID файлы
find / -perm -2000              # SGID файлы

# Пользователи
passwd -l username              # Заблокировать
passwd -u username              # Разблокировать
passwd -d username              # Удалить пароль
chage -l username               # Информация о пароле

# SSH
ssh-keygen -t ed25519           # Создать ключ
ssh-copy-id user@host           # Копировать ключ
ssh -i key.pem user@host        # Подключение с ключом

# Firewall
ufw status verbose
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80,443/tcp
ufw enable

# Аудит
last                            # История входов
lastb                           # Неудачные входы
grep "Failed password" /var/log/auth.log
who                             # Сейчас в системе
w                               # Кто и что делает

# SELinux/AppArmor
getenforce                      # SELinux статус
setenforce 0                    # Временно отключить
aa-status                       # AppArmor статус
```

---

## 🎯 Best Practices

### ✅ Делать

```bash
# 1. Используйте -i для интерактивного удаления
rm -i important.txt

# 2. Проверяйте команды с echo
for file in *.txt; do echo rm "$file"; done

# 3. Используйте rsync для копирования
rsync -avz --progress src/ dest/

# 4. Логи в syslog
logger "My script started"

# 5. Таймауты для команд
timeout 10 long-running-command

# 6. Проверяйте место перед записью
df -h /path

# 7. Используйте screen/tmux для долгих задач
tmux new -s session
# Ctrl+B, D - detach
tmux attach -t session
```

### ❌ Не делать

```bash
# 1. rm -rf / без проверки
rm -rf /var/*  # ❌ Проверьте путь!

# 2. chmod 777
chmod 777 file  # ❌ Используйте минимальные права

# 3. Работать под root
# Используйте sudo

# 4. Игнорировать логи
# Проверяйте /var/log/

# 5. Хардкод паролей
# Используйте переменные окружения или vault
```

---

## 🔗 Связанные заметки

- [[Bash-Cheatsheet]] — Bash скрипты
- [[Docker-Cheatsheet]] — Docker
- [[Security-Hardening]] — Безопасность

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **man pages** — ваша библия (`man ls`)
> 2. **--help** — быстрая справка (`ls --help`)
> 3. **history** — история команд (`history | grep docker`)
> 4. **alias** — сокращения (`alias ll='ls -la'`)
> 5. **tmux/screen** — сессии для долгих задач

> [!INFO] Полезные пакеты
> ```bash
> # Ubuntu/Debian
> apt update && apt upgrade
> apt install htop iotop iftop nethogs
> apt install tmux tree jq
> apt install net-tools dnsutils tcpdump
> apt install rsync unzip p7zip
> 
> # RHEL/CentOS
> yum install epel-release
> yum install htop iotop iftop
> ```

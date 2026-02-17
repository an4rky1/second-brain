---
created: 2026-02-16
tags:
  - cheat-sheet
  - bash
  - shell
  - scripting
aliases:
  - Bash Cheatsheet
  - Shell Scripting Reference
related:
  - Linux-Commands
  - Docker-Cheatsheet
  - CI-CD-Pipeline
---

# Bash — Полная шпаргалка

> [!SUMMARY] Обзор
> Bash — стандартная оболочка Linux/Unix. Незаменим для автоматизации, DevOps, CI/CD пайплайнов и системного администрирования. 15 лет опыта показали: хороший скрипт экономит часы ручной работы.

---

## 📚 Теория

### Переменные

```bash
# Объявление
NAME="John"
AGE=30
readonly PI=3.14  # Константа

# Использование
echo $NAME
echo ${NAME}
echo "${NAME} is ${AGE}"  # Всегда используйте кавычки!

# Special variables
$0      # Имя скрипта
$1..$9  # Позиционные параметры
$#      # Количество аргументов
$@      # Все аргументы (как отдельные слова)
$*      # Все аргументы (как одно слово)
$?      # Код выхода последней команды
$$      # PID текущего процесса
$!      # PID последнего фонового процесса
$_      # Последний аргумент последней команды

# Parameter expansion
${VAR:-default}      # Default если VAR пустой
${VAR:=default}      # Assign default если пустой
${VAR:+value}        # Value если VAR установлен
${VAR:?error}        # Error если VAR пустой
${#VAR}              # Длина строки
${VAR:0:5}           # Substring (позиция, длина)
${VAR/pattern/repl}  # Замена первого совпадения
${VAR//pattern/repl} # Замена всех совпадений
${VAR#prefix}        # Удалить shortest match с начала
${VAR##prefix}       # Удалить longest match с начала
${VAR%suffix}        # Удалить shortest match с конца
${VAR%%suffix}       # Удалить longest match с конца

# Case modification
${VAR^^}  # Uppercase
${VAR,,}  # Lowercase
```

### Массивы

```bash
# Объявление
arr=(one two three)
arr2=([0]="first" [2]="third")

# Доступ
echo ${arr[0]}      # Первый элемент
echo ${arr[@]}      # Все элементы
echo ${#arr[@]}     # Длина массива
echo ${!arr[@]}     # Индексы

# Операции
arr+=(four)         # Добавить
unset arr[1]        # Удалить элемент
slice=("${arr[@]:1:2}")  # Slice

# Ассоциативные массивы (Bash 4+)
declare -A assoc
assoc[name]="John"
assoc[age]=30
echo ${assoc[name]}
```

---

## ⚡ Быстрый старт

```bash
# Shebang
#!/usr/bin/env bash

# Запуск
bash script.sh
./script.sh  # Если executable (chmod +x)

# Отладка
bash -x script.sh      # Показать команды
bash -n script.sh      # Проверка синтаксиса
set -x                 # Включить отладку в скрипте
set +x                 # Выключить

# Strict mode (всегда используйте!)
set -euo pipefail
# -e: exit on error
# -u: error on undefined variable
# -o pipefail: error if any command in pipeline fails

# Trap (обработка сигналов)
trap 'echo "Interrupted"; exit 1' INT
trap 'echo "Error on line $LINENO"' ERR
trap 'rm -f /tmp/tempfile' EXIT
```

---

## 🔧 Практические примеры

### Условия

```bash
# if/else
if [ -f "file.txt" ]; then
    echo "File exists"
elif [ -d "dir" ]; then
    echo "Directory exists"
else
    echo "Neither"
fi

# Test operators
[ -e file ]    # Существует
[ -f file ]    # Файл
[ -d dir ]     # Директория
[ -r file ]    # Читаемый
[ -w file ]    # Записываемый
[ -x file ]    # Исполняемый
[ -s file ]    # Не пустой
[ -L link ]    # Symlink

# String comparison
[ "$a" = "$b" ]     # Равно
[ "$a" != "$b" ]    # Не равно
[ -z "$a" ]         # Пустая строка
[ -n "$a" ]         # Непустая строка

# Numeric comparison
[ $a -eq $b ]  # Равно
[ $a -ne $b ]  # Не равно
[ $a -lt $b ]  # Меньше
[ $a -le $b ]  # Меньше или равно
[ $a -gt $b ]  # Больше
[ $a -ge $b ]  # Больше или равно

# Логические операторы
[ $a -gt 0 ] && [ $a -lt 10 ]   # AND
[ $a -lt 0 ] || [ $a -gt 100]   # OR
! [ $a -eq 0 ]                   # NOT

# [[ ]] расширенный test (Bash 3+)
[[ $name == J* ]]     # Pattern matching
[[ $name =~ ^[A-Z] ]] # Regex
[[ -e $file && -r $file ]]

# case
case $1 in
    start|run)
        echo "Starting..."
        ;;
    stop|quit)
        echo "Stopping..."
        ;;
    restart)
        $0 stop
        $0 start
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```

### Циклы

```bash
# for
for i in 1 2 3; do
    echo $i
done

for i in {1..10}; do
    echo $i
done

for i in {0..100..10}; do  # С шагом
    echo $i
done

for file in *.txt; do
    echo "Processing $file"
done

# C-style for
for ((i=0; i<10; i++)); do
    echo $i
done

# while
while [ $count -lt 10 ]; do
    echo $count
    ((count++))
done

# Чтение файла
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt

# Чтение с process substitution
while IFS= read -r line; do
    echo "$line"
done < <(ls -la)

# until (пока условие ложно)
until [ $count -ge 10 ]; do
    echo $count
    ((count++))
done

# break/continue
for i in {1..10}; do
    [ $i -eq 5 ] && continue  # Пропустить 5
    [ $i -eq 8 ] && break     # Выйти на 8
    echo $i
done
```

### Функции

```bash
# Объявление
greet() {
    local name=$1  # Локальная переменная
    echo "Hello, $name!"
}

# С возвратом значения
add() {
    local a=$1
    local b=$2
    echo $((a + b))  # Вывод как результат
}

result=$(add 5 3)
echo $result  # 8

# С кодом выхода
check_file() {
    [ -f "$1" ]
    # Возвращает 0 если true, 1 если false
}

if check_file "file.txt"; then
    echo "Exists"
fi

# Рекурсия
factorial() {
    local n=$1
    [ $n -le 1 ] && echo 1 && return
    echo $((n * $(factorial $((n - 1)))))
}

# Функции с cleanup
with_temp_dir() {
    local tmpdir
    tmpdir=$(mktemp -d)
    trap 'rm -rf "$tmpdir"' EXIT
    
    # Работа с $tmpdir
    # ...
    
    trap - EXIT  # Снять trap
}
```

### Работа с файлами

```bash
# Чтение файла
content=$(cat file.txt)
content=$(<file.txt)  # Быстрее

# Построчное чтение
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Запись в файл
echo "Hello" > file.txt    # Перезаписать
echo "World" >> file.txt   # Дописать

# Здесь документы
cat <<EOF > file.txt
Line 1
Line 2
Variable: $NAME
EOF

# Здесь строка
read -r -d '' content <<'EOF'
Multi
line
content
EOF

# Разделение вывода
command > output.txt 2> error.txt
command > all.txt 2>&1  # Всё в один файл
command &> all.txt      # То же самое (Bash 4+)

# Проверка существования
[ -e file ] && echo "Exists" || echo "Not found"

# Создание с проверкой
mkdir -p /path/to/dir  # -p создаёт родительские

# Временные файлы
tmpfile=$(mktemp)
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT
```

### Работа со строками

```bash
# Длина
str="Hello World"
echo ${#str}  # 11

# Substring
echo ${str:0:5}   # Hello
echo ${str:6}     # World

# Замена
echo ${str/World/Universe}  # Hello Universe
echo ${str//l/L}            # HeLLo WorLd

# Удаление
echo ${str#Hello }   # World (с начала)
echo ${str% World}   # Hello (с конца)

# Trim whitespace
trimmed=$(echo "$str" | xargs)

# Split
IFS=',' read -ra parts <<< "a,b,c"
echo ${parts[0]}  # a

# Join
arr=(a b c)
joined=$(IFS=,; echo "${arr[*]}")  # a,b,c

# Case conversion
lower=${str,,}    # hello world
upper=${str^^}    # HELLO WORLD
```

### Арифметика

```bash
# Целочисленная
a=$((5 + 3))
b=$((10 - 4))
c=$((2 * 6))
d=$((15 / 3))
e=$((17 % 5))
f=$((2 ** 3))  # Степень

# Инкремент
((counter++))
((counter+=5))

# Сравнение
if (( count > 10 )); then
    echo "More than 10"
fi

# Float (через bc)
result=$(echo "scale=2; 10 / 3" | bc)  # 3.33
```

### Процессы

```bash
# Фоновый процесс
sleep 10 &
pid=$!

# Ожидание
wait $pid
wait  # Ждать все фоновые процессы

# Timeout
timeout 5 command  # Убить через 5 секунд

# Параллельное выполнение
command1 &
command2 &
wait  # Ждать завершения всех

# Ограничение параллелизма
for i in {1..100}; do
    ((i % 10 == 0)) && wait  # Ждать каждые 10
    process $i &
done
wait

# Pipeline
cat file.txt | grep pattern | sort | uniq -c

# Process substitution
diff <(ls dir1) <(ls dir2)

# Named pipes (FIFO)
mkfifo /tmp/mypipe
cat /tmp/mypipe &
echo "data" > /tmp/mypipe
```

### Скрипты production-ready

```bash
#!/usr/bin/env bash
set -euo pipefail

# Constants
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)

# Logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

error() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: $*" >&2
    exit 1
}

# Usage
usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [OPTIONS]

Options:
    -h, --help      Show this help
    -v, --verbose   Enable verbose mode
    -o, --output    Output file

Examples:
    $SCRIPT_NAME -o output.txt
    $SCRIPT_NAME --verbose
EOF
}

# Parse arguments
VERBOSE=false
OUTPUT=""

while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            usage
            exit 0
            ;;
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -o|--output)
            OUTPUT="$2"
            shift 2
            ;;
        -*)
            error "Unknown option: $1"
            ;;
        *)
            break
            ;;
    esac
done

# Main
main() {
    log "Starting $SCRIPT_NAME"
    
    if $VERBOSE; then
        log "Verbose mode enabled"
    fi
    
    if [[ -n "$OUTPUT" ]]; then
        log "Output file: $OUTPUT"
    fi
    
    # Your logic here
    
    log "Completed successfully"
}

# Cleanup
cleanup() {
    local exit_code=$?
    # Cleanup logic
    exit $exit_code
}
trap cleanup EXIT

# Run
main "$@"
```

---

## 🎯 Best Practices

### ✅ Делать

```bash
# 1. Всегда strict mode
set -euo pipefail

# 2. Всегда кавычки
echo "$VAR"      # ✅
echo $VAR        # ❌ (может сломаться на пробелах)

# 3. Проверка команд
if command -v docker &>/dev/null; then
    echo "Docker installed"
fi

# 4. Использование функций
build() {
    # ...
}

# 5. Логирование
log() { echo "[$(date)] $*"; }
log "Starting..."

# 6. Временные файлы
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT
```

### ❌ Не делать

```bash
# 1. Не использовать eval
eval $user_input  # ❌ Опасно!

# 2. Не парсить ls
ls | grep pattern  # ❌
find . -name "pattern"  # ✅

# 3. Не игнорировать ошибки
command || true  # ❌ (скрывает ошибки)

# 4. Не использовать backticks
result=`command`  # ❌
result=$(command)  # ✅

# 5. Не забывать кавычки
for f in *.txt; do
    cat $f  # ❌ (сломается на пробелах)
    cat "$f"  # ✅
done
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `command not found` | PATH не настроен | Проверьте `echo $PATH` |
| `permission denied` | Нет прав на выполнение | `chmod +x script.sh` |
| `unbound variable` | Strict mode + нет переменной | Используйте `${VAR:-default}` |
| `syntax error near unexpected` | Проблема с кавычками/скобками | Проверьте синтаксис |
| `pipefail` error | Ошибка в pipeline | Проверьте каждую команду |

---

## 🔗 Связанные заметки

- [[Linux-Commands]] — Команды Linux
- [[Docker-Cheatsheet]] — Docker команды
- [[CI-CD-Pipeline]] — CI/CD скрипты

---

## 📝 Заметки

> [!TIP] Совет от 15 лет опыта
> 
> 1. **set -euo pipefail** — всегда в начале скрипта
> 2. **Кавычки везде** — предотвращает 90% багов
> 3. **Функции для всего** — улучшает читаемость
> 4. **trap для cleanup** — всегда чистите за собой
> 5. **Тестируйте скрипты** — shellcheck ловит ошибки

> [!INFO] Полезные инструменты
> ```bash
> # Shellcheck (линтер)
> shellcheck script.sh
> 
> # Shfmt (форматтер)
> shfmt -w script.sh
> 
> # Bashate (стиль)
> bashate script.sh
> 
> # Проверка синтаксиса
> bash -n script.sh
> 
> # Отладка
> bash -x script.sh
> ```

> [!EXAMPLE] One-liners
> ```bash
> # Найти и удалить файлы старше 7 дней
> find /path -type f -mtime +7 -delete
> 
> # Посчитать строки в файлах
> wc -l *.txt
> 
> # Заменить текст в файлах
> sed -i 's/old/new/g' *.txt
> 
> # Параллельное выполнение
> parallel -j 4 process {} ::: *.txt
> 
> # JSON из командной строки
> jq '.name' file.json
> ```

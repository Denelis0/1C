# 1C

#!/bin/bash

# ... твои переменные (RAC, CLUSTER и т.д.) ...
LOGFILE="/var/log/1c/AAYK.log"
CUR_USER="${SUDO_USER:-$USER}"

# Функция для логирования
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$LOGFILE"
}

log_message "Попытка блокировки базы - Пользователь: $CUR_USER"

# 1. Пробуем обновить параметры базы
# '2>>' перенаправляет только сообщения об ошибках в файл лога
$RAC infobase --cluster=$CLUSTER --cluster-user=$CLUSTER_USER --cluster-pwd=$CLUSTER_PWD update --infobase=$BASE --infobase-user="user" --infobase-pwd="pwd" --sessions-deny=on --scheduled-jobs-deny=on --permission-code=$code 2>> "$LOGFILE"

if [ $? -ne 0 ]; then
    log_message "ОШИБКА: Не удалось установить блокировку на базу (см. технический текст выше)"
else
    log_message "УСПЕХ: Параметры базы обновлены"
fi

# 2. Пробуем получить сессии
SESSIONS=$($RAC session --cluster=$CLUSTER --cluster-user=$CLUSTER_USER --cluster-pwd=$CLUSTER_PWD list --infobase=$BASE 2>> "$LOGFILE" | grep 'session' | awk '{print $3}')

if [ -z "$SESSIONS" ] && [ $? -ne 0 ]; then
    log_message "ОШИБКА: Не удалось получить список сессий"
fi

# 3. Завершение сессий
for session in $SESSIONS
do
    $RAC session --cluster=$CLUSTER --cluster-user=$CLUSTER_USER --cluster-pwd=$CLUSTER_PWD terminate --session=$session 2>> "$LOGFILE"
    if [ $? -eq 0 ]; then
        log_message "Сессия $session успешно завершена"
    else
        log_message "ОШИБКА: Не удалось завершить сессию $session"
    fi
done

read -p "Нажмите Enter для выхода..."

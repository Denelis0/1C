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


<?xml version="1.0"?>
<config xmlns="http://v8.1c.ru/v8/tech-log">

  <dump create="true" location="C:\LOGS\Dumps\" prntscrn="false" type="3" externaldump="1"/>

  <log history="168" location="C:\LOGS\SRV">
    <event>
      <eq property="Name" value="ADMIN"/>
    </event>
    <event>
      <eq property="Name" value="HASP"/>
    </event>
    <event>
      <eq property="Name" value="CONN"/>
    </event>
    <event>
      <eq property="Name" value="CLSTR"/>
    </event>
    <event>
      <eq property="Name" value="EXCP"/>
    </event>
    <event>
      <eq property="Name" value="EXCPCNTX"/>
    </event>
    <event>
      <eq property="Name" value="ATTN"/>
    </event>
    <event>
      <eq property="Name" value="PROC"/>
    </event>
    <event>
      <eq property="Name" value="TDEADLOCK"/>
    </event>
    <event>
      <eq property="Name" value="TTIMEOUT"/>
    </event>
    <event>
      <eq property="Name" value="TLOCK"/>
      <ge value="3000000" property="Durationus"/>
    </event>
    <property name="all"/>
  </log>

  <log history="48" location="C:\LOGS\LONG">
    <event>
      <ne property="Name" value=""/>
      <gt property="Durationus" value="100000000"/>
    </event>
    <property name="all"/>
  </log>

</config>

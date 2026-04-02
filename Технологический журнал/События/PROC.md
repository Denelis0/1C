


Событие **`PROC` (Process)** — это твой главный системный «черный ящик». Если события `EXCP` показывают ошибки пользователей (например, «Деление на ноль»), то `PROC` показывает **жизнь и смерть самих программ (бинарников 1С) на уровне операционной системы Linux**.

Это событие **ОБЯЗАТЕЛЬНО** должно быть включено на рабочем сервере 24/7 (оно весит копейки, но спасает при разборе серьезных аварий).

---

### 1. Суть события `PROC` (О чем оно пишет?)

У этого события всего два главных состояния (они будут написаны в колонке **`Txt`** в логе):
1.  **`started`** — процесс успешно запущен (когда ты сделал `systemctl start`).
2.  **`Process terminated`** — процесс штатно завершен (например, сработал настроенный рециклинг, или сделан `systemctl stop`).

---

### 2. Главные значения для анализа

Когда ты откроешь лог `PROC`, ищи глазами эти параметры:
В `ragent_PID`:
*   **`Process`** 
    *   *Что это:* Имя убитого процесса (`rphost`, `rmngr`, `ragent`).
    *   *Зачем:* Чтобы понять масштаб трагедии. Если упал `rphost` — выкинуло часть пользователей. Если упал `rmngr` — встал весь кластер, никто не может получить лицензию.
*   **`Updated cached ping params for output cluster connection` - выключение `rphost` плавно. Кластер обновил ифнормацию о своих сетевых портах (удалил умирающий процесс из "адресной книги").
*   **`Supervision time expired`** - время проверки истекло. Процесс не отвечает `rmngr` и его убивают. `pid` - номер процесса.

В `rphost_PID`:
*   **`Process terminated. Any clients finished with error`** - процесс убит жестко, выскочит ошибка у пользователей.
*   **`Run process` - далее запускает минимум два процесса (`rmngr` и `rphost`). 
*   **`Process become disable` - у процесса в столбце Активный статус "нет".
*   **`Err=0,Txt=1C:Enterprise 8.3 (x86-64) (8.3.27.1859) Working Process (debug) terminated.` - Процесс запущен в режиме отладки, завершился процесс штатно (ошибка=0, если бы что-то страшное произошло, то был бы код ниже). После начнется отслеживание разрыва соединений (`Updated cached ping params for output cluster connection` - обновлением параметров проверки соединений в кластере). Включит на `rphost` модуль debug - `1C:Enterprise 8.3 (x86-64) (8.3.27.1859) Search server (debug) started`. 


---

### 3. Случаи

1. Если умер `rmngr`, то пишется в `ragent_PID` строчка `Supervision time expired` и pid процессов rphosts. После пишется `Process finished` и pid умирающего процесса. После этого начинается замена процесса новым - `Run process` (для `rmngr` и как минимум для одного `prhost`). После этого `ragent` отслеживает разрыв соединений (`Updated cached ping params for output cluster connection`) и после запускает режим отладки, то есть определенный компнонет (`1C:Enterprise 8.3 (x86-64) (8.3.27.1859) Search server (debug) started`).

2. Если умер `rmngr`, то пишется в `ragent_PID` строчки:
   * `Err=0,Txt=1C:Enterprise 8.3 (x86-64) (8.3.27.1859) Server Agent (debug) started` - создался новый `ragent`.
   * `Txt=Run process` - запустил процесс (`rmngr`).
   * `Func=authenticateStarter` - авторизация.
   * `Updated cached ping params for output cluster connection` - начинает отслеживать соединения с `rmgr` (первая попытка).
   * `Run process as job. Prog=/opt/1cv8/x86_64/8.3.27.1859/jre/bin/java, Command=(-Dconsole=none -Dcfg=/opt/1cv8/x86_64/8.3.27.1859/sm-searcher/e1c-fts-1.*.cfg -Dhost=0.0.0.0 -Dport=1676 -Droot=/var/lib/1c_cluster2/reg_1641 - Duser.dir=/opt/1cv8/x86_64/8.3.27.1859/sm-searcher -Dapp=....` - 
   * `Txt=Run process` - запустил процесс (`rphost`).
   * `Txt=Run process as job`
   * `Updated cached ping params for output cluster connection` - начинает отслеживать соединения с драгими серверами (это `ragent`) и `rphost` (это `rmngr`).

   В `rmngr_PID` строчки:
   * `Err=0,Txt=1C:Enterprise 8.3 (x86-64) (8.3.27.1859) Cluster Manager (debug) started` - создался `rmngr`.
   * `local cluster registry loaded, cluster id=...` -
   * `Updated cached ping params for output cluster connection` - 

   В `rphost_PID` строчки:
   * `Err=0,Txt=1C:Enterprise 8.3 (x86-64) (8.3.27.1859) Working Process (debug) started` - создался `rphost`.
   * `Updated cached ping params for output cluster connection` -    


---


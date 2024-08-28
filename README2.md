# Web application firewall #
Данный пакет представляет собой простую реализацию WAF для веб-сервера nginx

## Установка ##
**Шаг 1**. Скачать репозиторий (от рута):
```bash
mkdir -p /etc/nginx/conf.d/waf && cd /etc/nginx/conf.d/waf && \
git clone https://Web-format@github.com/Web-format/waf-i.git .
```

<b style="color:red;">**Важно!**</b> Если у вас установлен `etckeeper`, добавьте git-репозиторий в исключения во избежание конфликтов:
```bash
test -f /etc/.hgignore && echo "nginx/conf.d/waf/.git/" >> /etc/.hgignore
```


**Шаг 2**. Создать файл `$document_root/.waf/.disabled` и заранее отключить firewall:

В корневом каталоге каждого интересующего нас сайта от имени пользователя `root` нужно создать подпапку `.waf`, защищённую от редактирования непривилигированным пользователем (существование файла в этой папке будет отключать firewall):
```bash
# от имени пользователя root
# для каждого сайта
wafdir=/home/bitrix/www/.waf && \
mkdir $wafdir && chmod 751 $wafdir && touch $wafdir/.disabled
```
**Важно!** Даже если мы не планируем отключать firewall для некоторого сайта, папку обязательно нужно создать (с защитой от записи) - иначе потенциальный злонамеренный php-код сможет создать этот файл и отключить firewall без нашего ведома.

**Шаг 3**. Внести правки в ваш nginx.conf

<u>**`Во-первых`**</u>, нужно указать map_hash_bucket_size. В секцию `http` вашего `/etc/nginx/nginx.conf` прописать директиву:
```nginx
map_hash_bucket_size 512;
```
(это нужно для поддержки длинных выражений в map-конструкциях, которые используются в конфигах этого репозитория)


***

<u>**`Во-вторых`**</u>, скорректировать формат лог-файлов

В формате лог-файлов, задаваемых в директиве `log_format` вашего nginx.conf, нужно дописать:

`[${waf_warnings}reject reason: $waf_is_suspicious])`. 

А также добавить информацию о `$host`, `$http_accept`. Например:
```nginx
# вместо $time_local настоятельно рекомендуем использовать $time_iso8601 для удобства сортировки строк по времени в будущем
log_format main	'$remote_addr - $remote_user [$time_iso8601 - $upstream_response_time] '
    '$status $host "$request" $body_bytes_sent '
    '"ref:$http_referer" "agent:$http_user_agent" "accept:$http_accept" "$http_x_forwarded_for" '
    '[${waf_warnings}reject reason: $waf_is_suspicious]';
```
***
<u>**`В-третьих`**</u>, подключить правила блокировки
```nginx
include conf.d/waf/init[.]conf;
```
Эти правила следует подключать максимально рано - например, после объявления `log_format` или непосредственно перед строкой `include bx/maps/*.conf;`

<br>

**Шаг 4**. [опционально] Подключить специфичные для сайта исключения

В папке `./exclude/opt` располагаются файлы, специфичные для того или иного типа веб-ресурса, движка сайта. Подключите исключения через создание символьной ссылки на соответствующий файл:
```bash
cd /etc/nginx/conf.d/waf/exclude && \
ln -s ./opt/trusted_urls_{имя движка}.conf ./trusted_urls_{имя движка}.enabled.conf
```

<br>

**Шаг 5**. Включить код, блокирующий запросы

В секцию `server` ваших конфигов нужно подключить файл
```nginx
include conf.d/waf/take_actions[.]conf;
```

В nginx веб-окружения битрикс уместо подключить эту строку в файле `/etc/nginx/bx/conf/bitrix_general.conf` перед строкой `include bx/conf/bitrix_block.conf;`, ответственной за базовые правила блокировки самого веб-окружения.

**Важно!** `take_actions.conf` используют переменную $document_root для определения того, включен ли WAF для данного сайта. Поэтому подключать этот конфиг нужно после директивы `root`:
```nginx
root $root_path;
```


**Шаг 6**. Отладка и релоад

Для проверки синтаксиса выполните команду:
```bash
nginx -t
```
Если всё ок, то ещё раз проверьте в `document_root` интересующего вас сайта файл наличие файла `.waf/.disabled` - это отключит firewall (он будет работать пассивно - писать логи, но не будет блокировать запросы).
После этого:
```bash
systemctl reload nginx
```

**Шаг 7**. Отладка и релоад
Сделайте коммит для etckeeper:
```bash
etckeeper commit "WAF-I installed"
```

<br>
<br>

# Калибровка #
Исходим из того, что вначале межсетевой экран работает в пассивном режиме (т.е. существует файл `.waf/.disabled` от корня проекта).

В первую очередь нам нужно адаптировать сигнатуры, чтобы исключить ложноположительные срабатывания.

Для этого:

1. Подготавливаем набор файлов `access_log`'а, по которым будет производиться поиск. Для этого можно воспользоваться скриптом unpack_logs.sh из [репозитория cli-gate](https://bitbucket.org/wfrepo/cli-gate/src/master/utils/unpack_logs.sh). Например:

    ```bash
    # при многократных повторениях лучше прописать себе алиас:
    mcedit ~/.bashrc
    alias unpack_logs=/home/bitrix/shared/cli/utils/unpack_logs.sh
    . ~/.bashrc
    ```

    ```bash
    # чтобы ограничить набор файлов, используйте параметр "--wildcard"
    # чтобы указать папку, отличную от "/var/log/nginx", используйте параметр "--from"
    # чтобы запустить скрипт в холостом режиме (только отображение списка файлов), используйте параметр "--list-only"
    mkdir -p /root/logs/`date -I` && cd /root/logs/`date -I` && pwd
    unpack_logs --to="/root/logs/`date -I`" --since="`date -I -d '1 days ago'`"
    #или с момента последней проверки:
    wafLastCheck=`cat /root/last-waf-i-check.log` && \
    unpack_logs --to="/root/logs/`date -I`" --since="$wafLastCheck"
    ```



<br>

2. Формируем изначальный список запросов, которые были заблокированы (с интересующей нас даты):
    ```bash
    # если калибруем в I раз - записываем интересующую нас дату, с которой начинаем отбор:
    echo "`date -I -d '1 days ago'`T00:00:00" > /root/last-waf-i-check.log

    # если по дате фильтровать не нужно - объявляем "from" в awk пустым: awk -v from=""
    wafLastCheck=`cat /root/last-waf-i-check.log` && \
    cat *access* | awk -F'\\[|\\+' -v from="$wafLastCheck" \
    '/reason:((\s|\.)[0-9]+){3}/ && !/reason:\s0\.0\.0/ && $2 >= from' > analyse.log
    #cat *access* | grep -iP 'reason:\s\d+\.\d+\.\d+' | \
    #grep -viP 'reason:\s0\.0\.0' | awk 'substr($4, 2, 19) >= "'$wafLastCheck'"' > analyse.log

    
    ```

<br>

3. Формируем список ботов (отправляют вредоносные запросы, поэтому не надо тратить время на анализ их запросов)
    ```bash
    # выводим частотный список IP-адресов, с которых были 4xx-запросы
    # в `cnt` указываем минимальное кол-во интересующих нас запросов.
    awk -v cnt=3 '$7 ~ /^4[0-9][0-9]$/ {_[$1]++} END {for (ip in _) if (_[ip] >= cnt) print ip, _[ip]}' analyse.log | sort -k2,2nr > bots.log

    # формируем отдельный файл со списком N заблокированных запросов с каждого из этих IP среди файлов *access*:
    awk '{print $1}' bots.log | while read ip; do echo $ip":"; cat *access* | grep $ip | grep -viP ': 0(\.0\.0|$)' | head -10 | sed 's/^/\t\t/' | cut -c -300; echo -e "\n\n"; done > bot_hits.log
    
    awk '{print $1}' bots.log | while read ip; do echo $ip":"; cat *access* | awk -F'"' '/^'$ip'/ && !/:\s0(\.0\.0|$)/ { print "\t" $2 }' | head -10 | cut -c -300; echo -e "\n\n"; done > bot_hits2.log

    # просматриваем получившийся файл глазками, выписываем из него IP: "123.123.123.123|124.124.124.124" и т.д.
    # получившийся список прогоняем через sed для экранирования точек:
    echo '45.146.167.56|45.95.147.236|5.252.118.211' | sed 's/\./\\\./g'

    # на ранних этапах калибровки можно не заморачиваться и формировать полный список без ручного контроля:
    awk -v cnt=3 '$7 ~ /^4[0-9][0-9]$/ {_[$1]++} END {for (ip in _) if (_[ip] >= cnt) print ip}' analyse.log | tr '\n' '|' | sed 's/\./\\\./g'

    cnt=10 && cat analyse.log | grep -iP '\] (400|403|404) ' | grep -viP '(\bnull\b)' | awk '{print $1}' | sort | uniq -c | awk '$1 >= '$cnt | sort -nr | awk '{print $2}' | tr '\n' '|' | sed 's/\./\\\./g'
    ```

    **Лайфхак:** в сомнительных ситуациях можно пробить IP по базе вроде [https://www.abuseipdb.com](https://www.abuseipdb.com)

<br>

4. Составляем сокращённый файл (без ошибочных статусов из ботов)
    ```bash
    # в $cnt указываем минимальное кол-во запросов, сделанных ботом. Чем больше cnt, тем меньше ботом мы отловим, но тем больше точность их отлова
    cat analyse.log | grep -viP '\]( 400 | 403 | 404 )' | grep -vP '(...список IP из пред. пункта)...' > analyse2.log
    ```

<br>

5. Составляем частотную таблицу кодов блокировок
    ```bash
    cat analyse2.log | grep -iPo 'reason\:\s\d+\.\d+\.\d+' | awk '{print $2}' | tr '.' '\n' | grep -vP '^0$' | sort | uniq -c | sort -nr
    ```

    Получаем что-то вроде:
    ```ini
    ; Кол-во блокировок => код ошибки
    289 7
    243 78
    212 66
    14  30
    6   33
    5   35
    4   139
    ```

<br>

6. Осуществляем ручной перебор таблицы из пред. пункта
    ```bash
    # смотрим по две шт.
    sigId=7 && cat analyse2.log | grep -iP "reason:.*\b$sigId\b" | head -2

    # если нужно. добавляем исключения:
    sigId=7 && cat analyse2.log | grep -iP "reason:.*\b$sigId\b" | grep -viP '(fbclid|etext)' | head -2
    ```

    ...действуем так для каждого правила из таблицы выше, пока для каждого правила не будет выводиться пустой результат.

<br>

7. Проверяем, с каких IP и сколько запросов делается к чувствительным скриптам обмена (чтобы разрешить их) 
    ```bash
    cat *access* | grep -iP "\/bitrix\/admin\/1c_" | awk '{print $1}' | sort | uniq -c | sort -nr
    ```
    Каждый из этих IP сначала проверяем на предмет вредоносных запросов (чтобы случайно не включить в исключения хакерский IP).
    Учитываем, что помимо 1С-ки к скрипту 1c_exchange может обращаться и облако Битрикс, если настроен обмен заказами: `/crm/configs/external_sale/`. Хиты от него выглядят примерно так
    ```nginx
    # 213.219.215.186 - admin [2023-07-03T12:01:39+03:00 - 0.015] 200 "GET /bitrix/admin/1c_exchange.php?type=crm&mode=init&version=2.09&sessid=454d55adec09b0514952cc637164d041 HTTP/1.1" 77 "-" "BitrixCRM client" "-" [reject reason: 0.0.0]
    ```

    Эти доверенные IP добавляем в файл ./custom/trusted_ips.conf.

8. По окончании этапа калибровки - сделать релоад nginx и записать текущую метку времени:
    ```bash
    nginx -t && systemctl reload nginx
    # tee одновременно (для удобства) выводит дату в терминал и записываем в указанный файл
    date +"%Y-%m-%dT%H:%M:%S" | tee /root/last-waf-i-check.log
    ```
    Относительно этой метки мы будем проводить следующий этап калибровки в будущем.

### SELINUX

```
1. Запустить nginx на нестандартном порту 3-мя разными способами:
   * переключатели setsebool;
   * добавление нестандартного порта в имеющийся тип;
   * формирование и установка модуля SELinux.
К сдаче:
   * README с описанием каждого решения (скриншоты и демонстрация приветствуются).
2. Обеспечить работоспособность приложения при включенном selinux.
   * развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
   * выяснить причину неработоспособности механизма обновления зоны (см. README);
   * предложить решение (или решения) для данной проблемы;
   * выбрать одно из решений для реализации, предварительно обосновав выбор;
   * реализовать выбранное решение и продемонстрировать его работоспособность.
К сдаче:
   * README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
   * исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.
```

## Запуск nginx на нестандартном порту 3-мя разными способами

Итак, для демонстрации имеется готовый [Vagrantfile](Vagrantfile). Запускаем ВМ: `vagrant up`.

[Лог](vagrant_up.log) запуска ВМ (вывод сохранен утилитой script). Выполним вход: `vagrant ssh`, выполним вход `sudo -i`.

Проверим состояние сервиса `nginx`: `systemctl status nginx`

```bash
[vagrant@selinux ~]$ sudo -i
[root@selinux ~]# systemctl status nginx.service 
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Mon 2022-12-26 17:29:50 UTC; 28min ago
  Process: 2830 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2829 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Dec 26 17:29:50 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Dec 26 17:29:50 selinux nginx[2830]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Dec 26 17:29:50 selinux nginx[2830]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
Dec 26 17:29:50 selinux nginx[2830]: nginx: configuration file /etc/nginx/nginx.conf test failed
Dec 26 17:29:50 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Dec 26 17:29:50 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Dec 26 17:29:50 selinux systemd[1]: Unit nginx.service entered failed state.
Dec 26 17:29:50 selinux systemd[1]: nginx.service failed.
```

Согласно информации в логе, конфигурация сервиса `nginx` вызвала чрезвычайное событие (функция `bind()` не смогла получить доступ к порту `4881`). Данная ошибка возникает из-за того, что `SELinux` блокирует запуск сервиса `nginx` на нестандартном порту. Выполним проверку конфигурации `nginx` и режим работы `SELinux`:

```bash
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@selinux ~]# getenforce 
Enforcing
```

Также, можно воспользоваться инструментом `sealert`, он даст больше информации об ошибке и предложит несколько вариантов решений, которые будут рассмотрены в данной документации.

##### sealert

`sealert` — это компонент пользовательского интерфейса (либо GUI, либо командная строка) для системы `setroubleshoot`. `setroubleshoot` используется для диагностики отказов `SELinux` и пытается предоставить удобные объяснения отказа `SELinux` (например, `AVC`) и рекомендации о том, как можно настроить систему, чтобы предотвратить отказ в будущем. В стандартной конфигурации `setroubleshoot` состоит из двух компонентов: `setroubleshootd` и `sealert`. Двумя наиболее полезными параметрами командной строки являются `-l` для «поиска» идентификатора оповещения и `-a` для «анализа» файла журнала. Когда `setroubleshootd` генерирует новое оповещение, он присваивает ему локальный идентификатор и записывает его в виде сообщения системного журнала. Затем можно использовать параметр поиска `-l` для извлечения предупреждения из базы данных предупреждений `setroubleshootd` и записи его в стандартный вывод. Это наиболее полезно, когда `setroubleshootd` запускается в `headless` системе без средства оповещения рабочего стола с графическим интерфейсом. Параметр `-a` (analysis) эквивалентен команде «Scan Logfile» в браузере. Файл журнала сканируется на наличие сообщений аудита, выполняется анализ, генерируются предупреждения, а затем записывается в стандартный вывод.

`setroubleshootd` — это системный демон, который запускается с привилегиями `root` и прослушивает события аудита, исходящие от ядра, связанные с `SELinux`. Демон аудита должен быть запущен. Демон аудита отправляет сообщение `dbus` демону `setroubleshootd`, когда система получает отказ `SELinux AVC`. Затем демон `setroubleshootd` запускает серию подключаемых модулей анализа, которые проверяют данные аудита, относящиеся к `AVC`. Он записывает результаты анализа и сообщает всем клиентам, которые присоединились к демону `setroubleshootd`, о появлении нового предупреждения.

Выполним установку `setroubleshootd`: `yum install setroubleshoot-server -y`

Далее выполним команду `sealert -a /var/log/audit/audit.log` чтобы получить анализ лога `audit`:

```shell
[root@selinux ~]# sealert -a /var/log/audit/audit.log 
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux запрещает /usr/sbin/nginx доступ name_bind к tcp_socket port 4881.

*****  Модуль bind_ports предлагает (точность 92.2)  *************************

Если вы хотите разрешить /usr/sbin/nginx для привязки к сетевому порту $PORT_ЧИСЛО
То you need to modify the port type.
Сделать
# semanage port -a -t PORT_TYPE -p tcp 4881
    где PORT_TYPE может принимать значения: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Модуль catchall_boolean предлагает (точность 7.83)  *******************

Если хотите allow nis to enabled
То вы должны сообщить SELinux об этом, включив переключатель «nis_enabled».

Сделать
setsebool -P nis_enabled 1

*****  Модуль catchall предлагает (точность 1.41)  ***************************

Если вы считаете, что nginx должно быть разрешено name_bind доступ к port 4881 tcp_socket по умолчанию.
То рекомендуется создать отчет об ошибке.
Чтобы разрешить доступ, можно создать локальный модуль политики.
Сделать
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp


Дополнительные сведения:
Исходный контекст             system_u:system_r:httpd_t:s0
Целевой контекст              system_u:object_r:unreserved_port_t:s0
Целевые объекты               port 4881 [ tcp_socket ]
Источник                      nginx
Путь к источнику              /usr/sbin/nginx
Порт                          4881
Узел                          <Unknown>
Исходные пакеты RPM           nginx-1.20.1-10.el7.x86_64
Целевые пакеты RPM            
Пакет регламента              selinux-policy-3.13.1-268.el7_9.2.noarch
SELinux активен               True
Тип регламента                targeted
Режим                         Enforcing
Имя узла                      selinux
Платформа                     Linux selinux 3.10.0-1127.el7.x86_64 #1 SMP Tue
                              Mar 31 23:36:51 UTC 2020 x86_64 x86_64
Счетчик уведомлений           4
Впервые обнаружено            2022-12-26 17:29:49 UTC
В последний раз               2022-12-27 13:05:40 UTC
Локальный ID                  d6d21836-87c9-4cf8-8a26-d1b467138315

Построчный вывод сообщений аудита
type=AVC msg=audit(1672146340.337:1091): avc:  denied  { name_bind } for  pid=25267 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1672146340.337:1091): arch=x86_64 syscall=bind success=no exit=EACCES a0=6 a1=556db7c3e858 a2=10 a3=7fff77a9d400 items=0 ppid=1 pid=25267 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: nginx,httpd_t,unreserved_port_t,tcp_socket,name_bind
```

Вывод: в конфигурации **не найдены** синтаксические ошибки, служба `SELinux` работает в режиме `Enforcing`, которая блокирует запрещенную активность. Инструмент `sealert` предлагает 3 способа решения проблемы.

#### Способ 1: Разрешим в SELinux работу nginx на порту TCP 4881 с помощью переключателей setsebool (Модуль catchall_boolean предлагает (точность 7.83))

##### setsebool

Утилита `setsebool` устанавливает текущее значение для одного переключателя или целого списка переключателей `SELinux`. В качестве значения можно устанавливать `1`, `true` или `on` для включения переключателя. Для выключения можно использовать `0`, `false` или `off`. Если не использовать опцию `-P`, то изменения коснутся только текущих значений переключателей. Значения по умолчанию, которые устанавливаются во время загрузки, при этом не меняются. Если задана опция `-P`, то все текущие значения переключателей сохраняются в файле политики на диске. Таким образом, их значение сохранится и после перезагрузки. ([источник](https://www.opennet.ru/man.shtml?topic=setsebool&category=8&russian=0))

##### audit2why

Утилита `audit2why` обрабатывает сообщения аудита `SELinux`, принятые со стандартного ввода, и сообщает, какой компонент политики вызвал каждый из запретов. Если задана опция `-p`, то используется указанная этой опцией политика, в противном случае используется активная политика. Бывает три возможные причины: 1) отсутствует или отключено разрешительное `TE` правило (`TE allow rule`), 2) нарушение ограничения (`a constraint violation`) или 3) отсутствует разрешительное правило для роли (`role allow rule`). В первом случае разрешительное `TE` правило может присутствовать в политике, но было отключено через булевы переключатели. Смотрите [booleans(8)](https://www.opennet.ru/cgi-bin/opennet/man.cgi?topic=booleans&category=8). Если же разрешительного правила не было, то оно может быть создано при помощи утилиты [audit2allow(1)](README.md#audit2allow). Во втором случае могло произойти нарушение ограничения. Просмотрите `policy/constraints` или `policy/mls` для того, чтобы определить искомое ограничение. Обычно, проблема может быть решена добавлением атрибута типа к домену. В третьем случае была произведена попытка сменить роль, при том что не существует разрешительного правила для участвующей при этом пары ролей. Проблема может быть решена добавлением в политику разрешительного правила для этой пары ролей. ([источник](https://www.opennet.ru/man.shtml?topic=audit2why&category=8&russian=0))

Если отсутствует, установим утилиту `audit2why`, которая находится в пакете `policycoreutils-python`: `yum install policycoreutils-python -y`

Нам известен `PID` процесса `nginx` (`2830`), который вызвал ошибку и это действие было зафиксировано в логе сервиса `audit`, выполним команду: `ausearch -p 2830 | audit2why`:

![Screenshot_selinux1](https://raw.githubusercontent.com/mmmex/selinux/master/screenshots/screenshot_selinux1.png)

```bash
[root@selinux ~]# ausearch -p 2830 | audit2why
type=AVC msg=audit(1672075789.957:814): avc:  denied  { name_bind } for  pid=2830 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly. 
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
[root@selinux ~]# setsebool -P nis_enabled 1
[root@selinux ~]# systemctl start nginx 
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-12-26 21:50:39 UTC; 10s ago
  Process: 21929 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21927 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21926 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21931 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21931 nginx: master process /usr/sbin/nginx
           └─21932 nginx: worker process

Dec 26 21:50:38 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Dec 26 21:50:39 selinux nginx[21927]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Dec 26 21:50:39 selinux nginx[21927]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Dec 26 21:50:39 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Открываем браузер и переходим по адресу: `http://127.0.0.1:4881`:

![Screenshot_selinux2](https://raw.githubusercontent.com/mmmex/selinux/master/screenshots/screenshot_selinux2.png)

Возвращаем обратно, отключаем `nis_enabled`: `setsebool -P nis_enabled off`

#### Способ 2: Разрешим в SELinux работу nginx на порту TCP 4881 с помощью добавления нестандартного порта в тип http_port_t (Модуль bind_ports предлагает (точность 92.2))

##### semanage

`semanage` используется для настройки некоторых элементов политики `SELinux` без необходимости модификации или повторной компиляции исходного текста политики. В число таких настроек входит сопоставление имен пользователей `Linux` пользователям `SELinux` (которые контролируют исходный контекст безопасности, присваиваемый пользователям `Linux` во время их регистрации в системе, и ограничивают доступный набор ролей). Также в число настраиваемых элементов входит сопоставление контекстов безопасности для различных видов объектов, таких как: сетевые порты, интерфейсы, сетевые узлы (хосты), а также контексты файлов. Обратите внимание, что при вызове команды `semanage login` сопоставляются имена пользователей `Linux (logins)` пользователям `SELinux`. А при вызове команды `semanage user` сопоставляются пользователи `SELinux` доступному набору ролей. В большинстве случаев администратором выполняется настройка только первого типа сопоставлений. Второй тип сопоставлений как правило определяется базовой политикой и обычно не требует модификации. ([источник](https://www.opennet.ru/man.shtml?topic=semanage&category=8&russian=0))

Просмотрим имеющиеся типы `http`:

```bash
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавим порт в тип `http_port_t`: `semanage port -a -t http_port_t -p tcp 4881` и выполним запуск сервиса `nginx`.

```bash
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-12-26 22:16:10 UTC; 12s ago
  Process: 22016 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22013 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22012 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22018 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22018 nginx: master process /usr/sbin/nginx
           └─22020 nginx: worker process

Dec 26 22:16:10 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Dec 26 22:16:10 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Dec 26 22:16:10 selinux nginx[22013]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Dec 26 22:16:10 selinux nginx[22013]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Dec 26 22:16:10 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Удаляем нестандартный порт из типа http_port_t: `semanage port -d -t http_port_t -p tcp 4881`

#### Способ 3: Разрешим в SELinux работу nginx на порту TCP 4881 с помощью формирования и установки модуля SELinux (Модуль catchall предлагает (точность 1.41))

##### audit2allow

`audit2allow` - создает разрешающие правила политики `SELinux` из файлов журналов, содержащих сообщения о запрете операций. Утилита сканирует журналы в поиске сообщений, появляющихся когда система не дает разрешения на операцию. Далее утилита генерирует ряд правил, которые, будучи загруженными в политику, могли бы разрешить эти операции. Однако, данная утилита генерирует только разрешающие правила `Type Enforcement (TE)`. Некоторые отказы в использовании разрешений могут потребовать других изменений политики. Например, добавление атрибута в определение типа, для разрешения существующего ограничения `(constraint)`, добавления разрешающего правила для роли или модификации ограничения `(constraint)`. В случае сомнений для диагностики можно попробовать использовать утилиту [audit2why(8)](README.md#audit2why). Следует с осторожностью работать с выводом данной утилиты, убедившись, что разрешаемые операции не представляют угрозы безопасности. Обычно бывает лучше определить новый домен и/или тип или произвести другие структурные изменения. Лучше избирательно разрешить оптимальный набор операций вместо того, чтобы вслепую применить иногда слишком широкие разрешения, рекомендованные этой утилитой. Некоторые запреты на использование разрешений бывают не принципиальны для приложения. В таких случаях вместо использования разрешительного правила `('allow' rule)` лучше просто подавить журналирование этих запретов при помощи правила `dontaudit`. ([источник](https://www.opennet.ru/man.shtml?topic=audit2allow&category=1&russian=0))

Попробуем заново запустить сервис `nginx`:

```bash
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Mon 2022-12-26 22:25:16 UTC; 5s ago
  Process: 22049 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22083 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 22082 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22051 (code=exited, status=0/SUCCESS)

Dec 26 22:25:16 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Dec 26 22:25:16 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Dec 26 22:25:16 selinux nginx[22083]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Dec 26 22:25:16 selinux nginx[22083]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Dec 26 22:25:16 selinux nginx[22083]: nginx: configuration file /etc/nginx/nginx.conf test failed
Dec 26 22:25:16 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Dec 26 22:25:16 selinux systemd[1]: Unit nginx.service entered failed state.
Dec 26 22:25:16 selinux systemd[1]: nginx.service failed.
```

Сборка модуля `SELinux` осуществляется при помощи утилиты `audit2allow`:

```bash
[root@selinux ~]# ausearch -p 22083 | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]# ls -la /root | grep nginx
-rw-r--r--.  1 root root  960 Dec 26 22:28 nginx.pp
-rw-r--r--.  1 root root  257 Dec 26 22:28 nginx.te
```

После выполнения `audit2allow` создаст два файла в текущей папке (root): `nginx.pp` и `nginx.te`

* nginx.pp - готовый к установке скомпилированный модуль;

* nginx.te - исходный код модуля, в который при необходимости можно внести свои правки;

Установка модуля: `semodule -i nginx.pp`

```bash
[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-12-26 22:37:34 UTC; 3s ago
  Process: 22128 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22125 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22124 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22130 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22130 nginx: master process /usr/sbin/nginx
           └─22132 nginx: worker process

Dec 26 22:37:34 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Dec 26 22:37:34 selinux nginx[22125]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Dec 26 22:37:34 selinux nginx[22125]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Dec 26 22:37:34 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Для удаления модуля потребуется команда: `semodule -r nginx`

```bash
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```

---

## 
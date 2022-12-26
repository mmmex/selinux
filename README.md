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

Итак, имеется готовый [Vagrantfile](Vagrantfile). Запускаем ВМ: `vagrant up`.

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

Вывод: в конфигурации **не найдены** синтаксические ошибки, служба `SELinux` работает в режиме `Enforcing`, которая блокирует запрещенную активность.

#### Способ 1: Разрешим в SELinux работу nginx на порту TCP 4881 с помощью переключателей setsebool

Для начала установим дополнительную утилиту `audit2why`, которая находится в пакете `policycoreutils-python`: `yum install policycoreutils-python -y`

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

#### Способ 2: Разрешим в SELinux работу nginx на порту TCP 4881 с помощью добавления нестандартного порта в тип http_port_t

Просмотрим имеющиеся типы:

```bash
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавим порт в тип http_port_t: `semanage port -a -t http_port_t -p tcp 4881`

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

Удалим нестандартный порт из типа http_port_t: `semanage port -d -t http_port_t -p tcp 4881`

#### Способ 3: Разрешим в SELinux работу nginx на порту TCP 4881 с помощью формирования и установки модуля SELinux

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

Соберем модуль `SELinux` при помощи утилиты `audit2allow`:

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

* nginx.pp - готовый скомпилированный модуль, готовый к установке;

* nginx.te - исходный код модуля;

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
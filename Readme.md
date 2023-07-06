Стенд с Vagrant c SELinux

Цель домашнего задания
Диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.

Описание домашнего задания
1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

Поднимаем виртуальную машину с Centos 7 с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен


После запуска ВМ nginx не стартует из-за запрета SELinu работать на нестандартном порту 4881:

```
neva@Uneva:~$ vagrant ssh
[vagrant@selinux ~]$ systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2023-07-04 14:12:16 UTC; 5min ago
  Process: 2582 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2581 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
```

проверим, что в ОС отключен файервол:

```
[root@selinux ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Проверим конфигурацию nginx:

```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Проверим режим работы SELinux:

```
[root@selinux ~]# getenforce
Enforcing
```

Значит, SELinux блокирует работу nginx на нестандартном порту.

Вариант решения 1: Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

```

Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта

```
[root@selinux ~]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1688479936.920:663): avc:  denied  { name_bind } for  pid=2582 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим причину блокировки трафика

```
[root@selinux ~]# grep 1688479936.920:663 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1688479936.920:663): avc:  denied  { name_bind } for  pid=2582 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```

 Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled. 

ключим параметр nis_enabled и перезапустим nginx:

```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2023-07-05 11:47:44 UTC; 8s ago
  Process: 21999 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21997 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21996 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22001 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22001 nginx: master process /usr/sbin/nginx
           └─22003 nginx: worker process

Jul 05 11:47:44 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 05 11:47:44 selinux nginx[21997]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 05 11:47:44 selinux nginx[21997]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jul 05 11:47:44 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Проверить статус параметра nis можно с помощью команды: getsebool -a | grep nis_enabled

```
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```

Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled:

```
[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

Проверяем, Nginx более не запускается.

Вариант решения 2: Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:

Проверим, какие порты определены для http:

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавим порт в тип http_port_t:

```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

Теперь перезапустим службу nginx и проверим её работу:

```
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-07-06 08:42:03 UTC; 7s ago
  Process: 22793 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22791 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22790 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22795 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22795 nginx: master process /usr/sbin/nginx
           └─22797 nginx: worker process

Jul 06 08:42:03 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 06 08:42:03 selinux nginx[22791]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 06 08:42:03 selinux nginx[22791]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jul 06 08:42:03 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Удаляем порт 4881 из имеющегося типа, рестартуем nginx и видим, что он снова не запускается:

```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.


[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2023-07-06 08:54:30 UTC; 1min 22s ago
  Process: 22793 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22818 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 22817 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22795 (code=exited, status=0/SUCCESS)

Jul 06 08:54:30 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jul 06 08:54:30 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 06 08:54:30 selinux nginx[22818]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 06 08:54:30 selinux nginx[22818]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jul 06 08:54:30 selinux nginx[22818]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jul 06 08:54:30 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jul 06 08:54:30 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jul 06 08:54:30 selinux systemd[1]: Unit nginx.service entered failed state.
Jul 06 08:54:30 selinux systemd[1]: nginx.service failed.

```

Вариант решения 3: Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:

Посмотрим логи SELinux, которые относятся к nginx: 

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log
type=SOFTWARE_UPDATE msg=audit(1688479936.198:661): pid=2438 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-filesystem-1:1.20.1-10.el7.noarch" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=SOFTWARE_UPDATE msg=audit(1688479936.541:662): pid=2438 uid=0 auid=1000 ses=2 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 msg='sw="nginx-1:1.20.1-10.el7.x86_64" sw_type=rpm key_enforce=0 gpg_res=1 root_dir="/" comm="yum" exe="/usr/bin/python2.7" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1688479936.920:663): avc:  denied  { name_bind } for  pid=2582 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1688479936.920:663): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=56547f263878 a2=10 a3=7fff233fe5c0 items=0 ppid=1 pid=2582 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1688479936.920:664): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1688557664.204:903): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1688557891.580:908): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1688557891.580:909): avc:  denied  { name_bind } for  pid=22023 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1688557891.580:909): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=5560ee76c878 a2=10 a3=7fff6e605480 items=0 ppid=1 pid=22023 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1688557891.580:910): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1688632923.898:1103): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=SERVICE_STOP msg=audit(1688633670.140:1107): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
type=AVC msg=audit(1688633670.140:1108): avc:  denied  { name_bind } for  pid=22818 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1688633670.140:1108): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=55900bcba878 a2=10 a3=7fff43d24460 items=0 ppid=1 pid=22818 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1688633670.140:1109): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```

Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль. Выполним её:

```
[root@selinux ~]# semodule -i nginx.pp
```

Пробуем снова запустить nginx:

```
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-07-06 10:19:27 UTC; 9s ago
  Process: 22885 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22883 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22882 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22887 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22887 nginx: master process /usr/sbin/nginx
           └─22889 nginx: worker process

Jul 06 10:19:27 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 06 10:19:27 selinux nginx[22883]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 06 10:19:27 selinux nginx[22883]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jul 06 10:19:27 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@selinux ~]#
```

Ура, он работает! При использовании модуля изменения сохранятся после перезагрузки. 

Для удаления модуля воспользуемся командой: semodule -r nginx

```
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```


2. Обеспечение работоспособности приложения при включенном SELinux

Клонируем репозиторий:

```
git clone https://github.com/mbfx/otus-linux-adm.git
```

Перейдём в каталог со стендом и развернём 2 ВМ с помощью vagrant. Проверим, что ВМ поднялись: 

```
neva@Uneva:~/DZ12/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)
```

Подключимся к клиенту и попробуем внести изменения в зону::

```
vagrant ssh client
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```
Видим ошибку, поищем причину в логах SELinux. Для этого воспользуемся утилитой audit2why:

```
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
```

Ошибок нет. Выполним аналогичный запрос на ВМ ns01:

```
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1688649355.774:1962): avc:  denied  { create } for  pid=5295 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```

В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.
Проверим данную проблему в каталоге /etc/named:

```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```

Видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды: sudo semanage fcontext -l | grep named

```
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0
/var/named/chroot/proc(/.*)?                       all files          <<None>>
/var/named/chroot/var/tmp(/.*)?                    all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/usr/lib(/.*)?                    all files          system_u:object_r:lib_t:s0
/var/named/chroot/etc/pki(/.*)?                    all files          system_u:object_r:cert_t:s0
/var/named/chroot/run/named.*                      all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0
/usr/lib/systemd/system/named.*                    regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/run/dbus(/.*)?               all files          system_u:object_r:system_dbusd_var_run_t:s0
/usr/lib/systemd/system/unbound.*                  regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/log/named.*                  regular file       system_u:object_r:named_log_t:s0
/var/named/chroot/var/run/named.*                  all files          system_u:object_r:named_var_run_t:s0
/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0
/usr/lib/systemd/system/named-sdb.*                regular file       system_u:object_r:named_unit_file_t:s0
/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0
/var/named/chroot/etc/named\.rfc1912.zones         regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0
/var/run/ndc                                       socket             system_u:object_r:named_var_run_t:s0
/dev/gpmdata                                       named pipe         system_u:object_r:gpmctl_t:s0
/dev/initctl                                       named pipe         system_u:object_r:initctl_t:s0
/dev/xconsole                                      named pipe         system_u:object_r:xconsole_device_t:s0
/usr/sbin/named                                    regular file       system_u:object_r:named_exec_t:s0
/etc/named\.conf                                   regular file       system_u:object_r:named_conf_t:s0
/usr/sbin/lwresd                                   regular file       system_u:object_r:named_exec_t:s0
/var/run/initctl                                   named pipe         system_u:object_r:initctl_t:s0
/usr/sbin/unbound                                  regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-sdb                                regular file       system_u:object_r:named_exec_t:s0
/var/named/named\.ca                               regular file       system_u:object_r:named_conf_t:s0
/etc/named\.root\.hints                            regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/dev                              directory          system_u:object_r:device_t:s0
/etc/rc\.d/init\.d/named                           regular file       system_u:object_r:named_initrc_exec_t:s0
/usr/sbin/named-pkcs11                             regular file       system_u:object_r:named_exec_t:s0
/etc/rc\.d/init\.d/unbound                         regular file       system_u:object_r:named_initrc_exec_t:s0
/usr/sbin/unbound-anchor                           regular file       system_u:object_r:named_exec_t:s0
/usr/sbin/named-checkconf                          regular file       system_u:object_r:named_checkconf_exec_t:s0
/usr/sbin/unbound-control                          regular file       system_u:object_r:named_exec_t:s0
/var/named/chroot_sdb/dev                          directory          system_u:object_r:device_t:s0
/var/named/chroot/var/log                          directory          system_u:object_r:var_log_t:s0
/var/named/chroot/dev/log                          socket             system_u:object_r:devlog_t:s0
/etc/rc\.d/init\.d/named-sdb                       regular file       system_u:object_r:named_initrc_exec_t:s0
/var/named/chroot/dev/null                         character device   system_u:object_r:null_device_t:s0
/var/named/chroot/dev/zero                         character device   system_u:object_r:zero_device_t:s0
/usr/sbin/unbound-checkconf                        regular file       system_u:object_r:named_exec_t:s0
/var/named/chroot/dev/random                       character device   system_u:object_r:random_device_t:s0
/var/run/systemd/initctl/fifo                      named pipe         system_u:object_r:initctl_t:s0
/var/named/chroot/etc/rndc\.key                    regular file       system_u:object_r:dnssec_t:s0
/usr/share/munin/plugins/named                     regular file       system_u:object_r:services_munin_plugin_exec_t:s0
/var/named/chroot_sdb/dev/null                     character device   system_u:object_r:null_device_t:s0
/var/named/chroot_sdb/dev/zero                     character device   system_u:object_r:zero_device_t:s0
/var/named/chroot/etc/localtime                    regular file       system_u:object_r:locale_t:s0
/var/named/chroot/etc/named\.conf                  regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot_sdb/dev/random                   character device   system_u:object_r:random_device_t:s0
/etc/named\.caching-nameserver\.conf               regular file       system_u:object_r:named_conf_t:s0
/usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0
/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0
/var/named/chroot/lib64 = /usr/lib
/var/named/chroot/usr/lib64 = /usr/lib
```

Изменим тип контекста безопасности для каталога /etc/named и проверим, что получилось:

```
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

Попробуем снова внести изменения с клиента: 

```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```

Видим, что ошибок при изменении записи нет. Проверим:

```
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28700
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 8 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Jul 06 14:01:05 UTC 2023
;; MSG SIZE  rcvd: 96
```

Изменения применились. Попробуем перезагрузить хосты и ещё раз сделать запрос с помощью dig

```
[root@client ~]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56467
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Jul 06 14:03:12 UTC 2023
;; MSG SIZE  rcvd: 96
```

После перезагрузки настройки сохранились. Для того, чтобы вернуть правила обратно, можно ввести команду:

```
[root@ns01 ~]# restorecon -v -R /etc/named
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0


















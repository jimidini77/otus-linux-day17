# otus-linux-day17
SELinux

# **Prerequisite**
- Host OS: Debian 11.3.0
- Guest OS: CentOS 7.8.2003
- VirtualBox: 6.1.36
- Vagrant: 2.2.19
- Ansible: 2.12.4
- Git: 2.36.1
  
# **Содержание ДЗ**

* Запуск `nginx` на нестандартном порту тремя способами
* Обеспечить работоспособность приложения, развернутого из приложенного стенда, при включенном SELinux


# **Выполнение**

## Запуск `nginx` на нестандартном порту тремя способами

Создана машина с установленным `nginx` в конфиге, которого стандартный порт заменён на 4881.

На этапе разворачивание машины получены сообщения об ошибках. SELinux блокирует работу сервиса на нестандартном порту:
```
    selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Tue 2022-08-02 10:37:41 UTC; 23ms ago
    selinux:   Process: 2788 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 2787 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux:
    selinux: Aug 02 10:37:41 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Aug 02 10:37:41 selinux nginx[2788]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Aug 02 10:37:41 selinux nginx[2788]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
    selinux: Aug 02 10:37:41 selinux nginx[2788]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Aug 02 10:37:41 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: Aug 02 10:37:41 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: Aug 02 10:37:41 selinux systemd[1]: Unit nginx.service entered failed state.
    selinux: Aug 02 10:37:41 selinux systemd[1]: nginx.service failed.
The SSH command responded with a non-zero exit status. Vagrant
assumes that this means the command failed. The output for this command
should be in the log above. Please read the output to determine what
went wrong.
```
Проверка состояния файрвола:
```
[root@selinux ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Проверка конфига nginx:
```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Проверка режима работы SELinux:
```
[root@selinux ~]# getenforce
Enforcing
```
### **Способ 1** Разрешение c помощью переключателей setsebool
В логах аудита информация о блокировании порта анализируется утилитой `audit2why`:
```sh
[root@selinux ~]# grep 4881 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1659436661.782:804): avc:  denied  { name_bind } for  pid=2788 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```
Установка рекомендованного параметра и рестарт nginx:
```sh
[root@selinux ~]# setsebool -P nis_enabled 1
[root@selinux ~]# systemctl restart nginx.service
[root@selinux ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-08-02 10:58:15 UTC; 13s ago
  Process: 3060 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3058 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3057 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3062 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3062 nginx: master process /usr/sbin/nginx
           └─3063 nginx: worker process

Aug 02 10:58:15 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 02 10:58:15 selinux nginx[3058]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 02 10:58:15 selinux nginx[3058]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 02 10:58:15 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
nginx слушает порт и отдаёт данные:
```sh
[root@selinux ~]# ss -tnlp | grep 4881
LISTEN     0      128       [::]:4881                  [::]:*                   users:(("nginx",pid=3063,fd=7),("nginx",pid=3062,fd=7))
[root@selinux ~]#
[root@selinux ~]#
[root@selinux ~]# curl -I http://localhost:4881
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Tue, 02 Aug 2022 11:02:16 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
```
Текущее состояние параметра политики SELinux:
```sh
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
Возврат состояния параметра и остановка nginx для следующего способа:
```sh
[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]# systemctl stop nginx.service
```

### **Способ 2** Разрешение c помощью добавления нестандартного порта в имеющийся тип

Поиск имеющегося типа, для http трафика:
```sh
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавление порта в тип `http_port_t`:
```sh
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```
рестарт nginx:
```sh
[root@selinux ~]# systemctl start nginx.service
[root@selinux ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-08-02 12:06:46 UTC; 14s ago
  Process: 21943 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21940 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21939 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21945 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21945 nginx: master process /usr/sbin/nginx
           └─21946 nginx: worker process

Aug 02 12:06:46 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 02 12:06:46 selinux nginx[21940]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 02 12:06:46 selinux nginx[21940]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 02 12:06:46 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
nginx слушает порт и отдаёт данные:
```sh
[root@selinux ~]# ss -tnlp | grep 4881
LISTEN     0      128       [::]:4881                  [::]:*                   users:(("nginx",pid=21946,fd=7),("nginx",pid=21945,fd=7))
[root@selinux ~]#
[root@selinux ~]#
[root@selinux ~]# curl -I http://localhost:4881
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Tue, 02 Aug 2022 12:08:28 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
```

Возврат состояния параметра и остановка nginx для следующего способа:
```sh
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l| grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2022-08-02 18:34:46 UTC; 14s ago
  Process: 21943 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22111 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 22109 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21945 (code=exited, status=0/SUCCESS)

Aug 02 18:34:46 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Aug 02 18:34:46 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 02 18:34:46 selinux nginx[22111]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 02 18:34:46 selinux nginx[22111]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
Aug 02 18:34:46 selinux nginx[22111]: nginx: configuration file /etc/nginx/nginx.conf test failed
Aug 02 18:34:46 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Aug 02 18:34:46 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Aug 02 18:34:46 selinux systemd[1]: Unit nginx.service entered failed state.
Aug 02 18:34:46 selinux systemd[1]: nginx.service failed.
```

### **Способ 3** Разрешение c помощью формирования и установки модуля SELinux

Создание модуля утилитой `audit2allow` на основе логов SELinux:
```sh
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```

Применение сформированного модуля и запуск `nginx`:
```sh
[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl start nginx.service
[root@selinux ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-08-02 18:44:47 UTC; 7s ago
  Process: 22266 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22263 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22262 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22268 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22268 nginx: master process /usr/sbin/nginx
           └─22269 nginx: worker process

Aug 02 18:44:47 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 02 18:44:47 selinux nginx[22263]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 02 18:44:47 selinux nginx[22263]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 02 18:44:47 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

nginx слушает порт и отдаёт данные:
```sh
[root@selinux ~]# ss -tnlp | grep 4881
LISTEN     0      128       [::]:4881                  [::]:*                   users:(("nginx",pid=22269,fd=7),("nginx",pid=22268,fd=7))
[root@selinux ~]#
[root@selinux ~]#
[root@selinux ~]# curl -I http://localhost:4881
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Tue, 02 Aug 2022 18:46:21 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
```

Модуль в списке установленных:
```sh
[root@selinux ~]# semodule -l | grep -C 2 nginx
netutils        1.12.1
networkmanager  1.15.2
nginx   1.0
ninfod  1.0.0
nis     1.12.0
```

## Обеспечить работоспособность приложения, развернутого из приложенного стенда, при включенном SELinux

Склонирован репозиторий, развёрнут стенд `selinux_dns_problems`:
```sh
root@nas:~/work# git clone https://github.com/mbfx/otus-linux-adm.git
root@nas:~/work# cd otus-linux-adm/selinux_dns_problems
root@nas:~/work/otus-linux-adm/selinux_dns_problems# vagrant up
root@nas:~/work/otus-linux-adm/selinux_dns_problems# vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

На машине `client` при попытке внесения изменений в зону - ошибка:
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```

Логи SELinux ошибки не показывают:
```sh
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]#
```

На машине `ns01` логи SELinux содержат ошибки:
```
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1659777573.443:707): avc:  denied  { create } for  pid=683 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
Ошибка в контексте безопасности. Вместо типа `named_t` используется тип `etc_t`.
Проблема в каталоге `/etc/named`:
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

Необходимо выполнить изменение типа контекста безопасности для каталога `/etc/named`: 
```sh
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

Теперь на машине `client` при попытке внесения изменений в зону - успех:
```sh
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.15
> send
> quit
[root@client ~]#
[vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53465
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

;; Query time: 9 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Aug 06 09:52:58 UTC 2022
;; MSG SIZE  rcvd: 96
```

# **Результаты**
Выполнен запуск nginx на нестандартном порту
- установкой переключателя setsebool;
- добавлением нестандартного порта в имеющийся тип;
- формированием и установкой модуля SELinux

Предпочтительным является второй способ, позволяющий более точно и осознано выполнять настройку SELinux

- **GitHub** - https://github.com/jimidini77/otus-linux-day17
# SELinux
## Запустить nginx на нестандартном порту 3-мя разными способами.
### Переключатели setsebool
Изменим порт nginx на 7777, перезапусти nginx.service и проверим, что он не запустился:
````
root@localhost vagrant]# nano /etc/nginx/nginx.conf
[root@localhost vagrant]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@localhost vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2021-10-14 09:19:10 UTC; 37s ago
  Process: 26315 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 26332 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 26331 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 26317 (code=exited, status=0/SUCCESS)
 
Oct 14 09:19:10 localhost.localdomain systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Oct 14 09:19:10 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
Oct 14 09:19:10 localhost.localdomain nginx[26332]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Oct 14 09:19:10 localhost.localdomain nginx[26332]: nginx: [emerg] bind() to 0.0.0.0:7777 failed (13: Permission denied)
Oct 14 09:19:10 localhost.localdomain nginx[26332]: nginx: configuration file /etc/nginx/nginx.conf test failed
Oct 14 09:19:10 localhost.localdomain systemd[1]: nginx.service: control process exited, code=exited status=1
Oct 14 09:19:10 localhost.localdomain systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Oct 14 09:19:10 localhost.localdomain systemd[1]: Unit nginx.service entered failed state.
Oct 14 09:19:10 localhost.localdomain systemd[1]: nginx.service failed.
````
С помощью утилиты sealert сгенерируем отчет об ошибках из лога SELinux:
````
[root@localhost vagrant]# yum install setroubleshoot
[root@localhost vagrant]# sealert -a /var/log/audit/audit.log
````
В выводе видим причину, пот которой nginx не запустился на порту 7777: :
````
SELinux is preventing /usr/sbin/nginx from name_bind access on the tcp_socket port 7777.
````
Также утилита предлагает решить проблему несколькими способами, один из которых - с помощью переключателя setsebool:
````
If you want to allow nis to enabled
Then you must tell SELinux about this by enabling the 'nis_enabled' boolean.

Do
setsebool -P nis_enabled 1
````
Воспользуемся изменением значения переключателя nis_enabled (флаг -P используется для внесения изменений на постоянной основе):

````
[root@localhost vagrant]# setsebool -P nis_enabled 1
````
Nginx запустился с нестандартным портом:
````
[root@localhost vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2021-10-14 09:35:50 UTC; 6s ago
  Process: 26789 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 26787 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 26786 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 26791 (nginx)
   CGroup: /system.slice/nginx.service
           ├─26791 nginx: master process /usr/sbin/nginx
           └─26792 nginx: worker process

Oct 14 09:35:50 localhost.localdomain systemd[1]: Starting The nginx HTTP and reverse proxy server...
Oct 14 09:35:50 localhost.localdomain nginx[26787]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Oct 14 09:35:50 localhost.localdomain nginx[26787]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Oct 14 09:35:50 localhost.localdomain systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@localhost vagrant]# curl http://localhost:7777
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">
````
Возвращаем всё как было и Nginx уже не стартует
````
[root@localhost vagrant]# setsebool -P nis_enabled 0
[root@localhost vagrant]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
````
### Добавление нестандартного порта в имеющийся тип.
Выведем список разрешенных SELinux'ом портов для типа http_port_t:
````
[root@localhost vagrant]# semanage port -l | grep -w http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
````
Видим, что нашего порта 7777 в этом списке нет, добавим его:
````
[root@localhost vagrant]# semanage port -a -t http_port_t -p tcp 7777
````
Проверим что порт добавился:
````
[root@localhost vagrant]# semanage port -l | grep -w http_port_t
http_port_t                    tcp      7777, 80, 81, 443, 488, 8008, 8009, 8443, 9000
````
И снова Nginx отвечает нам на пору 7777:
````
[root@localhost vagrant]# systemctl restart nginx
[root@localhost vagrant]# curl http://localhost:7777
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">
  ````
Удалим порт из разрешенных и перезапустим сервис:
````
[root@localhost vagrant]# semanage port -d -t http_port_t -p tcp 7777
[root@localhost vagrant]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
````
### Формирование и установка модуля SELinux.
Для формирования и установки модуля SELinux воспользуемся утилитой audit2allow, перенаправив на её stdin лог SELinux:
````
[root@localhost vagrant]# audit2allow -M my_nginx_service < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my_nginx_service.pp
````
В результате будет создан модуль my_nginx_service.pp, который нужно установить следующим образом:
````
[root@localhost vagrant]# semodule -i my_nginx_service.pp
````
Перезапускаем Nginx и снова успех:
````
[root@localhost vagrant]# systemctl restart nginx
[root@localhost vagrant]# curl http://localhost:7777
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">
  ````
## Обеспечить работоспособность приложения при включенном selinux.
С клиента выполним попытку обновления зоны ddns.lab:
````
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
````
Проверим ошибки в логе SELinux:
````
[root@ns01 vagrant]# cat /var/log/audit/audit.log | grep denied
type=AVC msg=audit(1634209632.336:2469): avc:  denied  { create } for  pid=28895 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
````
Посмотрим вывод утилиты sealert:
````
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/named from create access on the file named.ddns.lab.view1.jnl.

*****  Plugin catchall_labels (83.8 confidence) suggests   *******************

If you want to allow named to have create access on the named.ddns.lab.view1.jnl file
Then you need to change the label on named.ddns.lab.view1.jnl
Do
# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'
where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.
Then execute:
restorecon -v 'named.ddns.lab.view1.jnl'


*****  Plugin catchall (17.1 confidence) suggests   **************************

If you believe that named should be allowed create access on the named.ddns.lab.view1.jnl file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000
# semodule -i my-iscworker0000.pp


Additional Information:
Source Context                system_u:system_r:named_t:s0
Target Context                system_u:object_r:etc_t:s0
Target Objects                named.ddns.lab.view1.jnl [ file ]
Source                        isc-worker0000
Source Path                   /usr/sbin/named
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages           bind-9.11.4-26.P2.el7_9.7.x86_64
Target RPM Packages
Policy RPM                    selinux-policy-3.13.1-266.el7.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     ns01
Platform                      Linux ns01 3.10.0-1127.el7.x86_64 #1 SMP Tue Mar
                              31 23:36:51 UTC 2020 x86_64 x86_64
Alert Count                   1
First Seen                    2021-10-14 11:07:12 UTC
Last Seen                     2021-10-14 11:07:12 UTC
Local ID                      fb5066e9-3676-4d47-be24-50c45783a255

Raw Audit Messages
type=AVC msg=audit(1634209632.336:2469): avc:  denied  { create } for  pid=28895 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0


type=SYSCALL msg=audit(1634209632.336:2469): arch=x86_64 syscall=open success=no exit=EACCES a0=7f63394f3050 a1=241 a2=1b6 a3=24 items=0 ppid=1 pid=28895 auid=4294967295 uid=25 gid=25 euid=25 suid=25 fsuid=25 egid=25 sgid=25 fsgid=25 tty=(none) ses=4294967295 comm=isc-worker0000 exe=/usr/sbin/named subj=system_u:system_r:named_t:s0 key=(null)

Hash: isc-worker0000,named_t,etc_t,file,create
````
## Следовательно:
Видим, что SELinux запрещает утилите /usr/sbin/named доступ к созданию файла named.ddns.lab.view1.jnl, а также предлагает два варианта решения проблемы: 1й - с помошью утилиты audit2allow создать ращрешающий модуль; 2й - с помощью утилиты semanage изменить контекст для файла named.ddns.lab.view1.jnl.

Создание модулей утилитой audit2allow не рекомендуется, так как можно предоставить какому-либо приложению слишком широкие полномочия, в которых оно по сути не нуждается, поэтому воспользуемся 2м способом.

Из файла /etc/named.conf узнаем, распололжение файла зоны ddns.lab: /etc/named/dynamic/named.ddns.lab.view1. Смотрим тип файла в его контексте безопасности:
````
[root@ns01 ~]# ll -Z /etc/named/dynamic/named.ddns.lab.view1
-rw-rw----. named named system_u:object_r:etc_t:s0       /etc/named/dynamic/named.ddns.lab.view1
````
Видим, что тип - etc_t, а из страницы руководства RedHat следует, что по умолчанию для динамических зон используется директория /var/named/dynamic/, файлы в которой наследуют тип named_cache_t. Изменим тип в контексте для директории /etc/named/dynamic/:
````
[root@ns01 ~]# semanage fcontext -a -t named_cache_t '/etc/named/dynamic(/.*)?'
[root@ns01 ~]# restorecon -R -v /etc/named/dynamic/
restorecon reset /etc/named/dynamic context unconfined_u:object_r:etc_t:s0->unconfined_u:object_r:named_cache_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:etc_t:s0->system_u:object_r:named_cache_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:etc_t:s0->system_u:object_r:named_cache_t:s0
````
Повторим попытку изменения зоны ddns.lab:
````
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> 
> quit
````
````
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.7 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53758
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

;; Query time: 4 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Oct 14 11:23:05 UTC 2021
;; MSG SIZE  rcvd: 96
````

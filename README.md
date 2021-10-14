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
